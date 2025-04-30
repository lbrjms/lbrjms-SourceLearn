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



