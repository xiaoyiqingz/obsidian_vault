
[腾讯云开发者](https://juejin.cn/user/3456520257538830/posts)

2017-04-01 10:11 1562

[腾讯云技术社区-掘金主页](https://juejin.cn/user/3456520257538830 "https://juejin.cn/user/3456520257538830")持续为大家呈现云计算技术文章，欢迎大家关注！

* * *

> 作者介绍:熊训德（英文名：Sundy），16年毕业于四川大学大学并加入腾讯。目前在腾讯云从事hadoop生态相关的云存储和计算等后台开发，喜欢并专注于研究大数据、虚拟化和人工智能等相关技术。

本文档说明go语言自带的测试框架未提供或者未方便地提供的测试方案，主要是用于解决写单元测试中比较头痛的依赖问题。也就是伪造模式，经典的伪造模式有桩对象(stub),模拟对象(mock)和伪对象(fake)。比较幸运的是，社区有丰富的第三方测试框架支持支持。下面就对笔者亲身试用并实践到项目中的几个框架做介绍：

#### 1.gomock

[godoc.org/github.com/…](https://link.juejin.cn/?target=https%3A%2F%2Fgodoc.org%2Fgithub.com%2Fgolang%2Fmock%2Fgomock "https://godoc.org/github.com/golang/mock/gomock")

gomock模拟对象的方式是让用户声明一个接口，然后使用gomock提供的mockgen工具生成mock对象代码。要模拟(mock)被测试代码的依赖对象时候，即可使用mock出来的对象来模拟和记录依赖对象的各种行为：比如最常用的返回值，调用次数等等。文字叙述有点抽象，直接上代码：

![[图片/go 单元测试进阶篇/1.png]]

dick.go中DickFunc依赖外部对象OutterObj，本示例就是说明如何使用gomock框架控制所依赖的对象。

```go
func DickFunc( outterObj MockInterface,para int)(result int){
    fmt.Println("This init DickFunc")
    fmt.Println("call outter.func:")

    return outterObj.OutterFunc(para)
}
```

mockgen工具命令是：

```bash
mockgen -source {source_file}.go -destination {dest_file}.go
```

比如，本示例即是：

```bash
mockgen -source src_mock.go -destination dst_mock.go
````

执行完后，可在同目录下找到生成的dst\_mock.go文件，可以看到mockgen工具也实现了接口：

![[图片/go 单元测试进阶篇/2.png]]

接下来就可以使用mockgen工具生成的NewMockInterFace来生产mock对象，使用这个mock对象。OutterFunc()这个函数，gomock在控制mock类时支持链式编程的方式，其原理和其他链式编程类似一直维持了一个Call对象，把需要控制的方法名，入参，出参，调用次数以及前置和后置动作等，最后使用反射来调用方法，所以这个Call对象是mock对象的代理。jmockit的早期版本也是jdk自带的java.reflect.Proxy动态代理实现的(最近的版本是动态Instrumentation配合代理模式)。  

![[图片/go 单元测试进阶篇/3.png]]

在本示例中只简单的更改了返回值，抛砖引玉：

```go
func TestDickFunc(t *testing.T ){
   mockCtrl := gomock.NewController(t)
//defer mockCtrl.Finish()

   mockObj := dick.NewMockMockInterface(mockCtrl)
   mockObj.EXPECT().OutterFunc(3).Return(10)

   result :=dick.DickFunc(mockObj,3)
   t.Log("resutl:",result)

}
```

使用go test命令执行这个单测  

![[图片/go 单元测试进阶篇/4.png]]

从结果看：本来应该输出3，最后输出就是10，和其他语言mock框架相似，生产出来的Mock对象不用自己去重定义这么麻烦。

更多示例可以查看官网一个囊括gomock几乎所有功能的例子：

[godoc.org/github.com/…](https://link.juejin.cn/?target=https%3A%2F%2Fgodoc.org%2Fgithub.com%2Fgolang%2Fmock%2Fsample "https://godoc.org/github.com/golang/mock/sample")

#### 2.httpexcept

由于go在网络架构上的优秀封装，使得go在很多网络场景被广泛使用，而http协议是其中重要部分，在面对http请求的时候，可以对http的client进行测试，算是mock的特殊应用场景。

看一个简单的示例就轻松的看懂了：

```go
func TestHttp(t *testing.T) {

    handler := FruitServer()

    server := httptest.NewServer(handler)
    defer server.Close()

    e := httpexpect.New(t, server.URL)

    e.GET("/fruits").
        Expect().
        Status(http.StatusOK).JSON().Array().Empty()
}
```

其中还支持对不同方法(包括Header,Post等)的构造以及返回值Json的自定义，更多细节查看其[官网](https://link.juejin.cn/?target=http%3A%2F%2Fwww.ctolib.com%2Farticle%2FgoGitHub%2Fhttpexpect.html "http://www.ctolib.com/article/goGitHub/httpexpect.html")

#### 3.testify

还有一个testify使用起来可以说兼容了《一》中的gocheck和gomock，但是其mock使用稍微有点烦杂，使用继承tetify.Mock(匿名组合)重新实现需要Mock的接口，在这个接口里使用者自己使用Called(反射实现)被Mock的接口。

《单元测试的艺术》中认为stub和mock最大的区别就依赖对象是否和被测对象有交互，而从结果看就是桩对象不会使测试失败，它只是为被测对象提供依赖的对象，并不改变测试结果，而mock则会根据不同的交互测试要求，很可能会更改测试的结果。说了这么多理论，但其实这两种方法都不是割裂的，所以gomock框架除了像其名字一样可以模拟对象以外，还提供了桩对象的功能(stub)。以其实现来说，更像是一个桩对象的注入。但是因为兼容了多个有用的功能，所以其在社区最为火爆。

具体用法可参考其[github主页](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fstretchr%2Ftestify "https://github.com/stretchr/testify")

#### 4.go-sqlmock

还有一种比较常见的场景就是和数据库的交互场景，go-sqlmock是sql模拟（Mock）驱动器，主要用于测试数据库的交互，go-sqlmock提供了完整的事务的执行测试框架，最新的版本(16.11.02)还支持prepare参数化提交和执行的Mock方案。

比如有这样的被测函数：

```go
func recordStats(db *sql.DB, userID, productID int64) (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }

    defer func() {
        switch err {
        case nil:
            err = tx.Commit()
        default:
            tx.Rollback()
        }
    }()

    if _, err = tx.Exec("UPDATE products SET views = views + 1"); err != nil {
        return
    }
    if _, err = tx.Exec("INSERT INTO product_viewers (user_id, product_id) VALUES (?, ?)", userID, productID); err != nil {
        return
    }
    return
}

func main() {

    db, err := sql.Open("mysql", "root@/root")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    if err = recordStats(db, 1 , 5 ); err != nil {
        panic(err)
    }
}
```

单测时：

```go
func TestShouldUpdateStats(t *testing.T) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("mock error: '%s' ", err)
    }
    defer db.Close()

    mock.ExpectBegin()
    mock.ExpectExec("UPDATE products").WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectExec("INSERT INTO product_viewers")
          .WithArgs(2, 3)
          .WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectCommit()

    if err = recordStats(db, 2, 3); err != nil {
        t.Errorf("exe error: %s", err)
    }

    if err := mock.ExpectationsWereMet(); err != nil {
        t.Errorf("not implements: %s", err)
    }
}

//测试回滚
func TestShouldRollbackStatUpdatesOnFailure(t *testing.T) {
    db, mock, err := sqlmock.New()
    if err != nil {
        t.Fatalf("mock error: '%s'", err)
    }
    defer db.Close()

    mock.ExpectBegin()
    mock.ExpectExec("UPDATE products").WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectExec("INSERT INTO product_viewers")
           .WithArgs(2, 3)
           .WillReturnError(fmt.Errorf("some error"))
    mock.ExpectRollback()

    // 执行被测方法,有错
    if err = recordStats(db, 2, 3); err == nil {
        t.Errorf("not error")
    }

    // 执行被测方法，mock对象
    if err := mock.ExpectationsWereMet(); err != nil {
        t.Errorf("not implements: %s", err)
    }
}
```


更多例子和详情，请查看官网:

[github.com/DATA-DOG/go…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FDATA-DOG%2Fgo-sqlmock "https://github.com/DATA-DOG/go-sqlmock")

介绍了这么多框架，最后需要说明的也可能最重要的是写代码时就应该考虑代码是可被测试的。要使得单元测试容易写，或者说代码容易被测，其实很重要的一个部分就是被测代码本身是容易被测的，也就是说在设计和编写代码的时候就应该先想到相好如何单元测试，甚至有人提出可以先写单元测试，再写具体被测代码。因为一个接口(或者称为单元)在被设计好后，它实现就确定了，实际效果也确定了。这种方式被称作测试驱动开发(Test-Driven Development, TDD)。而对于已经写好的代码，很大程度上不好测试，有一种方式是测试性重构，就是为了更好的测试而进行重构。这些一定程度上来说并了解这些框架更重要，有意向可以，可以查阅有关两本书《单元测试的艺术(第2版)》《xUnit测试模式》

> 参考：  
> [codethoughts.info/go/2015/04/…](https://link.juejin.cn/?target=http%3A%2F%2Fcodethoughts.info%2Fgo%2F2015%2F04%2F05%2Fhow-to-test-go-code%2F "http://codethoughts.info/go/2015/04/05/how-to-test-go-code/")
> 
> [nathany.com/go-testing-…](https://link.juejin.cn/?target=https%3A%2F%2Fnathany.com%2Fgo-testing-toolbox%2F "https://nathany.com/go-testing-toolbox/")
> 
> [shinley.com/index.html](https://link.juejin.cn/?target=http%3A%2F%2Fshinley.com%2Findex.html "http://shinley.com/index.html")
> 
> 《单元测试的艺术》
> 
> 《xUnit测试模式》

相关阅读：

[go单元测试基本篇](https://link.juejin.cn/?target=https%3A%2F%2Fwww.qcloud.com%2Fcommunity%2Farticle%2F56073001484044261%3FfromSource%3Dgwzcw.59732.59732.59732 "https://www.qcloud.com/community/article/56073001484044261?fromSource=gwzcw.59732.59732.59732")  
[【腾讯TMQ】敏捷测试-快速俘虏产品&开发](https://link.juejin.cn/?target=https%3A%2F%2Fwww.qcloud.com%2Fcommunity%2Farticle%2F898139001487750546%3FfromSource%3Dgwzcw.59733.59733.59733 "https://www.qcloud.com/community/article/898139001487750546?fromSource=gwzcw.59733.59733.59733")

