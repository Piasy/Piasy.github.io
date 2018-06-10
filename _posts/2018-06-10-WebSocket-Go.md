---
layout: post
title: WebSocket Go
tags:
    - 网络
---

最近工作需要，对 WebSocket 进行了一点研究，今天在这里给大家分享一下我对 WebSocket 协议、WebSocket 的 Go 语言实现，以及 socket.io 服务端库的 Go 语言实现的理解。

## WebSocket 简介

WebSocket 旨在实现 Web 应用（例如 IM、游戏等）里和 Server 的双向通信（两端称为 peer），以替代 HTTP 轮询等方案。

WebSocket 协议包括两部分：握手，数据传输。它基于 TCP 的字节流传输机制，提供了 frame 的传输机制。

WebSocket 是基于 TCP 的协议，它和 HTTP 的关系仅仅是其握手可以被 HTTP 服务器理解为 Upgrade 请求。

### 建立连接的握手

握手的请求与响应和 HTTP 1.1 格式相同，这是为了让 HTTP 协议的服务器程序和 WebSocket 协议的服务器程序可以挂在同一个 Web 服务器后面。

客户端请求：

~~~ bash
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
~~~

服务端响应：

~~~ bash
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
~~~

客户端握手请求实际上是一个 HTTP Upgrade 请求。`Upgrade` 和 `Connection` 字段表明这是 WebSocket Upgrade 请求和响应。

`Sec-WebSocket-Key` 是客户端随机提供的字符串的 base64 编码结果，服务端收到后，将（编码结果）值与一个特殊的 GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` 拼接，然后求拼接结果字符串的 SHA-1，最后将 SHA-1 二进制结果做 base64 编码，作为 `Sec-WebSocket-Accept` 字段返回。客户端会对其进行校验。

101 状态码表明服务器接受了 Upgrade，其他状态码都表明 Upgrade 失败。

客户端提供的 `Sec-WebSocket-Protocol` 字段表明了客户端支持的一系列子协议，服务端可以选择其中一个返回。

### 关闭连接的握手

任一端都可以发送一个「开始关闭」的控制帧，另一端收到后，如果未发送过「关闭」帧则发送之，发起端收到后，就可以关闭连接了。

peer 发送了「开始关闭」帧后，就不再发送数据，peer 收到「开始关闭」帧后，会丢弃之后收到的数据。

这个关闭连接的握手，是对 TCP 四次握手的补充。

### 数据传输

握手成功后，数据被封装为 message 在连接内传输。message 由 frame 构成，这两个概念和底层协议的封包没有关系。

frame 结构：

![](https://imgs.piasy.com/2018-06-06-websocket_frame.png)

我们可以看到，WebSocket 的 frame 采取的是 header + payload 的结构，header 里有 length 字段以分隔 frame。

客户端发送的数据必须应用掩码，服务端发送的数据一定不能应用掩码。

## WebSocket Go 源码导读

接下来我们分析一下 [gorilla/websocket](https://github.com/gorilla/websocket) 这个 Go 语言的 WebSocket 协议实现的源码。

### 建立连接

客户端建立连接的接口为 `client.go Dialer.Dial`：

~~~ go
c, _, err := websocket.DefaultDialer.Dial("ws://localhost:8080/echo", nil)
~~~

+ 准备 `http.Request` 对象，用来发起握手请求；
+ 默认使用 `net.Dialer Dialer.Dial` 函数建立 TCP 连接；
+ 连接建立成功后，取得了 `net.Conn` 对象，于是调用 `conn.go newConn` 创建 WebSocket 的核心类 `websocket.Conn` 对象；
+ 创建完对象之后，把握手请求数据写入之，并读取响应，检查握手是否成功；
+ 如果一切顺利，就把 `websocket.Conn` 对象返回，之后 ws client 就能 `ReadMessage` 和 `WriteMessage` 了；

服务端接受客户端连接的接口为 `server.go Upgrader.Upgrade`：

![](https://imgs.piasy.com/2018-06-07-websocket_listen.png)

+ 通常这个接口都由 HTTP handler 调用，传入 `w http.ResponseWriter`, `r *http.Request` 和 `responseHeader http.Header`；
+ 检查请求的 header，确保是握手请求；
+ 把 `w` 强转为 `http.Hijacker`，进而调用 `Hijacker.Hijack` 得到 `net.Conn` 对象，这是 Go http 模块的一套机制，使得 HTTP handler 可以接管网络连接，之后 http 模块不会对连接做任何操作，这正是 WebSocket 需要的；
+ 拿到了 `net.Conn` 之后，调用 `conn.go newConnBRW` 创建 WebSocket 的核心类 `websocket.Conn` 对象，并把握手响应返回给客户端；
+ 如果一切顺利，就把 `websocket.Conn` 对象返回，之后 ws server 就能 `ReadMessage` 和 `WriteMessage` 了；

值得一提的是，WebSocket 库并未提供监听客户端请求的功能，不过这件事利用 Go 的 http 模块即可完成，拿到 `http.ResponseWriter` 和 `http.Request` 即可。

最后，`conn.go newConn` 内部是调用 `conn.go newConnBRW` 实现的，只是 `isServer` 字段取值不同。

### 读数据

读数据的接口是 `conn.go Conn.ReadMessage`：

~~~ go
mt, message, err := c.ReadMessage()
~~~

+ `ReadMessage` 是一个辅助接口，其内部调用 `NextReader`，并从中读出一个 frame；
+ `NextReader` 内部则是循环调用 `advanceFrame` 取得下一个 frame（可能阻塞），如果取到了，就构造一个 `messageReader` 对象并返回，如果支持压缩，就包一层解压；
+ `advanceFrame` 会读取 frame header，如果是 text 或 binary frame，就返回 frame type，等待之后的 `Read` 调用消费 payload；如果是 control frame，就读取 payload 并处理；frame header 里有长度字段，所以接收端知道应该读取多少数据；
+ 无论是 `advanceFrame` 里的读操作，还是 `NextReader` 返回之后的读操作，最终都是调用的 `conn_read.go Conn.read` 读取数据，而其中都是从 `Conn` 的 `bufio.Reader` 成员读取数据；
+ `Conn` 的 `bufio.Reader` 成员的赋值，在 server 端是由 `Hijacker.Hijack` 返回（创建自 `net.Conn` 对象）并传入 `newConnBRW` 函数；在 client 端是在 `newConnBRW` 里创建自 `net.Conn` 对象；总之，就是从 `net.Conn` 对象读取数据；

### 写数据

写数据的接口是 `conn.go Conn.WriteMessage`：

~~~ go
err := c.WriteMessage(websocket.TextMessage, []byte("hello"))
~~~

+ `WriteMessage` 也是一个辅助接口，其内部调用 `NextWriter`，并把数据写入；
+ 不过对于 server 来说，有一个优化：如果不需要压缩，那就直接创建 `messageWriter` 对象，并把数据写入；
+ `NextWriter` 内部其实也就是创建一个 `messageWriter` 对象并返回，如果支持压缩，就包一层压缩；
+ 数据写入主要分为两步：把数据拷贝到 `writeBuf` 中；调用 `messageWriter.flushFrame` 把 `writeBuf` 里的数据写到网络；发送端会把数据长度写入 frame header，以便接收端读取；
+ `flushFrame` 实际调用 `Conn.write`，最终把数据写到 `net.Conn` 对象里；

---

socket.io 是一个更上层的长连接开源库，它在客户端和服务端都提供了异步事件接口，使用起来更加简单。它同时支持 WebSocket 模式和 HTTP 轮询模式，这两套传输层封装在 engine.io 里。所以 socket.io 使用 engine.io，后者又使用了 WebSocket 和 HTTP 轮询。

接下来我们就分析一下 engine.io 和 socket.io 的源码。

## engine.io Go 源码导读

engine.io 的使用主要分为三步：

+ 创建 server，添加到 HTTP handler 中，开始监听请求；
+ 新起 goroutine，接受连接；
+ 新起 goroutine，使用连接读写数据；

### 监听请求

engine.io 监听请求的方式和 WebSocket 类似：

![](https://imgs.piasy.com/2018-06-07-engineio_listen.png)

收到 HTTP 请求后，Go http 模块会调用 `server.go Server.ServeHTTP` 函数，处理请求。

+ `ServeHTTP` 首先会获取客户端请求里的 `sid` 参数，用来标识客户端，对于 WebSocket 来说，它的作用不大，但在 HTTP 轮询时，如何把多次请求对应到同一个客户端？靠的就是这个 sid；
+ 首次请求时，客户端不会带着 sid，sid 是由 server 分配的，此外 server 还会检查当前连接数量，目前最多允许 1000 并发连接；
+ 如果请求可以处理，就创建 `server_conn.go serverConn` 对象，并存入 sid -> serverConn 的 map 里，以便之后客户端的请求可以被同一个 serverConn 对象处理；这个场景只在 HTTP 轮询模式下存在，由 `serverConn.ServeHTTP` 函数处理之后的请求，这些请求最终是在 `polling/server.go Polling.ServeHTTP` 函数中处理；
+ 初次请求的情况，创建完 Conn 对象之后，会把它写到 `socketChan` 里；

### 接受连接

接受连接的接口是 `server.go Server.Accept`：

~~~ go
conn, _ := server.Accept()
~~~

它其实就是从 `socketChan` 里读数据，还没有请求时这个调用会阻塞，客户端初次请求时就会读到创建的 Conn 对象。

### 读写数据

读写数据的接口是 `serverConn` 的 `NextReader` 和 `NextWriter`，它们返回的 reader 和 writer 其实是对 WebSocket 库的 `NextReader` 和 `NextWriter` 返回结果的包装。

但读数据中间还有一条 channel：

+ `serverConn.NextReader` 是从 `readerChan` 里读出 reader；
+ 往 `readerChan` 里写入 reader 是在 `serverConn.OnPacket` 接口里；
+ `OnPacket` 则分别由 `engineio.websocket.Server.serveHTTP` 函数和 `engineio.polling.Polling.post` 调用，即 WebSocket 和 Polling 读到数据之后，通知收到数据；

写数据则直接一些，`serverConn.NextWriter` 调用 `engineio.transport.Server.NextWriter` 取得 writer，最终则是调用 `websocket.Conn.NextWriter` 或者 Polling 的 writer 对象。

## socket.io Go 源码导读

socket.io 的使用主要分为五步：

+ 创建 server，添加到 HTTP handler 中，开始监听请求；
+ 监听 server 的事件，例如 connection, disconnection, error 等；
+ connection 事件会传入 `socketio.Socket` 对象，进而我们可以让 socket 加入房间；
+ 加入房间后，我们就可以监听 socket 的事件了，例如 disconnection，以及自定义事件名称；
+ 我们还可以用 `socket.Emit` 发送消息（发往这个 socket 对应的客户端），也可以用 `socket.BroadcastTo` 在房间里广播；

### 监听请求

socket.io 监听请求的方式则和 engine.io 完全一样：

![](https://imgs.piasy.com/2018-06-07-socketio_listen.png)

+ `NewServer` 里会调用 `engineio.NewServer` 创建 `engineio.Server` 对象；
+ 创建了 `socketio.Server` 对象后，会起一个协程，执行 `Server.loop` 函数；
+ 在 `loop` 函数里，会不停调用 engine.io 的 `Server.Accept` 函数；拿到了 `engineio.Conn` 对象后，会创建一个 socket 对象，并起一个协程调用 `socket.loop` 函数；
+ `socket.loop` 函数包含了 socket.io 异步事件的核心实现逻辑，在其中会不停调用 `parser.go newDecoder` 接口，从中读取数据，读到之后通过 `socketHandler.onPacket` 回调出来；

HTTP 请求的处理，则是由 Go http 模块传递到 `socketio.Server`，进而传递到 `engineio.Server`。

### 监听事件

监听事件包括两部分，server 事件和 socket 事件：

+ `socket.loop` 开始的时候，会回调一次 `socketHandler.onPacket`，事件类型为 connection；
+ socketHandler 会把注册进去的事件名和事件处理函数存入一个 map 中，每个事件只能注册一个处理函数；
+ 读取到数据之后，会通过 `socketHandler.onPacket` 发出事件，事件处理函数的调用，利用了 Go 的反射技术；
+ server 事件和 socket 事件是存在一起的；

### 收发数据

数据读取通过 `decoder` 实现：

+ 构造 decoder 时传入的 `frameReader` 实际上是 `engineio.Conn` 对象；
+ 在 `decoder.Decode` 函数里，会调用 `Conn.NextReader`，进而读取数据；

数据发送的接口包括 `Emit` 和 `BroadcastTo`，`BroadcastTo` 最终也是遍历房间里的所有 socket，然后调用其 `Emit` 接口。

`Emit` 最终发送数据则是通过 `encoder` 实现：

+ 构造 encoder 时传入的 `frameWriter` 也是 `engineio.Conn` 对象；
+ 在 `encoder.Encode` 函数里，会调用 `Conn.NextWriter`，进而发送数据；

在 encoder 和 decoder 里，还实现了一套 socket.io 的协议，包括 ACK、编码等逻辑，这里就不做展开了。

## 总结

好了，对 WebSocket 相关内容的分享就到这里，感谢阅读和支持 :)

## 参考文章

+ [The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
+ [package websocket](https://godoc.org/github.com/gorilla/websocket)
