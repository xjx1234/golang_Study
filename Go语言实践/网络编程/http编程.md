# Http网络编程



## 1. http Service

### 1.1 创建 http Service

对于Golang来说，实现一个简单的`http server`非常容易，只需要短短几行代码。同时有了协程的加持，Go实现的`http server`能够取得非常优秀的性能。这篇文章将会对go标准库`net/http`实现http服务的原理进行较为深入的探究，以此来学习了解网络编程的常见范式以及设计思路。

基于HTTP构建的网络应用包括两个端，即客户端(`Client`)和服务端(`Server`)。两个端的交互行为包括从客户端发出`request`、服务端接受`request`进行处理并返回`response`以及客户端处理`response`。所以http服务器的工作就在于如何接受来自客户端的`request`，并向客户端返回`response`。

典型的http server 的处理流程可以用下图表示：

![](https://myvoice1.oss-cn-beijing.aliyuncs.com/github/bfd4c9f1f3a224e07b27dd7ad7e15b8a.png)

服务器在接收到请求时，首先会进入路由(`router`)，这是一个`Multiplexer`，路由的工作在于为这个`request`找到对应的处理器(`handler`)，处理器对`request`进行处理，并构建`response`。Golang实现的`http server`同样遵循这样的处理流程。

举例说明一个最简单的 开启 server例子，我们成为V1版本的server：

直接使用 http.HandleFunc(partern,function(http.ResponseWriter,*http.Request){}) 

HandleFunc接受两个参数，第一个为路由地址，第二个为处理方法。

```go
package main

import (
	"net/http"
)

//定义处理函数
func TestFunc(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("v1, Hello world!!!"))
}

func main() {
	http.HandleFunc("/test", TestFunc) //调用HandleFunc处理
	http.ListenAndServe(":8888", nil) //监听端口
}
```



通过上面简简单单的几句代码，就可以启动一个 Http server服务，但是这种方式都是调用默认的http相关参数实现的，并不能很好的自己把控server个性化操作，这个那就得从源码层面分析，借鉴 `HandleFunc` 方法的实现来自己定义一些server细节，`HandleFunc`代码实现为：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler) // 默认使用DefaultServeMux
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler)) //最终调用实现
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
	......
}
```

通过源码代码得知，`HandleFunc` 实现主要调用`mux.Handle`  函数实现, 而  `func (mux *ServeMux) Handle(pattern string, handler Handler)` 主要有几个要点组成：

-  `ServeMux` 

  从源码函数 `HandleFunc`  代码中的 `DefaultServeMux.HandleFunc(pattern, handler)`   这句代码来看， `HandleFunc` 方法使用的 `ServeMux` 是默认的`DefaultServeMux` ，ServeMux 以及 DefaultServeMux 源码如下：

  ```go
  type ServeMux struct {
  	mu    sync.RWMutex
  	m     map[string]muxEntry
  	es    []muxEntry // slice of entries sorted from longest to shortest.
  	hosts bool       // whether any patterns contain hostnames
  }
  
  func NewServeMux() *ServeMux { return new(ServeMux) }
  
  // DefaultServeMux is the default ServeMux used by Serve.
  var DefaultServeMux = &defaultServeMux
  
  var defaultServeMux ServeMux
  ```

  通过 对 `ServeMux  与 DefaultServeMux` 的相关源码解析来看，我们可以通过 `NewServeMux` 函数 新建一个属于自己的 `ServeMux` 来替代 `DefaultServeMux`，然后自己定义的 `ServeMux` 来直接调用函数 `Handle` 来替代  `http.HandleFunc` ， 即：

  ```go
  mux := http.NewServeMux() //自定义 NewServeMux
  ```

- `Handler` 对象

  由最终的 `func (mux *ServeMux) Handle(pattern string, handler Handler)` 函数定义来看，最终需要把自定义的 handler方法 转化为 Handler对象， 上面例子中的 `TestFunc`自定义处理函数，最终通过 HandlerFunc 强制转换为 Handler类型，源码如下：

  ```go
  type HandlerFunc func(ResponseWriter, *Request)
  
  // ServeHTTP calls f(w, r).
  func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  	f(w, r)
  }
  
  //Handler 接口对象
  type Handler interface {
  	ServeHTTP(ResponseWriter, *Request)
  }
  ```

  从 `Handler`接口对象看来，外部只需要实现 `ServeHTTP(ResponseWriter, *Request)` 方法即可满足 `mux.Handle`的调用，那么这个就简单了，我们通过自定义 ServeHTTP 来实现更多维度的操作变得更容易了，代码如下：

  ```go
  type myHandler struct{} //自定义 Handler
  
  //实现 ServeHTTP方法
  func (*myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  	......
  }
  ```

通过对 `ServeMux`  以及 `Handler` 对象 分析，我们可以组装出更自定义化的 V2版本的server，代码如下：

```go
package main

import (
	"net/http"
)

type myHandler struct{}  //自定义Handler

//实现Handler的ServeHTTP方法
func (*myHandler) ServeHTTP方法(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("this is version 2"))
}

func main() {
	mux := http.NewServeMux() //自定义ServeMux 
	mux.Handle("/test2", &myHandler{}) 
	http.ListenAndServe(":8888", mux)
}
```

除了 对  `ServeMux`  以及 `Handler` 对象的自定义，我们还可以更深入的对 `http.ListenAndServe`中的 `Server`对象就行自定义，源码如下：

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

type Server struct {
 Addr string // TCP address to listen on, ":http" if empty
 Handler Handler // handler to invoke, http.DefaultServeMux if nil
 TLSConfig *tls.Config
 ReadTimeout time.Duration
 ReadHeaderTimeout time.Duration
 WriteTimeout time.Duration
 IdleTimeout time.Duration
 MaxHeaderBytes int
 TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
 ConnState func(net.Conn, ConnState)
 ErrorLog *log.Logger
 
 disableKeepAlives int32  // accessed atomically.
 inShutdown  int32  // accessed atomically (non-zero means we're in Shutdown)
 nextProtoOnce  sync.Once // guards setupHTTP2_* init
 nextProtoErr  error  // result of http2.ConfigureServer if used

 mu   sync.Mutex
 listeners map[*net.Listener]struct{}
 activeConn map[*conn]struct{}
 doneChan chan struct{}
 onShutdown []func()
}

```

从 `ListenAndServe`中  `&Server{Addr: addr, Handler: handler}`  代码看出，http默认实例化了 Server结构体中的  `Addr` 和 `Handler` ，如果想实例化更多Server对象值，则可以自定义实现，即 V3版本的server：

```go
package main

import (
	"net/http"
)

type myHandler struct{}  //自定义Handler

//实现Handler的ServeHTTP方法
func (*myHandler) ServeHTTP方法(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("this is version 2"))
}

func main() {
	mux := http.NewServeMux() //自定义ServeMux 
	mux.Handle("/test2", &myHandler{}) 
	server := &http.Server{
		Addr:         ":8888",
		WriteTimeout: time.Second * 3,            //设置3秒的写超时
		Handler:      mux,
	}
	server.ListenAndServe()
}
```

### 1.2 参数获取

- GET参数获取

  ```go
  r.URL.Query().Get("key名")
  ```

- application/x-www-form-urlencoded模式下获取 POST  GET PUT DELETE 值

  ```go
  r.ParseForm()
  r.Form.Get("key名")  // get 或者 post值
  r.PostForm.Get("key名")  //只支持post
  ```

- multipart/form-data 模式下获取  POST值

  ```go
  r.ParseMultipartForm(100)
  r.Form.Get("key名")  // get 或者 post值
  r.PostForm.Get("key名")  //只支持post
  ```

- Header获取

  ```go
  r.Header.Get("key名")
  
  //遍历Header
  if len(r.Header) > 0 {
        for k,v := range r.Header {
           fmt.Printf("%s=%s\n", k, v[0])
        }
     }
  ```

  

- Cookies获取

  ```go
  cookie, err := r.Cookie("key名")
  if err == nil {
  	fmt.Fprintln(w, "Domain:", cookie.Domain)
  	fmt.Fprintln(w, "Expires:", cookie.Expires)
  	fmt.Fprintln(w, "Name:", cookie.Name)
  	fmt.Fprintln(w, "Value:", cookie.Value)
  
  ```



## 2. http Client

已经简单的从源码以及示例角度演示了Http Server方面的操作，既然有了Server端操作，肯定也少不了Client端操作。

### 2.1 GET POST简单使用

我们看下一个最简单的Http Get请求， 代码如下:

```go
	resp, err := http.Get("http://www.baidu.com")
	fmt.Println(resp)
	fmt.Println(err)
```

继续看下POST请求方式，post请求主要操作主要有以下2种方式：

- http.Post

  ```go
  	body := strings.NewReader("apiKey=pHxuVfGa78420dc9d7a51359516312c052f2ff637a32f2c&postcode=324013&page=1&pageSize=1")
  	resp, error := http.Post("http://api.apishop.net/common/postcode/getAddrByPostcode", "application/x-www-form-urlencoded", body)
  	fmt.Println(resp, error)
  ```

- http.PostForm

  ```go
  	values := url.Values{"apiKey":{"pHxuVfGa78420dc9d7a51359516312c052f2ff637a32f2c"}, "postcode":{"324013"}, "page":{"1"}, "pageSize":{"15"}}
  	resp, error := http.PostForm("http://api.apishop.net/common/postcode/getAddrByPostcode", values)
  	fmt.Println(resp, error)
  ```

### 2.2 自定义Client使用

上述提到的GET POST 请求方法能够应对常用的一些http请求，但应对更复杂的情况就很难胜任，例如需要设置header cookies等数据。所以我们就要从源码层面看下了解下，下面列举了源码层面调用的涉及到的一些常用函数和方法代码：

```go
//http.Get源码
func (c *Client) Get(url string) (resp *Response, err error) {
	req, err := NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	return c.Do(req)
}

//http.Post函数源码
func Post(url, contentType string, body io.Reader) (resp *Response, err error) {
	return DefaultClient.Post(url, contentType, body)
}

//http.PostForm函数源码
func (c *Client) PostForm(url string, data url.Values) (resp *Response, err error) {
	return c.Post(url, "application/x-www-form-urlencoded", strings.NewReader(data.Encode()))
}

//client.Post函数源码
func (c *Client) Post(url, contentType string, body io.Reader) (resp *Response, err error) {
	req, err := NewRequest("POST", url, body)
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", contentType)
	return c.Do(req)
}

//client.Do函数源码
func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}
```

经过查看源码，总结一下默认自带函数  `http.Get`  `http.Post`  `http.PostForm` 的操作流程：

1.  建立 默认http客户端`DefaultClient`
2. 构建 Request请求包数据，通过 `NewRequest`函数
3. 通过 `c.do()`  提交 Request数据，返回 Response以及错误信息数据

该流程中，我们可以在建立http客户端以及构建 Request请求包数据这2个操作做很多参数自定义操作，所以我们调用系统函数自行实现http客户端 以及 Request构造改造可以带来更多的操作自主性。

- 自定义构建http client

  参考 `DefaultClient`构造生成以及 Client结构体源码，如下：

  ```go
  type Client struct {
  	Transport RoundTripper  //Transport用于确定HTTP请求的创建机制。如果为空，将会使用DefaultTransport
    	/* CheckRedirect定义重定向策略。
       如果CheckRedirect不为空，客户端将在跟踪HTTP重定向前调用该函数。
       两个参数req和via分别为即将发起的请求和已经发起的所有请求，最早的
       已发起请求在最前面。
       如果CheckRedirect返回错误，客户端将直接返回错误，不会再发起该请求。
       如果CheckRedirect为空， Client将采用一种确认策略，将在10个连续
       请求后终止 */
  	CheckRedirect func(req *Request, via []*Request) error 
  	Jar CookieJar //如果Jar为空， Cookie将不会在请求中发送，并会在响应中被忽略
  	Timeout time.Duration //请求超时时间
  }
  
  var DefaultClient = &Client{}
  ```

  我们建立自己一个自己的 `Client` ，这边暂时不管 Client结构体的一些变量的用法，我们只做简单的示例：

  ```go
  	client := &http.Client{
  		Timeout: time.Second * 30,
  	}
  ```

-  构建 Request请求包数据

  根据需求，自定义构造一个 Request请求数据

  ```go
  	header := map[string]interface{}{"Content-Type": "application/x-www-form-urlencoded", "X-Xsrftoken":"b6d695bbdcd111e8b681002324e63af81"}
  	url := "http://api.apishop.net/common/postcode/getAddrByPostcode"
  	body := strings.NewReader("apiKey=pHxuVfGa78420dc9d7a51359516312c052f2ff637a32f2c&postcode=324013&page=1&pageSize=1")
  	request, err := http.NewRequest("POST", url, body) //构造 NewRequest
  	if err == nil {
  		for t, v := range header {
  			request.Header.Add(t, fmt.Sprint(v)) //增加header
  		}
  		cookies := &http.Cookie{Name: "X-Xsrftoken",Value: "df41ba54db5011e89861002324e63af81", HttpOnly: true}
  		request.AddCookie(cookies)  //增加cookies
  	}
  ```

- 发送请求

  ```go
  	res, err := client.Do(request)//用自己创建的 client 发送 自己构造的 request包
  	fmt.Println(res, err)
  ```

完成了自己构造请求发送后，因为http返回的数据是一个 `Response` 数据包， 我们无法直观的看到的数据，我们在代码后面增加上数据解析部分，这样就基本完成了一个自定义的请求操作，完整代码如下：

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
	"time"
)

func main() {
	//构建client
	client := &http.Client{
		Timeout: time.Second * 30,
	}
	header := map[string]interface{}{"Content-Type": "application/x-www-form-urlencoded", "X-Xsrftoken": "b6d695bbdcd111e8b681002324e63af81"}
	url := "http://api.apishop.net/common/postcode/getAddrByPostcode"
	body := strings.NewReader("apiKey=pHxuVfGa78420dc9d7a51359516312c052f2ff637a32f2c&postcode=324013&page=1&pageSize=1")
	request, err := http.NewRequest("POST", url, body) //构造 NewRequest
	if err == nil {
		for t, v := range header {
			request.Header.Add(t, fmt.Sprint(v)) //增加header
		}
		cookies := &http.Cookie{Name: "X-Xsrftoken", Value: "df41ba54db5011e89861002324e63af81", HttpOnly: true}
		request.AddCookie(cookies)      //增加cookies
		resp, err := client.Do(request) //发送请求
		if err != nil {
			fmt.Errorf(err.Error())
		} else {
			defer resp.Body.Close()
			body, err := ioutil.ReadAll(resp.Body) //读取解析 resp.Body 数据
			if err == nil {
				fmt.Println(string(body)) //打印最后数据
			} else {
				fmt.Errorf(err.Error())
			}
		}
	} else {
		fmt.Errorf("http do error!")
	}
}
```

运行结果：

```
{"statusCode":"000000","desc":"请求成功","result":{"list":[{"PostNumber":"324013","Province":"浙江省","City":"衢州市","Dist
rict":"衢江区","Address":"后溪镇百灵街村"}],"totalCount":25,"totalPage":25,"currentPage":1,"pageSize":1}}
```

