# WebSocket编程



## 1.  WebSocket概念

WebSocket 协议在2008年诞生，2011年成为国际标准。现在所有浏览器都已经支持了。WebSocket 的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话。

由于使用WebSocket使用HTTP的端口，因此TCP连接建立后的握手消息是基于HTTP的，由服务器判断这是一个HTTP协议，还是WebSocket协议。 WebSocket连接除了建立和关闭时的握手，数据传输和HTTP没有任何关系。websocket有以下几个特点：

- websocket 是一种网络传输协议
- websocket 是基于tcp 协议的
- websocket 是全双工的 (客户端和服务端都可以主动通信)
- websocket 是持久性连接

下面看看一次客户端与服务器之间的握手。客户端向服务器发送如下请求：

```http
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: no-cache
Connection: Upgrade
Host: 123.207.136.134:9010
Origin: http://coolaf.com
Pragma: no-cache
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Key: IfMXwR+5XrIJNFoXJzCRCw==
Sec-WebSocket-Version: 13
Upgrade: websocket
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
```

可以看到，上面的握手请求中，

```http
Upgrade: websocket
Connection: Upgrade
```

表明这是WebSocket协议，而非HTTP协议。

然后服务器依据客户端的请求头信息，生成16位安全密钥（Sec-WebSocket-Accept）返回给客户端，就表示WebSocket连接成功。

```http
Connection: Upgrade
Sec-WebSocket-Accept: PxHIdIk2clJ8NPbJkxaVfvQC9wY=
Upgrade: websocket
```

一次握手成功之后，客户端和服务器之间就可以建立持久连接的双向传输数据通道，而且服务器不需要被动地等客户端的请求，服务器这边有新消息就可以通知客户端，化被动为主动。此外，使用WebSocket时，不会像HTTP一样无状态，服务器会一直知道客户端的身份。服务器与客户端之间交换的标头信息也很小。



## 2. WebSocket 服务端

完成一个简单的WebSocket服务端实现，我们需要完成下面几个步骤：

- 协议转换，主要采用Upgrade函数进行协议转换。指定了ReadBufferSize、WriteBufferSize、HandshakeTimeout参数，同时跨域叫为采用默认校验函数，自定义的校验函数总是返回true跳过了跨域校验。
- 连接处理，处理WebSocket客户端连接以及管理
- Handler协调，协调处理WebSocket连接后续操作
- 信息的读写处理

为了更简单的完成 WebSocket服务，引入第三方库 `github.com/gorilla/websocket` 来协助处理。



- **协议转换**

  协议转换部分，主要代码如下：

  ```go
  //定义 upgrade，将协议转换为 websocket并且设定相关参数
  var upgrade = websocket.Upgrader{
  	ReadBufferSize:   1024,
  	WriteBufferSize:  1024,
  	HandshakeTimeout: 10 * time.Second,
  	//跨域处理
  	CheckOrigin: func(r *http.Request) bool {
  		return true
  	},
  }
  ```

  定义了 `upgrade` 变量后，后续就可以使用下面方式来升级协议了。 

  ```go
  ws, err := upgrade.Upgrade(w, r, nil)
  ```

- **连接处理**

  连接处理主要是处理客户端的连接以及关闭的操作，相关变量以及函数主要定义代码如下：

  ```go
  var wsConnAll map[int64]*wsConnection //定义全部连接存储map
  var maxConnId int64                   //定义当前最大连接ID
  
  //定义websocket连接结构体
  type wsConnection struct {
  	wsSocket  *websocket.Conn   //当前连接
  	id        int64             //当前连接ID
  	inChan    chan *wsNessage   //获取信息chan
  	outChan   chan *wsNessage   //发送信息chan
  	mutex     sync.Mutex    
  	isClosed  bool              //连接是否关闭
  	closeChan chan byte         //连接关闭chan
  }
  
  //定义连接关闭函数
  func (wsConn *wsConnection) close() {
  	log.Println("close wsConnection")
  	wsConn.wsSocket.Close()
  	wsConn.mutex.Lock()
  	defer wsConn.mutex.Unlock()
  	if wsConn.isClosed == false {
  		wsConn.isClosed = true
  		delete(wsConnAll, wsConn.id)
  		close(wsConn.closeChan)
  	}
  }
  
  
  ```

  定义完成后，后续可以用下面方式处理连接：

  ```go
  	maxConnId++
  	wsConn := &wsConnection{
  		wsSocket:  ws,
  		inChan:    make(chan *wsNessage, 1000),
  		outChan:   make(chan *wsNessage, 1000),
  		closeChan: make(chan byte),
  		isClosed:  false,
  		id:        maxConnId,
  	}
  	wsConnAll[maxConnId] = wsConn
  ```

  

- 消息读写处理

  读写处理主要功能是将 websocket服务端的信息读取 到相关读管道中，在通过程序将读管道数据写入到写管道中，再通过*Goroutine* 写入发送给客户端。

  基础读写通道函数定义：

  ```go
  //websocket信息结构体
  type wsNessage struct {
  	Type int
  	data []byte
  }
  
  const (
  	writeWait  = 10 * time.Second
  	pongWait   = 60 * time.Second
  	pingPeriod = (pongWait * 9) / 10
  )
  
  //读取通道信息读取函数
  func (wsConn *wsConnection) wsRead() (*wsNessage, error) {
  	select {
  	case msg := <-wsConn.inChan:
  		return msg, nil
  	case <-wsConn.closeChan:
  	}
  	return nil, errors.New("conn is closed")
  }
  
  //写入通道信息写入函数
  func (wsConn *wsConnection) wsWrite(messageType int, data []byte) error {
  	select {
  	case wsConn.outChan <- &wsNessage{messageType, data}:
  	case <-wsConn.closeChan:
  		return errors.New("连接已经关闭")
  	}
  	return nil
  }
  ```

  从 wsSocket读取 信息并写入读取通道函数定义：

  ```go
  func (wsConn *wsConnection) readMessage() {
  	wsConn.wsSocket.SetReadLimit(512)
  	wsConn.wsSocket.SetReadDeadline(time.Now().Add(pongWait))
  	for {
  		msgType, data, err := wsConn.wsSocket.ReadMessage()
  		if err != nil {
  			websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure)
  			log.Println("readMessage error:" + err.Error())
  			wsConn.close()
  			return
  		}
  		req := &wsNessage{
  			Type: msgType,
  			data: data,
  		}
  		select {
  		case wsConn.inChan <- req:
  		case <-wsConn.closeChan:
  			return
  		}
  	}
  }
  ```

  从 读取通道函数获取信息并发送到wsSocket函数定义：

  ```go
  func (wsConn *wsConnection) writeMessage() {
  	ticker := time.NewTicker(pingPeriod)
  	defer func() {
  		ticker.Stop()
  	}()
  	for {
  		select {
  		case msg := <-wsConn.outChan:
  			if err := wsConn.wsSocket.WriteMessage(msg.Type, msg.data); err != nil {
  				log.Println("send message error:", err.Error())
  				wsConn.close()
  				return
  			}
  		case <-wsConn.closeChan:
  			return
  		case <-ticker.C:
  			wsConn.wsSocket.SetWriteDeadline(time.Now().Add(writeWait))
  			if err := wsConn.wsSocket.WriteMessage(websocket.PingMessage, nil); err != nil {
  				return
  			}
  		}
  	}
  }
  ```

  不断处理读与写之间通道信息转换函数定义如下：

  ```go
  func (wsConn *wsConnection) processLoop() {
  	for {
  		msg, err := wsConn.wsRead()
  		if err != nil {
  			log.Println("get message error", err.Error())
  			break
  		}
  		log.Println("get a message", string(msg.data))
  		err = wsConn.wsWrite(msg.Type, msg.data)
  		if err != nil {
  			log.Println("发送消息给客户端出现错误", err.Error())
  			break
  		}
  	}
  }
  ```

  

- Handler 协调处理

  Handler 协调处理主要任务是对连接控制，以及信息的读写操作控制，其代码如下：

  ```go
  func Handler(w http.ResponseWriter, r *http.Request) {
  	ws, err := upgrade.Upgrade(w, r, nil)  //转换协议
  	if err != nil {
  		log.Println("upgrade:", err)
  		return
  	}
      //处理连接
  	maxConnId++
  	wsConn := &wsConnection{
  		wsSocket:  ws,
  		inChan:    make(chan *wsNessage, 1000),
  		outChan:   make(chan *wsNessage, 1000),
  		closeChan: make(chan byte),
  		isClosed:  false,
  		id:        maxConnId,
  	}
  	wsConnAll[maxConnId] = wsConn
  	log.Println("当前在线人数", len(wsConnAll))
  
  	go wsConn.processLoop() //处理读写关系 gorutine
  	go wsConn.readMessage() //从websocket读取信息处理gorutine
  	go wsConn.writeMessage() //将信息写入websocket处理gorutine
  
  }
  ```

完整的代码如下：

```go
package main

import (
	"errors"
	"flag"
	"fmt"
	"github.com/gorilla/websocket"
	"log"
	"net/http"
	"sync"
	"time"
)

var wsConnAll map[int64]*wsConnection //定义全部连接存储map
var maxConnId int64                   //定义当前最大连接ID

const (
	writeWait  = 10 * time.Second
	pongWait   = 60 * time.Second
	pingPeriod = (pongWait * 9) / 10
)

//websocket信息结构体
type wsNessage struct {
	Type int
	data []byte
}

//定义websocket连接结构体
type wsConnection struct {
	wsSocket  *websocket.Conn
	id        int64
	inChan    chan *wsNessage
	outChan   chan *wsNessage
	mutex     sync.Mutex
	isClosed  bool
	closeChan chan byte
}

var upgrade = websocket.Upgrader{
	ReadBufferSize:   1024,
	WriteBufferSize:  1024,
	HandshakeTimeout: 10 * time.Second,
	//跨域处理
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

var addr = flag.String("addr", "127.0.0.1:6666", "ws server address")

func Handler(w http.ResponseWriter, r *http.Request) {
	ws, err := upgrade.Upgrade(w, r, nil)
	if err != nil {
		log.Println("upgrade:", err)
		return
	}
	maxConnId++
	wsConn := &wsConnection{
		wsSocket:  ws,
		inChan:    make(chan *wsNessage, 1000),
		outChan:   make(chan *wsNessage, 1000),
		closeChan: make(chan byte),
		isClosed:  false,
		id:        maxConnId,
	}
	wsConnAll[maxConnId] = wsConn
	log.Println("当前在线人数", len(wsConnAll))

	go wsConn.processLoop()
	go wsConn.readMessage()
	go wsConn.writeMessage()

}

func (wsConn *wsConnection) processLoop() {
	for {
		msg, err := wsConn.wsRead()
		if err != nil {
			log.Println("get message error", err.Error())
			break
		}
		log.Println("get a message", string(msg.data))
		err = wsConn.wsWrite(msg.Type, msg.data)
		if err != nil {
			log.Println("发送消息给客户端出现错误", err.Error())
			break
		}
	}
}

func (wsConn *wsConnection) writeMessage() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
	}()
	for {
		select {
		case msg := <-wsConn.outChan:
			if err := wsConn.wsSocket.WriteMessage(msg.Type, msg.data); err != nil {
				log.Println("send message error:", err.Error())
				wsConn.close()
				return
			}
		case <-wsConn.closeChan:
			return
		case <-ticker.C:
			wsConn.wsSocket.SetWriteDeadline(time.Now().Add(writeWait))
			if err := wsConn.wsSocket.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}

func (wsConn *wsConnection) readMessage() {
	wsConn.wsSocket.SetReadLimit(512)
	wsConn.wsSocket.SetReadDeadline(time.Now().Add(pongWait))
	for {
		msgType, data, err := wsConn.wsSocket.ReadMessage()
		if err != nil {
			websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure)
			log.Println("readMessage error:" + err.Error())
			wsConn.close()
			return
		}
		req := &wsNessage{
			Type: msgType,
			data: data,
		}
		select {
		case wsConn.inChan <- req:
		case <-wsConn.closeChan:
			return
		}
	}
}

func (wsConn *wsConnection) wsRead() (*wsNessage, error) {
	select {
	case msg := <-wsConn.inChan:
		return msg, nil
	case <-wsConn.closeChan:
	}
	return nil, errors.New("conn is closed")
}

func (wsConn *wsConnection) wsWrite(messageType int, data []byte) error {
	select {
	case wsConn.outChan <- &wsNessage{messageType, data}:
	case <-wsConn.closeChan:
		return errors.New("连接已经关闭")
	}
	return nil
}

func (wsConn *wsConnection) close() {
	log.Println("close wsConnection")
	wsConn.wsSocket.Close()
	wsConn.mutex.Lock()
	defer wsConn.mutex.Unlock()
	if wsConn.isClosed == false {
		wsConn.isClosed = true
		delete(wsConnAll, wsConn.id)
		close(wsConn.closeChan)
	}
}

func main() {
	flag.Parse()
	wsConnAll = make(map[int64]*wsConnection)
	http.HandleFunc("/send", Handler)
	fmt.Println(*addr)
	http.ListenAndServe(*addr, nil)
}
```



## 3. WebSocket 客户端

客户端代码这边举例2种方式，一种是通过JS现实，一种是通过go语言实现，代码我就直接贴下面了，不做过多解析了。

Go客户端：

```go
package main

import (
	"fmt"
	"golang.org/x/net/websocket"
	"log"
)

func main() {
	origin := "http://127.0.0.1:8888/"
	url := "ws://127.0.0.1:8888/send"
	ws, err := websocket.Dial(url, "", origin)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(ws)
	_, err = ws.Write([]byte("zzzzzzzzzz"))
	fmt.Println(err)

	for {
		var msg = make([]byte, 1000)
		m, err := ws.Read(msg)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("Receive: %s\n", msg[:m])
	}
}
```

HTML+JS 客户端：

```js
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <script>
        window.addEventListener("load", function(evt) {
            var output = document.getElementById("output");
            var input = document.getElementById("input");
            var ws;
            var print = function(message) {
                var d = document.createElement("div");
                d.innerHTML = message;
                output.appendChild(d);
            };
            document.getElementById("open").onclick = function(evt) {
                if (ws) {
                    return false;
                }
                ws = new WebSocket("ws://127.0.0.1:8888/send");
                ws.onopen = function(evt) {
                    print("OPEN");
                }
                ws.onclose = function(evt) {
                    print("CLOSE");
                    ws = null;
                }
                ws.onmessage = function(evt) {
                    print("RESPONSE: " + evt.data);
                }
                ws.onerror = function(evt) {
                    print("ERROR: " + evt.data);
                }
                return false;
            };
            document.getElementById("send").onclick = function(evt) {
                if (!ws) {
                    return false;
                }
                print("SEND: " + input.value);
                ws.send(input.value);
                return false;
            };
            document.getElementById("close").onclick = function(evt) {
                if (!ws) {
                    return false;
                }
                ws.close();
                return false;
            };
        });
    </script>
</head>
<body>
<table>
    <tr><td valign="top" width="50%">
        <p>Click "Open" to create a connection to the server,
            "Send" to send a message to the server and "Close" to close the connection.
            You can change the message and send multiple times.
        </p>
            <form>
                <button id="open">Open</button>
                <button id="close">Close</button>
            <input id="input" type="text" value="Hello world!">
            <button id="send">Send</button>
            </form>
    </td><td valign="top" width="50%">
        <div id="output"></div>
    </td></tr></table>
</body>
</html>
```

