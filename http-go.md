# goroutine
简单来说，Goroutine 就是 Golang 里的轻量级线程，它比操作系统级别的线程要“轻”得多，创建一个 Goroutine 仅需几 KB 的栈空间，而不是像传统线程那样动辄占用几 MB。Golang 运行时会自动管理它们的栈大小，并在需要时动态扩容，这使得 Goroutine 非常适合高并发场景。

更重要的是，Goroutine 的调度是由 Golang 自己的运行时（runtime）管理的，而不是依赖操作系统的线程调度器。换句话说，Go 自己实现了一套“迷你版的操作系统”来管理 Goroutine
### 创建
`go f(a,b,c)`
```

func main() {
  for i := 0; i < 5; i++ {
		go func() {
      // 可以看到打印的结果是乱序的 并且也是奇怪的数字 因为闭包中的变量是共享的
			fmt.Println(i)
		}()
		go func(n int) {
      // 可以看到打印的结果是乱序的 但是数字是正常的
			fmt.Println(n)
		}(i)
	}
  time.Sleep(time.Second)
}
```

# http

Request：用户请求的信息，用来解析用户的请求信息，包括post、get、cookie、url等信息

Response：服务器需要反馈给客户端的信息

Conn：用户的每次请求链接

Handler：处理请求和生成返回信息的处理逻辑

# http代码的执行流程
通过对http包的分析之后，现在让我们来梳理一下整个的代码执行过程。

首先调用Http.HandleFunc

按顺序做了几件事：

1 调用了DefaultServeMux的HandleFunc

2 调用了DefaultServeMux的Handle

3 往DefaultServeMux的map[string]muxEntry中增加对应的handler和路由规则

其次调用http.ListenAndServe(":9090", nil)

按顺序做了几件事情：

1 实例化Server

2 调用Server的ListenAndServe()

3 调用net.Listen("tcp", addr)监听端口

4 启动一个for循环，在循环体中Accept请求

5 对每个请求实例化一个Conn，并且开启一个goroutine为这个请求进行服务go c.serve()

6 读取每个请求的内容w, err := c.readRequest()

7 判断handler是否为空，如果没有设置handler（这个例子就没有设置handler），handler就设置为DefaultServeMux

8 调用handler的ServeHttp

9 在这个例子中，下面就进入到DefaultServeMux.ServeHttp

10 根据request选择handler，并且进入到这个handler的ServeHTTP

mux.handler(r).ServeHTTP(w, r)
11 选择handler：

A 判断是否有路由能满足这个request（循环遍历ServeMux的muxEntry）

B 如果有路由满足，调用这个路由handler的ServeHTTP

C 如果没有路由满足，调用NotFoundHandler的ServeHTTP

## 基本使用

```
func httpHandler(w http.ResponseWriter, r *http.Request) {

	r.ParseForm() // 解析参数
	fmt.Println(r.Form)
	fmt.Println("path", r.URL.Path)
	fmt.Println("scheme", r.URL.Scheme)
	fmt.Println(r.Form["url_long"])
	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("val:", strings.Join(v, ""))
	}
	fmt.Fprintf(w, "Hello astaxie!")
}
func main() {
	http.HandleFunc("/", httpHandler)
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatal("ListenAndServe", err)
	}
}
```

### 源码分析
ListenAndServe会初始化一个sever对象，然后调用了Server对象的方法ListenAndServe
```
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```
server 底层用TCP协议搭建了一个服务，最后调用src.Serve 接收Listen 监控我们设置的端口
```
func (s *Server) ListenAndServe() error {
	if s.shuttingDown() {
		return ErrServerClosed
	}
	addr := s.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return s.Serve(ln)
}
```
Serve内部for 死循环 
通过Listener接收请求：l.Accept()，
其次创建一个Conn：c := srv.newConn(rw)，
最后单独开了一个goroutine，
把这个请求的数据当做参数扔给这个conn去服务：go c.serve(connCtx)。
这个就是高并发体现了，用户的每一次请求都是在一个新的goroutine去服务，相互不影响
```
func (s *Server) Serve(l net.Listener) error {
	...
	ctx := context.WithValue(baseCtx, ServerContextKey, s)
	for {
		rw, err := l.Accept()
		...
		connCtx := ctx
		if cc := s.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := s.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}

```
conn首先会解析request:w, err := c.readRequest(ctx), 然后获取相应的handler去处理请求:serverHandler{c.server}.ServeHTTP(w, w.req)，ServeHTTP的具体实现如下：
```
func (c *conn) serve(ctx context.Context) {
    ...

	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		w, err := c.readRequest(ctx)
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
        ...

	}
}
```
```
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
![](images/http-go.png)

sh.srv.Handler就是我们刚才在调用函数ListenAndServe时候的第二个参数，
我们前面例子传递的是nil，也就是为空，那么默认获取handler = DefaultServeMux,
那么这个变量用来做什么的呢？对，这个变量就是一个路由器，它用来匹配url跳转到其相应的handle函数，
那么这个我们有设置过吗?有，我们调用的代码里面第一句不是调用了http.HandleFunc("/", sayhelloName)嘛。
这个作用就是注册了请求/的路由规则，当请求uri为"/"，路由就会转到函数sayhelloName，DefaultServeMux会调用ServeHTTP方法，
这个方法内部其实就是调用sayhelloName本身，最后通过写入response的信息反馈到客户端。






# ServeMux

conn.server 内部是调用了http包默认的路由器，通过路由器把本次请求的信息传递到了后端的处理函数
```
type ServeMux struct {
	mu sync.RWMutex   //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
	m  map[string]muxEntry  // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
	hosts bool // 是否在任意的规则中带有host信息
}
```
```
type muxEntry struct {
	explicit bool   // 是否精确匹配
	h        Handler // 这个路由表达式对应哪个handler
	pattern  string  //匹配字符串
}
```
```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)  // 路由实现器
}
```

Handler是一个接口，
sayhelloName函数并没有实现ServeHTTP这个接口，为什么能添加呢？
原来在http包里面还定义了一个类型HandlerFunc,我们定义的函数sayhelloName就是这个HandlerFunc调用之后的结果，
这个类型默认就实现了ServeHTTP这个接口，即我们调用了HandlerFunc(f),
强制类型转换f成为HandlerFunc类型，这样f就拥有了ServeHTTP方法。

```
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

如上所示路由器接收到请求之后，如果是*那么关闭链接，不然调用mux.Handler(r)返回对应设置路由的处理Handler，然后执行h.ServeHTTP(w, r)

也就是调用对应路由的handler的ServerHTTP接口，那么mux.Handler(r)怎么处理的呢？

路由器里面存储好了相应的路由规则之后，那么具体的请求又是怎么分发的呢？请看下面的代码，默认的路由器实现了ServeHTTP：
```
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		w.Header().Set("Connection", "close")
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

如上所示路由器接收到请求之后，如果是*那么关闭链接，不然调用mux.Handler(r)返回对应设置路由的处理Handler，然后执行h.ServeHTTP(w, r)

也就是调用对应路由的handler的ServerHTTP接口，那么mux.Handler(r)怎么处理的呢？
```

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	if r.Method != "CONNECT" {
		if p := cleanPath(r.URL.Path); p != r.URL.Path {
			_, pattern = mux.handler(r.Host, p)
			return RedirectHandler(p, StatusMovedPermanently), pattern
		}
	}	
	return mux.handler(r.Host, r.URL.Path)
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
```

原来他是根据用户请求的URL和路由器里面存储的map去匹配的，当匹配到之后返回存储的handler，调用这个handler的ServeHTTP接口就可以执行到相应的函数了。

通过上面这个介绍，我们了解了整个路由过程，Go其实支持外部实现的路由器 ListenAndServe的第二个参数就是用以配置外部路由器的，它是一个Handler接口，即外部路由器只要实现了Handler接口就可以,我们可以在自己实现的路由器的ServeHTTP里面实现自定义路由功能。

如下代码所示，我们自己实现了一个简易的路由器

```

package main

import (
	"fmt"
	"net/http"
)

type MyMux struct {
}

func (p *MyMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path == "/" {
		sayhelloName(w, r)
		return
	}
	http.NotFound(w, r)
	return
}

func sayhelloName(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello myroute!")
}

func main() {
	mux := &MyMux{}
	http.ListenAndServe(":9090", mux)
}
```

