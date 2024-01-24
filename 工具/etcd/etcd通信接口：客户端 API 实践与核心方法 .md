
我们在前面一课时介绍了 etcd 的整体架构。学习客户端与 etcd 服务端的通信以及 etcd 集群节点的内部通信接口对于我们更好地使用和掌握 etcd 组件很有帮助，也是所必需了解的内容。本课时我们将会介绍 etcd 的 gRPC 通信接口以及客户端的实践。

### etcd clientv3 客户端

etcd 客户端 clientv3 接入的示例将会以 Go 客户端为主，读者需要准备好基本的开发环境。

首先是 etcd clientv3 的初始化，我们根据指定的 etcd 节点，建立客户端与 etcd 集群的连接。

```go
cli,err := clientv3.New(clientv3.Config{ 
	Endpoints:[]string{"localhost:2379"}, 
	DialTimeout: 5 * time.Second, 
})
```

如上的代码实例化了一个 client，这里需要传入的两个参数：

-   Endpoints：etcd 的多个节点服务地址，因为我是单点本机测试，所以只传 1 个。
-   DialTimeout：创建 client 的首次连接超时，这里传了 5 秒，如果 5 秒都没有连接成功就会返回 err；值得注意的是，一旦 client 创建成功，我们就不用再关心后续底层连接的状态了，client 内部会重连。

### etcd 客户端初始化

解决完包依赖之后，我们初始化 etcd 客户端。客户端初始化代码如下所示：

```go
// client_init_test.go
package client

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	"testing"
	"time"
)
// 测试客户端连接
func TestEtcdClientInit(t *testing.T) {

	var (
		config clientv3.Config
		client *clientv3.Client
		err    error
	)
	// 客户端配置
	config = clientv3.Config{
		// 节点配置
		Endpoints:   []string{"localhost:2379"},
		DialTimeout: 5 * time.Second,
	}
	// 建立连接
	if client, err = clientv3.New(config); err != nil {
		fmt.Println(err)
	} else {
		// 输出集群信息
		fmt.Println(client.Cluster.MemberList(context.TODO()))
	}
	client.Close()
}

```

如上的代码，预期的执行结果如下：

```css
=== RUN   TestEtcdClientInit
&{cluster_id:14841639068965178418 member_id:10276657743932975437 raft_term:3  [ID:10276657743932975437 name:"default" peerURLs:"http://localhost:2380" clientURLs:"http://0.0.0.0:2379" ] {} [] 0} <nil>
--- PASS: TestEtcdClientInit (0.08s)
PASS
```

可以看到 clientv3 与 etcd Server 的节点 localhost:2379 成功建立了连接，并且输出了集群的信息，下面我们就可以对 etcd 进行操作了。

### client 定义

接着我们来看一下 client 的定义：

```go
type Client struct {
    Cluster
    KV
    Lease
    Watcher
    Auth
    Maintenance

    // Username is a user name for authentication.
    Username string
    // Password is a password for authentication.
    Password string
}

```

注意，这里显示的都是可导出的模块结构字段，代表了客户端能够使用的几大核心模块，其具体功能介绍如下：

-   Cluster：向集群里增加 etcd 服务端节点之类，属于管理员操作。
-   KV：我们主要使用的功能，即操作 K-V。
-   Lease：租约相关操作，比如申请一个 TTL=10 秒的租约。
-   Watcher：观察订阅，从而监听最新的数据变化。
-   Auth：管理 etcd 的用户和权限，属于管理员操作。
-   Maintenance：维护 etcd，比如主动迁移 etcd 的 leader 节点，属于管理员操作。

#### proto3

etcd v3 的通信基于 gRPC，proto 文件是定义服务端和客户端通讯接口的标准。包括：

-   客户端该传什么样的参数
-   服务端该返回什么参数
-   客户端该怎么调用
-   是阻塞还是非阻塞
-   是同步还是异步。

gRPC 推荐使用 proto3 消息格式，在进行核心 API 的学习之前，我们需要对 proto3 的基本语法有初步的了解。proto3 是原有 Protocol Buffer 2(被称为 proto2)的升级版本，删除了一部分特性，优化了对移动设备的支持。

#### gRPC 服务

发送到 etcd 服务器的每个 API 请求都是一个 gRPC 远程过程调用。etcd3 中的 RPC 接口定义根据功能分类到服务中。

处理 etcd 键值的重要服务包括：

-   KV 服务，创建、更新、获取和删除键值对。
-   监视，监视键的更改。
-   租约，消耗客户端保持活动消息的基元。
-   锁，etcd 提供分布式共享锁的支持。
-   选举，暴露客户端选举机制。
![[5.jpg]]
#### 请求和响应

etcd3 中的所有 RPC 都遵循相同的格式。每个 RPC 都有一个函数名，该函数将 NameRequest 作为参数并返回 NameResponse 作为响应。例如，这是 Range RPC 描述：

```scss
service KV {
  Range(RangeRequest) returns (RangeResponse)
  ...
}
```

#### 响应头

etcd API 的所有响应都有一个附加的响应标头，其中包括响应的群集元数据：

```proto
message ResponseHeader {
  uint64 cluster_id = 1;
  uint64 member_id = 2;
  int64 revision = 3;
  uint64 raft_term = 4;
}
```

-   Cluster\_ID - 产生响应的集群的 ID。
-   Member\_ID - 产生响应的成员的 ID。
-   Revision - 产生响应时键值存储的修订版本号。
-   Raft\_Term - 产生响应时，成员的 Raft 称谓。

应用服务可以通过 Cluster\_ID 和 Member\_ID 字段来确保，当前与之通信的正是预期的那个集群或者成员。

应用服务可以使用修订号字段来知悉当前键值存储库最新的修订号。当应用程序指定历史修订版以进行时程查询并希望在请求时知道最新修订版时，此功能特别有用。

应用服务可以使用 Raft\_Term 来检测集群何时完成一个新的 leader 选举。

下面开始介绍 etcd 中这几个重要的服务和接口。

### KV 存储

kv 对象的实例获取通过如下的方式：

```css
kv := clientev3.NewKV(client)
```

我们来看一下 kv 接口的具体定义：

```scss
type KV interface { Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error) // 检索 keys. Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error) // 删除 key，可以使用 WithRange(end), [key, end) 的方式 Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error) // 压缩给定版本之前的 KV 历史 Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error) // 指定某种没有事务的操作 Do(ctx context.Context, op Op) (OpResponse, error) // Txn 创建一个事务 Txn(ctx context.Context) Txn }
```

从 KV 对象的定义我们可知，它就是一个接口对象，包含几个主要的 kv 操作方法：

#### kv 存储 put

put 的定义如下：

```go
type KV interface {

    Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)

    // 检索 keys.
    Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)

    // 删除 key，可以使用 WithRange(end), [key, end) 的方式
    Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)

    // 压缩给定版本之前的 KV 历史
    Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)

    // 指定某种没有事务的操作
    Do(ctx context.Context, op Op) (OpResponse, error)

    // Txn 创建一个事务
    Txn(ctx context.Context) Txn
}
```

其中的参数：

-   ctx: Context 包对象，是用来跟踪上下文的，比如超时控制
-   key: 存储对象的 key
-   val: 存储对象的 value
-   opts: 　可变参数，额外选项

Put 将一个键值对放入 etcd 中。请注意，键值可以是纯字节数组，字符串是该字节数组的不可变表示形式。要获取字节字符串，请执行 `string([] byte {0x10，0x20})` 。

put 的使用方法如下所示：

```go
putResp, err := kv.Put(context.TODO(),"aa", "hello-world!")
```

#### kv 查询 get

现在可以对存储的数据进行取值了。默认情况下，Get 将返回 “ key” 对应的值。

```go
Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)
```

OpOption 为可选的函数传参，传参为 `WithRange(end)` 时，Get 将返回 \[key，end）范围内的键；传参为 `WithFromKey()` 时，Get 返回大于或等于 key 的键；当通过 rev> 0 传递 `WithRev(rev)` 时，Get 查询给定修订版本的键；如果压缩了所查找的修订版本，则返回请求失败，并显示 ErrCompacted。 传递 `WithLimit(limit)` 时，返回的 key 数量受 limit 限制；传参为 `WithSort` 时，将对键进行排序。对应的使用方法如下：

```go
getResp, err := kv.Get(context.TODO(), "aa")
```

从以上数据的存储和取值，我们知道 put 返回 PutResponse、get 返回 GetResponse，注意：不同的 KV 操作对应不同的 response 结构，定义如下：

```go
type (
    CompactResponse pb.CompactionResponse
    PutResponse     pb.PutResponse
    GetResponse     pb.RangeResponse
    DeleteResponse  pb.DeleteRangeResponse
    TxnResponse     pb.TxnResponse
)
```

我们分别来看一看 PutResponse 和 GetResponse 映射的 RangeResponse 结构的定义：

```go
type PutResponse struct {
    Header *ResponseHeader `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
    // if prev_kv is set in the request, the previous key-value pair will be returned.
    PrevKv *mvccpb.KeyValue `protobuf:"bytes,2,opt,name=prev_kv,json=prevKv" json:"prev_kv,omitempty"`
}
//Header 里保存的主要是本次更新的 revision 信息

type RangeResponse struct {
    Header *ResponseHeader `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
    // kvs is the list of key-value pairs matched by the range request.
    // kvs is empty when count is requested.
    Kvs []*mvccpb.KeyValue `protobuf:"bytes,2,rep,name=kvs" json:"kvs,omitempty"`
    // more indicates if there are more keys to return in the requested range.
    More bool `protobuf:"varint,3,opt,name=more,proto3" json:"more,omitempty"`
    // count is set to the number of keys within the range when requested.
    Count int64 `protobuf:"varint,4,opt,name=count,proto3" json:"count,omitempty"`
}
```

Kvs 字段，保存了本次 Get 查询到的所有 kv 对，我们继续看一下 mvccpb.KeyValue 对象的定义：

```go
type KeyValue struct {

    Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
    // create_revision 是当前 key 的最后创建版本
    CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
    // mod_revision 是指当前 key 的最新修订版本
    ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
    // key 的版本，每次更新都会增加版本号
    Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`

    Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`

    // 绑定了 key 的租期 Id，当 lease 为 0 ，则表明没有绑定 key；租期过期，则会删除 key
    Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
}
```

至于 RangeResponse.More 和 Count，当我们使用 withLimit() 选项进行 Get 时会发挥作用，相当于分页查询。

接下来，我们通过一个特别的 Get 选项，获取 aa 目录下的所有子目录：

```go
rangeResp, err := kv.Get(context.TODO(), "/aa", clientv3.WithPrefix())
```

`WithPrefix()` 用于查找以 `/aa` 为前缀的所有 key，因此可以模拟出查找子目录的效果。

我们知道 etcd 是一个有序的 kv 存储，因此 `/aa` 为前缀的 key 总是顺序排列在一起。

withPrefix 实际上会转化为范围查询，它根据前缀 `/aa` 生成了一个 key range，\[“/aa/”, “/aa0”)，这是因为比 `/` 大的字符是 `0`，所以以 `/aa0` 作为范围的末尾，就可以扫描到所有的 `/aa/` 打头的 key 了。

#### KV 操作实践

键值对的操作是 etcd 中最基本、最常用的功能，主要包括读、写、删除三种基本的操作。在 etcd 中定义了 kv 接口，用来对外提供这些操作，下面我们进行具体的测试：

```go
package client

import (
	"context"
	"fmt"
	"github.com/google/uuid"
	"go.etcd.io/etcd/clientv3"
	"testing"
	"time"
)

func TestKV(t *testing.T) {
	rootContext := context.Background()
	// 客户端初始化
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379"},
		DialTimeout: 2 * time.Second,
	})
	// etcd clientv3 >= v3.2.10, grpc/grpc-go >= v1.7.3
	if cli == nil || err == context.DeadlineExceeded {
		// handle errors
		fmt.Println(err)
		panic("invalid connection!")
	}
	// 客户端断开连接
	defer cli.Close()
	// 初始化 kv
	kvc := clientv3.NewKV(cli)
	//获取值
	ctx, cancelFunc := context.WithTimeout(rootContext, time.Duration(2)*time.Second)
	response, err := kvc.Get(ctx, "cc")
	cancelFunc()
	if err != nil {
		fmt.Println(err)
	}
	kvs := response.Kvs
	// 输出获取的 key
	if len(kvs) > 0 {
		fmt.Printf("last value is :%s\r\n", string(kvs[0].Value))
	} else {
		fmt.Printf("empty key for %s\n", "cc")
	}
	//设置值
	uuid := uuid.New().String()
	fmt.Printf("new value is :%s\r\n", uuid)
	ctx2, cancelFunc2 := context.WithTimeout(rootContext, time.Duration(2)*time.Second)
	_, err = kvc.Put(ctx2, "cc", uuid)
	// 设置成功之后，将该 key 对应的键值删除
	if delRes, err := kvc.Delete(ctx2, "cc"); err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("delete %s for %t\n", "cc", delRes.Deleted > 0)
	}
	cancelFunc2()
	if err != nil {
		fmt.Println(err)
	}
}
```

如上的测试用例，主要是针对 kv 的操作，依次获取 key，即 Get()，对应 etcd 底层实现的 range 接口；其次是写入键值对，即 put 操作；最后删除刚刚写入的键值对。预期的执行结果如下所示：

```css
=== RUN   Test
empty key for cc
new value is: 41e1362a-28a7-4ac9-abf5-fe1474d93f84
delete cc for true
--- PASS: Test (0.11s)
PASS
```

可以看到，刚开始 etcd 并没有存储键 `cc` 的值，随后写入新的键值对并测试将其删除。

### 其他通信接口

其他常用的接口还有 Transaction、Compact、watch、Lease、Lock 等。我们依次看看这些接口的定义。

#### 事务 Transaction

Txn 方法在单个事务中处理多个请求。txn 请求增加键值存储的修订版本并为每个完成的请求生成带有相同修订版本的事件。etcd 不容许在一个 txn 中多次修改同一个 key。 Txn 接口定义如下：

```scss
rpc Txn(TxnRequest) returns (TxnResponse) {}
```

#### Compact 方法

Compact 方法压缩 etcd 键值对存储中的事件历史。键值对存储应该定期压缩，否则事件历史会无限制的持续增长。

```scss
rpc Compact(CompactionRequest) returns (CompactionResponse) {}
```

请求的消息体是 CompactionRequest， CompactionRequest 压缩键值对存储到给定修订版本。所有修订版本比压缩修订版本小的键都将被删除

#### watch

Watch API 提供了一个基于事件的接口，用于异步监视键的更改。etcd3 监视程序通过从给定的修订版本（当前版本或历史版本）持续监视 key 更改，并将 key 更新流回客户端。

在 rpc.proto 中 Watch service 定义如下：

![[6.jpg]]

```go
service Watch {
  rpc Watch(stream WatchRequest) returns (stream WatchResponse) {}
}
```

Watch 观察将要发生或者已经发生的事件。输入和输出都是流;输入流用于创建和取消观察，而输出流发送事件。一个观察 RPC 可以在一次性在多个 key 范围上观察，并为多个观察流化事件。整个事件历史可以从最后压缩修订版本开始观察。 WatchService 只有一个 Watch 方法。

#### Lease service

Lease service 提供租约的支持。Lease 是一种检测客户端存活状况的机制。群集授予具有生存时间的租约。如果 etcd 群集在给定的 TTL 时间内未收到 keepAlive，则租约到期。

为了将租约绑定到键值存储中，每个 key 最多可以附加一个租约。当租约到期或被撤销时，该租约所附的所有 key 都将被删除。每个过期的密钥都会在事件历史记录中生成一个删除事件。

在 rpc.proto 中 Lease service 定义的接口如下：

```go
service Lease {

  rpc LeaseGrant(LeaseGrantRequest) returns (LeaseGrantResponse) {}

  rpc LeaseRevoke(LeaseRevokeRequest) returns (LeaseRevokeResponse) {}

  rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}

  rpc LeaseTimeToLive(LeaseTimeToLiveRequest) returns (LeaseTimeToLiveResponse) {}
}
```

-   LeaseGrant 创建一个租约
-   LeaseRevoke 撤销一个租约
-   LeaseKeepAlive 用于维持租约
-   LeaseTimeToLive 获取租约信息

#### Lock service

Lock service 提供分布式共享锁的支持。Lock service 以 gRPC 接口的方式暴露客户端锁机制。在 v3lock.proto 中 Lock service 定义如下：

```go
service Lock {

  rpc Lock(LockRequest) returns (LockResponse) {}

  rpc Unlock(UnlockRequest) returns (UnlockResponse) {}
}
```

-   Lock 方法，在给定命令锁上获得分布式共享锁。
-   Unlock 使用 Lock 返回的 key 并释放对锁的持有。

### 小结

本文主要介绍了 etcd 的 gRPC 通信接口以及 clientv3 客户端的实践，主要包括键值对操作（增删改查）、watch、Lease、锁和 Compact 等接口。通过本课时的学习，了解 etcd 客户端的使用以及常用功能的接口定义，对于我们在日常工作中能够得心应手的使用 etcd 实现相应的功能能够很有帮助。

当然，本课时限于篇幅，只是介绍了常用的几个通信接口，如果你对其他的接口还有疑问，欢迎在留言区提出。下一课时我们将介绍 etcd 的存储，如何实现键值对的读写操作。