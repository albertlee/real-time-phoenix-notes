# Real-Time Phoenix 2 - Connect a Simple WebSocket

## by: 韩祝鹏

客户端设备可能是手机、PC或各种IoT设备。
WebSocket是当前实时web应用的基础，因此理解WebSocket的原理很重要。
（在负载均衡后面，WebSocket连接可能有问题，内存的使用，等，都需要理解WebSocket原理）

## Why WebSockets

以前的实时通讯技术，有浏览器兼容性问题。开发成本高。

2011年HTML5引入了WebSocket协议，用来解决实时web通讯问题。现在主流的浏览器已经都原生支持了。

- WebSocket允许在一个TCP连接里做双向的数据通讯。降低带宽消耗和连接的建立消耗。
- Elixir里用 cowboy web server 对WebSocket支持的很好，对应到Erlang的Process model。
- WebSocket起源于 HTTP request，因此各种技术如负载均衡，Proxy都可以使用。
  
应用：Facebook的聊天，Yahoo财经的实时股票数据，多人在线网络游戏。

## Connecting our First WebSocket

创建实例项目

`$ mix phx.new hello_sockets --no-ecto`

由于墙的存在，deps get：

`HEX_MIRROR=http://hexpm.upyun.com HEX_HTTP_CONCURRENCY=1 mix deps.get`

修改 assets/js/app.js  ：

`import socket from "./socket"`

启动：

`mix phx.server`

## WebSocket Protocol
学习如何建立连接，保持连接alive，发送和接受数据，保持连接的安全性。
需要深度Debug的时候，读RFC。
学习调试，用Chrome的DevTools。

### 建立连接
打开 Chrome DevTools->Network -> WS ，访问实例项目的页面。
(两个WebSocket连接，其中一个是开发环境用来reload代码的。)

Request Headers，Response Headers, HTTP method(GET)

WebSocket从一个普通的Web Request开始，然后"upgraded" 到一个 WebSocket。

发出Request后，收到 101 response status code， “101 Switching Protocols”。从HTTP变成WebSocket。这样设计可以兼容以前的服务器代理。

WebSocket可以用于浏览器，也可以用于非浏览器环境，如服务器间或移动客户端。

过程：

1. 发出 GET HTTP连接请求，到 WebSocket endpoint
2. 收到服务端返回的 101 code
3. 升级协议到 WebSocket
4. 通过WebSocket连接进行双向通讯

### 发送接收数据

DevTools 里 Message Tab下，可以看到上行，下行的消息。全双工连接（full-duplex connection)。

扩展协议。

### Staying Alive, Keep-Alive

WebSocket协议规定了 Ping-Pone frame，确定连接依然活着，这是可选的，Phoenix没有用它。
客户端每30秒发送心跳信息给Phoenix服务器。Phoenix服务器如果一段时间收不到Ping消息就会关闭连接，默认60秒。

`发送：[null,"57","phoenix","heartbeat",{}]`

`接受：[null,"57","phoenix","phx_reply",{"response":{},"status":"ok"}]`

客户端负责检查连接，等发现连接断了，可以立刻重连。

### 安全性

- 生产环境需要使用 wss:// https 协议来保证安全性。
- Request的 Header里，检查Origin，确定来源。
- WebSocket 不适用 CORS保护 cross-origin resource sharing.
- 注意跨域问题。CSRF攻击

## Long Polling， 另一种实时的选择

在特定情况下，可以使用不同的通讯层，WebSocket并不是唯一的。
WebSocket和Long Polling可以同时使用。

### Long polling的request流程：

1. 客户端创建 HTTP request
2. 服务端不 response 这个 request，而是留着它。等有了新数据或过一段时间，再response。
3. 服务器发送完整的response到客户端，此时客户端收到了服务器的实时数据。
4. 客户端 loops

### 是否使用 Long Polling

优点：
    Long Polling完全基于 HTTP。

不足：
    [总结Long Polling的问题与最佳实践](https://tools.ietf.org/html/rfc6202)
    - 资源的浪费，每次请求都要处理 Request Headers
    - 网络不好时消息的延迟严重。

相对来说，WebSocket长连接要好的多，不需要每次都重发Header，重新连接。

Long Polling在负载均衡时可能有好处，因为经常重新建立连接。而WebSocket总是连在一台服务器上，连接的越长，越不容易更换。

## WebSocket 与 Phoenix Channels

Phoenix对不活跃的WebSocket连接进行休眠处理，占用很少内存。
