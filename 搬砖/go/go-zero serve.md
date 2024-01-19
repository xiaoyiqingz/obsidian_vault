
1.  `zrpc.MustNewServer(c RpcServerConf, register internal.RegisterFn) *RpcServer`

2.  `zrpc.RpcServer`

```go
type RpcServer struct {
	server   internal.Server
	register internal.RegisterFn
} 
```

3. `internal.Server is interface` 

```go
if c.HasEtcd() {
		server, err = internal.NewRpcPubServer(c.Etcd, c.ListenOn, c.Middlewares, serverOptions...)
		} else {
		server = internal.NewRpcServer(c.ListenOn, c.Middlewares, serverOptions...)
	}

internal.Server = internal.keepAliveServer or
internal.Server = internal.rpcServer
```

4.  `internal.keepAliveServer`
```go
type keepAliveServer struct {
	registerEtcd func() error
	Server internal.Server
}

此处 
internal.Server = NewRpcServer(listenOn, middlewares, opts...)
即
internal.Server = internal.rpcServer
```

5. `zrpc.RpcServer`
```go
type RpcServer struct {
	server : internal.KeepAliveServer {
		registerEtcd func() error
		Server: internal.rpcServer{
			*internal.baseRpcServer
			name          string
			middlewares   ServerMiddlewaresConf
			healthManager health.Probe
		}
	}
	register: internal.RegisterFn
}

当 c.HasEtcd() = false 时

type RpcServer struct {
	server : internal.rpcServer{
		*internal.baseRpcServer
		name          string
		middlewares   ServerMiddlewaresConf
		healthManager health.Probe
	}
	register: internal.RegisterFn
}

```

5.  `internal.baseRpcServer`
```go
	baseRpcServer struct {
		address            string
		health             *health.Server
		metrics            *stat.Metrics
		options            []grpc.ServerOption
		streamInterceptors []grpc.StreamServerInterceptor
		unaryInterceptors  []grpc.UnaryServerInterceptor
	}
```

7.  `zrpc.Server.Start()` 过程
* `zrpc.Server.Start()`  ->  `internal.KeepAliveServer.Start()` -> `internal.rpcServer.Start()`
* `zrpc.Server.Start()` -> `internal.rpcServer.Start()`

`internal.rpcServer.Start()` 中 `grpc.NewServer().Serve()`

9. `zrpc.Server.Stop()`  -> `logx.Close()`