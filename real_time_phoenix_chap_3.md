# Real-Time Phoenix 3 - First Steps with Phoenix Channels

## by: 韩祝鹏

Phoenix Channels 是我们的实时应用的核心。



## What are Phoenix Channels

channels工作在一个高层抽象，让客户端连接的web server，并订阅各种topic。客户端发送和接收订阅的topics上的消息。一个连接可以订阅多个topic，不需要创建昂贵的多个连接。

> *注意* 每个Channel 都是一个 GenServer，每个client join进一个Channel，就会创建这个连接相应的 Channel 进程。客户端连接中断后，相应的channel进程就结束。一个Channel进程是对应到一个Client连接上的。

从Client的视角：

1. subscribe topics
2. send/receive data (on the topics)

从 Server的视角：

- 用OTP的process设计，容错性。
- Transport agnostic，应用使用了 WebSocket，但并不是被它限定的。理论上可以方便的替换通讯层的实现，而不需要改动业务逻辑。（设计的分隔，类似于Spring依赖注入的思想）
  
## 理解 Channel的结构

![Channel结构](images/chap_3_channel_structure.png)

- client通过一个连接机制（如 WebSocket）直接连接到一个管理连接的OTP Process。
- Transport Process将特定操作代理到应用代码里，我们的应用实现 Phoenix.Socket behaviour。（实现各个回调函数，处理特定事件） 在 HelloSocketsWeb.UserSocket 模块里。
- 每一个不同的Topic都有分开的进程处理。
- Phoenix.Socket 模块，讲客户端的请求路由到实现 Phoenix.Channel 的模块。
- Phoenix.PubSub 用于Channel间路由消息。允许集群消息广播。

## Sockets
职责：连接的处理，路由。确定连接的安全验证，用户id标识，定义及route topic。

一般使用 Phoenix.Socket， 如果特殊情况需要定制实现不同的传输方式，可以实现自己的 Phoenix.Socket.Transport。

要实现的回调函数：
- connect/3  （可以用来实现安全检查）
- id/1   （确定不同连接client的id）

`hello_sockets/lib/hello_sockets_web/channels/user_socket.ex`



```elixir
use Phoenix.Socket
## Channels
channel "ping", HelloSocketsWeb.PingChannel
```

channel 宏定义一个topic，并route到一个指定的 Channel 实现module。

## Channels
放置业务逻辑，应用的请求处理代码。职责：

- 接受或拒绝 join 请求
- 处理client发的消息
- 处理PubSub来的消息
- 向client推送消息

Channel类似 MVC里的Controller。Socket模块类似 Router。

*skinny controllers* 设计，尽量让应用逻辑放在应用的内核里，而不是在 Channel里实现。与实时连接相关的逻辑，放在这里。

Channel 就是在GenServer上做的一层包装，因此可以处理GenServer的各个回调函数。

callback函数：

- join/3
- handle_in/3
- handle_out/3
- terminate/2
- handle_info/2
- handle_call/3
- handle_cast/2
- code_change/3

示例代码：

```elixir
defmodule HelloSocketsWeb.PingChannel do
  use Phoenix.Channel

  def join(_topic, _payload, socket) do
    {:ok, socket}
  end

  def handle_in("ping", _payload, socket) do
    {:reply, {:ok, %{ping: "pong"}}, socket}
  end
end
```

join/3 用于加入认证。

handle_in 处理事件，参数：event,payload, 当前socket的状态。收到消息后，可以：

- 回复信息： {:reply, {:ok, map()}, Phoenix.Socket}
- 不回复: {:noreply, Phoenix.Socket}
- 断开Channel， {:stop, reason, Phoenix.Socket}

用 wscat 可以在命令行实验

```shell
sudo npm install -g wscat
wscat -c 'ws://localhost:5000/socket/websocket?vsn=2.0.0'
Connected (press CTRL+C to quit)
> ["1","1","ping","phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
> ["1","2","ping","ping",{}]
< ["1","2","ping","phx_reply",{"response":{"ping":"pong"},"status":"ok"}]
>  ["1","2","ping","ping2",{}]
< ["1","1","ping","phx_error",{}]
## 这时服务端上，PingChannel进程终止，并被重启
>  ["1","2","ping","ping",{}]
< [null,"2","ping","phx_reply",{"response":{"reason":"unmatched topic"},"status":"error"}]
> ["1","1","ping","phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
>  ["1","2","ping","ping",{}]
< ["1","2","ping","phx_reply",{"response":{"ping":"pong"},"status":"ok"}]
```

#### Fault Tolerance

与传统Web controller不同的是，Channel是常驻的。现实中会遇到bug以及网络断开等，进程会crash。

在上面的例子里client发出未定义的消息，channel进程会crash，然后重启，这是client需要重新join。官方的JavaScript客户端已经处理了这种情况。

channel重启了，但是客户的connection并没有断，客户如果join了其他的channel，其他channel也不会受到影响。

#### 其他

Channel可以自动休眠，减少内存占用。设置Channel的休眠时间（默认15秒）

```elixir
use Phoenix.Channel, hibernate_after: 60_000
```

### Topics

topic 是一个string，习惯上用 "topic:subtopic" 格式。因为 channel/3 可以使用通配符。

```elixir
channel "ping", HelloSocketsWeb.PingChannel
channel "ping:*", HelloSocketsWeb.PingChannel
channel "wild:*", HelloSocketsWeb.WildcardChannel
```

在Channel的实现模块里，通过join/3 回调可以自定义topic名称通配符的判断规则：

```elixir
defmodule HelloSocketsWeb.WildcardChannel do
  use Phoenix.Channel
  def join("wild:" <> numbers, _payload, socket) do
    if numbers_correct?(numbers) do
      {:ok, socket}
    else
      {:error, %{}} end
    end

  def handle_in("ping", _payload, socket) do
    {:reply, {:ok, %{ping: "pong"}}, socket}
  end

  defp numbers_correct?(numbers) do
    numbers
    |> String.split(":")
    |> Enum.map(&String.to_integer/1)
    |> case do
      [a, b] when b == a * 2 -> true
      _ -> false
    end
  end
end
```

上面的代码里，如果 client join发过来的是 wild:1:2 就join 成功，如果不是 wild:[Int]:[Int]这样的格式 Channel进程就会崩溃重启，但至少阻止了不合法的连接。

> 读源代码， lib/phoenix/socket.ex ，用的macro，在编译时将名称字符串生成了模式匹配的源代码。这块实现类似Common Lisp了。


### Topic 名称的选择

动态的Topic名称，可以实现特定用户，或特定群组的私有消息，如 "notifications:t-1:u-2" 这样格式，可以指明 Team 1 下的 User 2.系统可以在任意位置，向指定的用户发送消息，只要提供相应的用户id。

选择Topic名称，对扩展性很重要。例如一个提供库存更新的Channel，可以用不同的方式实现：

- "inventory" 不同SKU之间没有区分。一个商品的变化，将广播到所有连接的客户端，即使这个客户端并不关心这个商品。代码简单，但会用掉更多带宽，并且暴露全部的数据。
- "inventory:*" 用通配符区分不同的SKU。需要连接多个topic，每个商品一个，但对不关注的商品不会发送数据。

## PubSub

Phoenix.PubSub (publisher/subscriber)，实现topic订阅与消息广播。Channel在内部使用 PubSub，一般不用跟它打交道，但要理解它，才能更好的配置及确保性能及扩展性。

PubSub 在本地node 和所有连接的远程 node之间都连接。可以让PubSub在整个集群里广播消息。多个PubSub之间的通讯，PubSub内置使用pg2 adapter。另外还有一个基于 Redis的adapter，不需要node之间连接在一起。（见Chap11 部署应用）

看 pubsub的源代码，broadcast的实现，也是通过注入的 adapter 实现具体的功能。依赖注入。

broadcast例子：

```shell
iex(6)> HelloSocketsWeb.Endpoint.broadcast("ping","test", %{data: "test"})
:ok
```

```shell
$ wscat -c 'ws://localhost:5000/socket/websocket?vsn=2.0.0'
Connected (press CTRL+C to quit)
> ["1","1","ping","phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
< [null,null,"ping","test",{"data":"test"}]
```

## 发送与接收消息

### Phoenix Message Structure

Message的内容可以让客户端追踪请求和回复的流，因为单个Channel会有多个异步的请求。

```elixir
defmodule Phoenix.Socket.Message do
    @type t :: %Phoenix.Socket.Message{}
    defstruct topic: nil, event: nil, payload: nil, ref: nil, join_ref: nil
```

- `:topic` - The string topic or topic:subtopic pair namespace, for example "messages", "messages:123"
- `:event`- The string event name, for example "phx_join"，Channel的实现可以用模式匹配来处理不同的event。
- `:payload` - The message payload， JSON map
- `:ref` - The unique string ref，发送消息时递增数字
- `:join_ref` - The unique string ref when joining，join到topic时递增的数字

官方的 Channel客户端库，在发送请求时，给出 join_ref, ref, topic，服务端返回消息时，带有相同的 join_ref,ref,topic，这样客户端就可以方便的组织消息的处理。

### 从客户端接收消息

客户端的消息通过Socket，路由到相应的 Channel，通过 handle_in/3 回调处理。这样可以用一个Socket连接多个Channel，并依然保持高性能。

见 socket.ex 下， handle_in 方法的实现。

#### 用模式匹配实现回调函数

可以对event名字，和 payload 内容的匹配：

```elixir
  def handle_in("ping", %{"ack_phrase" => ack_phrase}, socket) do
    {:reply, {:ok, %{ping: ack_phrase}}, socket}
  end
  def handle_in("ping", _payload, socket) do
    {:reply, {:ok, %{ping: "pong"}}, socket}
  end
  def handle_in("ping_" <> phrase, _payload, socket) do
    {:reply, {:ok, %{ping: phrase}}, socket}
  end
```

```shell
$ wscat -c 'ws://localhost:5000/socket/websocket?vsn=2.0.0'
Connected (press CTRL+C to quit)
> ["1","1","ping","phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
> ["1","2","ping","ping",{}]
< ["1","2","ping","phx_reply",{"response":{"ping":"pong"},"status":"ok"}]
> ["1","3","ping","ping_hello",{}]
< ["1","3","ping","phx_reply",{"response":{"ping":"hello"},"status":"ok"}]
> ["1","4","ping","ping",{"phrase":"good"}]
< ["1","4","ping","phx_reply",{"response":{"ping":"pong"},"status":"ok"}]
> ["1","5","ping","ping",{"ack_phrase":"good"}]
< ["1","5","ping","phx_reply",{"response":{"ping":"good"},"status":"ok"}]
```

Tips：

- Atom 不会被BEAM虚拟机gc，系统要注意不要耗尽Atom，所以Phoenix中用户提交的数据不能是Atom。
- payload可以用复杂的结构和数据类型，比event name灵活

#### Other Response types

```elixir
def handle_in("pong", _payload, socket) do # We only handle ping
  {:noreply, socket}
end
def handle_in("ding", _payload, socket) do
  {:stop, :shutdown, {:ok, %{msg: "shutting down"}}, socket}
end
```

- noreply 不做回复
- stop

### Push Messages to a Client

不需要在Channel里写 handler代码，把消息发送给topic，就可以给连接到topic的client发送消息。我们可以通过intercept发出的消息，来定制这个行为。

```elixir
defmodule HelloSocketsWeb.PingChannel do
  use Phoenix.Channel
  intercept ["request_ping"]
  # ...
  def handle_out("request_ping", payload, socket) do
    push(socket, "sned_ping", Map.put(payload, "from_node", Node.self()))
    {:noreply, socket}
  end
end
```

运行：

```elixir
iex(6)> HelloSocketsWeb.Endpoint.broadcast("ping", "request_ping", %{})
:ok
```

```shell
$ wscat -c 'ws://localhost:5000/socket/websocket?vsn=2.0.0'
Connected (press CTRL+C to quit)
> ["1","1","ping","phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
< [null,null,"ping","sned_ping",{"from_node":"nonode@nohost"}]
```

Tips：

- Best practice: 不需要定制化payload，就不要intercept。
- Intercepting Events for Metrics

## Channel Clients

### 官方 JavaScript 客户端

客户端的职责

- 连接到server，发送心跳保持连接
- join topics
- 向 topic中push 消息，处理response
- 接受 topic 发来的消息
- 处理断开及其他错误，尽量保持连接

#### 发送消息

在socket.js 中，创建socket，channel，join。

```javascript
import {Socket} from "phoenix"

let socket = new Socket("/socket", {params: {token: window.userToken}})
socket.connect()

let channel = socket.channel("ping")
channel.join()
  .receive("ok", resp => { console.log("Joined ping successfully", resp) })
  .receive("error", resp => { console.log("Unable to join ping", resp) })

console.log("send ping")
channel.push("ping")
  .receive("ok", (resp) => console.log("receive:", resp.ping))
export default socket
```

- channel.join
- channel.push  发送消息，添加 receive处理回调，处理 ok,error, timeout 事件
- channel.on 注册接受处理特定消息的回调函数

客户端在join一个topic之前，发送的消息先缓存在客户端内存里，一旦连接上就发送。缓存是一个5秒的短生命期缓存。应对连接不稳定的问题。

#### 接收消息

服务端Channel可以随时给连接的客户端发送消息，不只是回复进入的消息。客户端通过 channel.on 注册特定event 的处理函数，这需要 event name 已知，可以将动态的内容放在 payload里。

```js
channel.on("send_ping", (payload) => {
  console.log("ping requested", payload)
})
```

```elixir
$ iex -S mix phx.server
iex(1)> HelloSocketsWeb.Endpoint.broadcast("ping", "request_ping", %{})
:ok
```

### JavaScript客户端的容错和错误处理

- Phoenix的JS客户端做了自动重连的机制。与服务器断开后会尝试重连。
- 服务端的Channel 如果遇到错误 Crash了，发送 phx_error 消息给客户端。Channel进程会重启，客户端会重连。
