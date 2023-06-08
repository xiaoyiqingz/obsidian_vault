
### 前言

对于 Golang 来说，实现一个简单的 `http server` 非常容易，只需要短短几行代码。同时有了协程的加持，Go 实现的 `http server` 能够取得非常优秀的性能。这篇文章将会对 go 标准库 `net/http` 实现 http 服务的原理进行较为深入的探究，以此来学习了解网络编程的常见范式以及设计思路。

### HTTP 服务

基于 HTTP 构建的网络应用包括两个端，即客户端 ( `Client` ) 和服务端 ( `Server` )。两个端的交互行为包括从客户端发出 `request`、服务端接受 `request` 进行处理并返回 `response` 以及客户端处理 `response`。所以 http 服务器的工作就在于如何接受来自客户端的 `request`，并向客户端返回 `response`。


服务器在接收到请求时，首先会进入路由 ( `router` )，这是一个 `Multiplexer`，路由的工作在于为这个 `request` 找到对应的处理器 ( `handler` )，处理器对 `request` 进行处理，并构建 `response`。Golang 实现的 `http server` 同样遵循这样的处理流程。

我们先看看 Golang 如何实现一个简单的 `http server`：

```go
package main

import (
    "fmt"
    "net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world")
}

func main() {
    http.HandleFunc("/", indexHandler)
    http.ListenAndServe(":8000", nil)
}
```

运行代码之后，在浏览器中打开 `localhost:8000` 就可以看到 `hello world`。这段代码先利用 `http.HandleFunc` 在根路由 `/` 上注册了一个 `indexHandler`, 然后利用 `http.ListenAndServe` 开启监听。当有请求过来时，则根据路由执行对应的 `handler` 函数。

我们再来看一下另外一种常见的 `http server` 实现方式：

```go
package main

import (
    "fmt"
    "net/http"
)

type indexHandler struct {
    content string
}

func (ih *indexHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, ih.content)
}
func main() {
    http.Handle("/", &indexHandler{content: "hello world!"})
    http.ListenAndServe(":8001", nil)
}
```

Go 实现的 `http` 服务步骤非常简单，首先注册路由，然后创建服务并开启监听即可。下文我们将从注册路由、开启服务、处理请求这几个步骤了解 Golang 如何实现 `http` 服务。

### 注册路由

`http.HandleFunc` 和 `http.Handle` 都是用于注册路由，可以发现两者的区别在于第二个参数，前者是一个具有 `func(w http.ResponseWriter, r *http.Requests)` 签名的函数，而后者是一个结构体，该结构体实现了 `func(w http.ResponseWriter, r *http.Requests)` 签名的方法。  
`http.HandleFunc` 和 `http.Handle` 的源码如下：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    mux.Handle(pattern, HandlerFunc(handler))
}
```

```go
func Handle(pattern string, handler Handler) {
    DefaultServeMux.Handle(pattern, handler)
}
```

可以看到这两个函数最终都由 `DefaultServeMux` 调用 `Handle` 方法来完成路由的注册。  
这里我们遇到两种类型的对象：`ServeMux` 和 `Handler`，我们先说 `Handler`。

##### Handler

`Handler` 是一个接口：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`Handler` 接口中声明了名为 `ServeHTTP` 的函数签名，也就是说任何结构只要实现了这个 `ServeHTTP` 方法，那么这个结构体就是一个 `Handler` 对象。其实 go 的 `http` 服务都是基于 `Handler` 进行处理，而 `Handler` 对象的 `ServeHTTP` 方法也正是用以处理 `request` 并构建 `response` 的核心逻辑所在。

回到上面的 `HandleFunc` 函数，注意一下这行代码：

```go
mux.Handle(pattern, HandlerFunc(handler))
```

可能有人认为 `HandlerFunc` 是一个函数，包装了传入的 `handler` 函数，返回了一个 `Handler` 对象。然而这里 `HandlerFunc` 实际上是将 `handler` 函数做了一个**类型转换**，看一下 `HandlerFunc` 的定义：

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

`HandlerFunc` 是一个类型，只不过表示的是一个具有 `func(ResponseWriter, *Request)` 签名的函数类型，并且这种类型实现了 `ServeHTTP` 方法（在 `ServeHTTP` 方法中又调用了自身），也就是说这个类型的函数其实就是一个 `Handler` 类型的对象。利用这种类型转换，我们可以将一个 `handler` 函数转换为一个  
`Handler` 对象，而不需要定义一个结构体，再让这个结构实现 `ServeHTTP` 方法。读者可以体会一下这种技巧。

##### ServeMux

Golang 中的路由（即 `Multiplexer` ）基于 `ServeMux` 结构，先看一下 `ServeMux` 的定义：

```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
    h       http.Handler
    pattern string
}
```

这里重点关注 `ServeMux` 中的字段 `m`，这是一个 `map`，`key` 是路由表达式，`value` 是一个 `muxEntry` 结构， `muxEntry` 结构体存储了对应的路由表达式和 `handler`。

值得注意的是，`ServeMux` 也实现了 `ServeHTTP` 方法：

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
```

也就是说 `ServeMux` 结构体也是 `Handler` 对象，只不过 `ServeMux` 的 `ServeHTTP` 方法不是用来处理具体的 `request` 和构建 `response`，而是用来确定路由注册的 `handler`。

##### 注册路由

搞明白 `Handler` 和 `ServeMux` 之后，我们再回到之前的代码：

```go
DefaultServeMux.Handle(pattern, handler)
```

这里的 `DefaultServeMux` 表示一个默认的 `Multiplexer`，当我们没有创建自定义的 `Multiplexer`，则会自动使用一个默认的 `Multiplexer`。

然后再看一下 `ServeMux` 的 `Handle` 方法具体做了什么：

```go
func (mux *ServeMux) Handle(patternstring, handlerHandler) {
    mux.mu.Lock()
    defermux.mu.Unlock()
    if pattern == "" {
        panic("http:invalidpattern")
    }
    if handler == nil {
        panic("http:nilhandler")
    }
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }
    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    //利用当前的路由和handler创建muxEntry对象
    e := muxEntry{h: handler, pattern: pattern}
    //向ServeMux的map[string]muxEntry增加新的路由匹配规则
    mux.m[pattern] = e
    //如果路由表达式以'/'结尾，则将对应的muxEntry对象加入到[]muxEntry中，按照路由表达式长度排序
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }
    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```

`Handle` 方法主要做了两件事情：一个就是向 `ServeMux` 的 `map[string]muxEntry` 增加给定的路由匹配规则；然后如果路由表达式以 `'/'` 结尾，则将对应的 `muxEntry` 对象加入到 `[]muxEntry` 中，按照路由表达式长度排序。前者很好理解，但后者可能不太容易看出来有什么作用，这个问题后面再作分析。

##### 自定义 ServeMux

我们也可以创建自定义的 `ServeMux` 取代默认的 `DefaultServeMux`：

```go
package main

import (
    "fmt"
    "net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello  world")
}
func htmlHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/html")
    html := `<!doctype  html>  
    <META  http-equiv="Content-Type"  content="text/html"  charset="utf-8">    
    <html  lang="zhCN">  
            <head>   
                    <title>Golang</title>  
                    <meta  name="viewport"  content="width=device-width,  initial-scale=1.0,  maximum-scale=1.0,  user-scalable=0;"  />   
            </head>     
            <body>        
            <div  id="app">Welcome!</div>       
            </body>   
    </html>`
    fmt.Fprintf(w, html)
}
func main() {
    mux := http.NewServeMux()
    mux.Handle("/", http.HandlerFunc(indexHandler))
    mux.HandleFunc("/welcome", htmlHandler)
    http.ListenAndServe(":8001", mux)
}
```

`NewServeMux()` 可以创建一个 `ServeMux` 实例，之前提到 `ServeMux` 也实现了 `ServeHTTP` 方法，因此 `mux` 也是一个 `Handler` 对象。对于 `ListenAndServe()` 方法，如果传入的 `handler` 参数是自定义 `ServeMux` 实例 `mux`，那么 `Server` 实例接收到的路由对象将不再是 `DefaultServeMux` 而是 `mux`。

### 开启服务

首先从 `http.ListenAndServe` 这个方法开始：

```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```

这里先创建了一个 `Server` 对象，传入了地址和 `handler` 参数，然后调用 `Server` 对象 `ListenAndServe()` 方法。

看一下 `Server` 这个结构体，`Server` 结构体中字段比较多，可以先大致了解一下：

```go
type Server struct {
    Addr              string  // TCP address to listen on, ":http" if empty
    Handler           Handler // handler to invoke, http.DefaultServeMux if nil
    TLSConfig         *tls.Config
    ReadTimeout       time.Duration
    ReadHeaderTimeout time.Duration
    WriteTimeout      time.Duration
    IdleTimeout       time.Duration
    MaxHeaderBytes    int
    TLSNextProto      map[string]func(*Server, *tls.Conn, Handler)
    ConnState         func(net.Conn, ConnState)
    ErrorLog          *log.Logger
    disableKeepAlives int32     // accessed atomically.
    inShutdown        int32     // accessed atomically (non-zero means we're in Shutdown)
    nextProtoOnce     sync.Once // guards setupHTTP2_* init
    nextProtoErr      error     // result of http2.ConfigureServer if used
    mu                sync.Mutex
    listeners         map[*net.Listener]struct{}
    activeConn        map[*conn]struct{}
    doneChan          chan struct{}
    onShutdown        []func()
}
```

在 `Server` 的 `ListenAndServe` 方法中，会初始化监听地址 `Addr`，同时调用 `Listen` 方法设置监听。最后将监听的 TCP 对象传入 `Serve` 方法：

```go
func (srv *Server) Serve(l net.Listener) error {
    ...
    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, e := l.Accept() // 等待新的连接建立
        ...
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)             // 创建新的协程处理请求
    }
}
```

这里隐去了一些细节，以便了解 `Serve` 方法的主要逻辑。首先创建一个上下文对象，然后调用 `Listener` 的 `Accept()` 等待新的连接建立；一旦有新的连接建立，则调用 `Server` 的 `newConn()` 创建新的连接对象，并将连接的状态标志为 `StateNew`，然后开启一个新的 `goroutine` 处理连接请求。

### 处理连接

我们继续探索 `conn` 的 `serve()` 方法，这个方法同样很长，我们同样只看关键逻辑。坚持一下，马上就要看见大海了。

```go
func (c *conn) serve(ctx context.Context) {
    ...
    for {
        w, err := c.readRequest(ctx)
        if c.r.remain != c.server.initialReadLimitSize() {
            // If we read any bytes off the wire, we're active.
            c.setState(c.rwc, StateActive)
        }
        ...
        // HTTP cannot have multiple simultaneous active requests.[*]
        // Until the server replies to this request, it can't read another,
        // so we might as well run the handler in this goroutine.
        // [*] Not strictly true: HTTP pipelining. We could let them all process
        // in parallel even if their responses need to be serialized.
        // But we're not going to implement HTTP pipelining because it
        // was never deployed in the wild and the answer is HTTP/2.
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        if c.hijacked() {
            return
        }
        w.finishRequest()
        if !w.shouldReuseConnection() {
            if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
                c.closeWriteAndWait()
            }
            return
        }
        c.setState(c.rwc, StateIdle)
        c.curReq.Store((*response)(nil))
        ...
    }
}
```

当一个连接建立之后，该连接中所有的请求都将在这个协程中进行处理，直到连接被关闭。在 `serve()` 方法中会循环调用 `readRequest()` 方法读取下一个请求进行处理，其中最关键的逻辑就是一行代码：

```go
serverHandler{c.server}.ServeHTTP(w, w.req)
```

进一步解释 `serverHandler`：

```go
type serverHandler struct {
    srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

在 `serverHandler` 的 `ServeHTTP()` 方法里的 `sh.srv.Handler` 其实就是我们最初在 `http.ListenAndServe()` 中传入的 `Handler` 对象，也就是我们自定义的 `ServeMux` 对象。如果该 `Handler` 对象为 `nil`，则会使用默认的 `DefaultServeMux`。最后调用 `ServeMux` 的 `ServeHTTP()` 方法匹配当前路由对应的 `handler` 方法。

后面的逻辑就相对简单清晰了，主要在于调用 `ServeMux` 的 `match` 方法匹配到对应的已注册的路由表达式和 `handler`。

```go
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()
    // Host-specific pattern takes precedence over generic ones
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}

// Find a handler on a handler map given a path string.
// Most-specific (longest) pattern wins.
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    // Check for exact match first.
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }
    // Check for longest valid match.  mux.es contains all patterns
    // that end in / sorted from longest to shortest.
    for _, e := range mux.es {
        if strings.HasPrefix(path, e.pattern) {
            return e.h, e.pattern
        }
    }
    return nil, ""
}
```

在 `match` 方法里我们看到之前提到的 mux 的 `m` 字段 (类型为 `map[string]muxEntry` ) 和 `es` (类型为 `[]muxEntry`)。这个方法里首先会利用进行精确匹配，在 `map[string]muxEntry` 中查找是否有对应的路由规则存在；如果没有匹配的路由规则，则会利用 `es` 进行近似匹配。

之前提到在注册路由时会把以 `'/'` 结尾的路由（可称为**节点路由**）加入到 `es` 字段的 `[]muxEntry` 中。对于类似 `/path1/path2/path3` 这样的路由，如果不能找到精确匹配的路由规则，那么则会去匹配和当前路由最接近的已注册的父节点路由，所以如果路由 `/path1/path2/` 已注册，那么该路由会被匹配，否则继续匹配下一个父节点路由，直到根路由 `/`。

由于 `[]muxEntry` 中的 `muxEntry` 按照路由表达式从长到短排序，所以进行近似匹配时匹配到的节点路由一定是已注册父节点路由中最相近的。

至此，Go 实现的 `http server` 的大致原理介绍完毕！

### 总结

Golang 通过 `ServeMux` 定义了一个多路器来管理路由，并通过 `Handler` 接口定义了路由处理函数的统一规范，即 `Handler` 都须实现 `ServeHTTP` 方法；同时 `Handler` 接口提供了强大的扩展性，方便开发者通过 `Handler` 接口实现各种中间件。相信大家阅读下来也能感受到 `Handler` 对象在 `server` 服务的实现中真的无处不在。理解了 `server` 实现的基本原理，大家就可以在此基础上阅读一些第三方的 `http server` 框架，以及编写特定功能的中间件。