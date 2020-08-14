# Real-Time Phoeniex 4. Restrict Socket and Channel Access

## By: 韩祝鹏 2020-08

- 权限控制，限制Socket 和 Channel的访问。
- Phoenix.Token 传递认证信息
- 何时使用单独或多个Socket

## 为什么要限制访问

防止非认证的用户获取网站的信息，防止用户访问到其他用户的私有信息。

防止非认证的用户访问应用，对Socket增加认证限制。
限制访问用户的数据，对Channel和topic增加认证限制。

在本书第二部分，商城应用的例子里，后台管理界面增加Socket的认证，对用户的购物车topic "cart:{userid}" 增加join的权限认证。


Channel 在 Channel.join/3 函数里加。

## 为 Socket 增加认证

要阻止非授权的用户访问应用，直接在Socket的连接函数里拒掉。验证逻辑在一个单点上，避免在其他代码里再验证用户是否登录。
（单一职责原则，DRY: Don't Repeat Yourself原则）

Socket的访问限制在 Socket.connect/3 函数里加，客户端连接时调用此回调，返回：

- {:ok, socket} 允许连接
- {:error, ..} 拒绝连接

connect 函数可以给一个连接生命期里存储数据。 Socket.assigns 管理 Socket 的state。认证通过时，将用户的id存储在Socket的state里，供后续使用。

### 用签名Token 保护 Socket 安全

WebSocket没有CORS(Cross-origin resource sharing)限制，因此容易被CSRF跨站攻击。
防止这种攻击，可以用检查 origin 来源是否是已知的domain，或者使用CSRF token。

本书里使用的认证策略，不是使用cookie，而是用签名的token。
客户端连接时在参数里提供签名的token，服务端验证这个token是否是自己服务器里一定时间内生成的。

Endpoint 里增加一个Socket：

```elixir
  socket "/auth_socket", HelloSocketsWeb.AuthSocket,
    websocket: true,
    longpoll: false
```

在 AuthSocket 模块里：

```elixir
defmodule HelloSocketsWeb.AuthSocket do
  use Phoenix.Socket
  require Logger

  channel "ping", HelloSocketsWeb.PingChannel
  channel "tracked", HelloSocketsWeb.TrackedChannel

  def connect(%{"token" => token}, socket) do
    case verify(socket, token) do
      {:ok, user_id} ->
        socket = assign(socket, :user_id, user_id)
        {:ok, socket}
      {:error, err} ->
        Logger.error("#{__MODULE__} connect error #{inspect(err)}")
        :error
    end
  end

  def connect(_, _socket) do
    Logger.error("#{__MODULE__} connect error missing params")
    :error
  end
end
```

verify 函数验证登录信息, 给 AuthSocket增加 id/1 函数：

```elixir
  @one_day 86400
  
  defp verify(socket, token) do
    Phoenix.Token.verify(socket, "salt identifier", token, max_age: @one_day)
  end

  def id(%{assigns: %{user_id: user_id}}) do
    "auth_socket:#{user_id}"
  end
```

生成 secret_key_base

```shell
mix phx.gen.secret
```

### 重要安全提示

> Phoenix.Token.verify/4 。salt 增加保护，可以写在配置文件里，所有的用户都用一个。
*Phoenix.Token 使用 sccret key （secret_key_base) 来签名所有数据，这个 key必须严格保密。*
在生产环境上，可以放在环境变量里。不能把这个Key放到源码仓库里！
任何人有了这个key就可以构造有效的token，做好保护。
Phoenix.Token 对消息签名，阻止篡改信息，但并不加密数据。签名的信息里，可以让user_id 之类的信息可见，不要将任何敏感信息放在签名的消息里，如密码，或个人的其他信息。

#### 实验

未提供token，提供假的token：

```shell
$ wscat -c 'ws://localhost:5000/auth_socket/websocket?vsn=2.0.0'
error: Unexpected server response: 403

$ wscat -c 'ws://localhost:5000/auth_socket/websocket?vsn=2.0.0&token=xxx'
error: Unexpected server response: 403
```

服务端信息：

```elixir
iex(3)> [error] Elixir.HelloSocketsWeb.AuthSocket connect error missing params
[info] REFUSED CONNECTION TO HelloSocketsWeb.AuthSocket in 279µs
  Transport: :websocket
  Serializer: Phoenix.Socket.V2.JSONSerializer
  Parameters: %{"vsn" => "2.0.0"}
[error] Elixir.HelloSocketsWeb.AuthSocket connect error :invalid
[info] REFUSED CONNECTION TO HelloSocketsWeb.AuthSocket in 17ms
  Transport: :websocket
  Serializer: Phoenix.Socket.V2.JSONSerializer
  Parameters: %{"token" => "xxx", "vsn" => "2.0.0"}

```

生成一个合法的签名数据, 123 是userid：

```elixir
iex(5)> Phoenix.Token.sign(HelloSocketsWeb.Endpoint, "salt identifier", 123)
"SFMyNTY.g2gDYXtuBgAXjCXjcwFiAAFRgA.8KIYT8-K1tNWPUAUF-uoFKDRNPOFlTgxjXN95EZd0rQ"
```

实验连接：

```shell
$ wscat -c 'ws://localhost:5000/auth_socket/websocket?vsn=2.0.0&token=SFMyNTY.g2gDYXtuBgAXjCXjcwFiAAFRgA.8KIYT8-K1tNWPUAUF-uoFKDRNPOFlTgxN95EZd0rQ'
Connected (press CTRL+C to quit)
> ["1","1", "ping", "phx_join",{}]
< ["1","1","ping","phx_reply",{"response":{},"status":"ok"}]
```

服务端：

```elixir
iex(7)> [info] CONNECTED TO HelloSocketsWeb.AuthSocket in 412µs
  Transport: :websocket
  Serializer: Phoenix.Socket.V2.JSONSerializer
  Parameters: %{"token" => "SFMyNTY.g2gDYXtuBgAXjCXjcwFiAAFRgA.8KIYT8-K1tNWPUAUF-uoFKDRNPOFlTgxjXN95EZd0rQ", "vsn" => "2.0.0"}

nil
iex(8)> [info] JOINED ping in 30µs
  Parameters: %{}
```

### 不同类型的 Token

Phoenix.Token 是 Elixir特定的解决方案，有时需要跨语言的解决方案。

JWT JSON Web Token, 与Phoenix.Token类似，但是是标准的格式，可以在各种语言里使用。
Elixir里用Joken 库处理 JWT。

不论用什么方案，要注意设置token的过期时间。

## 为 Channel 增加认证功能

一些channel 的 topic 是给特定用户服务的，如 "user_info:{user-id}" ，限定特定用户join。
用户 join时，调用 Channel 的 join/3 回调函数，在这里做认证，判断检查token。
可以用两种方式：

- 基于 parameter
- 基于 socket state, 推荐，简单，安全。
  
下面试例子：

auth_socket.ex 里添加 channel路由：

```elixir
channel "user:*", HelloSocketsWeb.AuthChannel
```

AuthChannel：

```elixir
defmodule HelloSocketsWeb.AuthChannel do
  use Phoenix.Channel

  require Logger

  def join("user:" <> req_user_id,
        _payload,
        socket = %{assigns: %{user_id: user_id}}
      ) do

    if req_user_id == to_string(user_id) do
      {:ok, socket}
    else
      Logger.error("#{__MODULE__} failed! #{req_user_id} != #{user_id}")
      {:error, %{reason: "unauthorized"}}
    end
  end
end
```

实验，用户的user_id 是 123， 先join user:1 被拒，然后join user:123 成功：

```shell
$ wscat -c 'ws://localhost:5000/auth_socket/websocket?vsn=2.0.0&token=SFMyNTY.g2gDYXtuBgAXjCXjcwFiAAFRgA.8KIYT8-K1tNWPUAUF-uoFKDRNPOFlTgxjXN95EZd0rQ'
Connected (press CTRL+C to quit)
> ["1","1","user:1","phx_join",{}]
< ["1","1","user:1","phx_reply",{"response":{"reason":"unauthorized"},"status":"error"}]
> ["1","1","user:123","phx_join",{}]
< ["1","1","user:123","phx_reply",{"response":{},"status":"ok"}]
```

服务端:

```elixir
[info] CONNECTED TO HelloSocketsWeb.AuthSocket in 11ms
  Transport: :websocket
  Serializer: Phoenix.Socket.V2.JSONSerializer
  Parameters: %{"token" => "SFMyNTY.g2gDYXtuBgAXjCXjcwFiAAFRgA.8KIYT8-K1tNWPUAUF-uoFKDRNPOFlTgxjXN95EZd0rQ", "vsn" => "2.0.0"}
[error] Elixir.HelloSocketsWeb.AuthChannel failed! 1 != 123
[info] REFUSED JOIN user:1 in 186µs
  Parameters: %{}
[info] JOINED user:123 in 67µs
  Parameters: %{}
```

## JavaScript 中使用认证

controller 里生成一个token，渲染到页面里。

```elixir
defmodule HelloSocketsWeb.PageController do
  use HelloSocketsWeb, :controller

  def index(conn, _params) do
    fake_user_id = 123

    conn
    |> assign(:auth_token, generate_auth_token(conn, fake_user_id))
    |> assign(:user_id, fake_user_id)
    |> render("index.html")
  end

  defp generate_auth_token(conn, user_id) do
    Phoenix.Token.sign(conn, "salt identifier", user_id)
  end
end
```

页面模板里，把 token和userid写入：

```html
<script>
  window.authToken = "<%= assigns[:auth_token] %>";
  window.userId = "<%= assigns[:user_id] %>";
</script>
```

Tips：

- 可以放在layout 模板里，让各个页面自动包含
- Token的生成可以放在 Plug 的 pipeline里，这样每个请求都可以生成。

socket.js

```js
let authSocket = new Socket("/auth_socket", {
  params: {token: window.authToken}
})

authSocket.onOpen(() => console.log("authSocket connected"))
authSocket.connect()
```

## 什么时候需要写一个新的Socket

主要考虑是否有用户认证的需求。

添加Socket的成本：
每个连接的socket，都给server增加一个连接。但增加Channel，不会给server增加连接。
Channel的成本比Socket低得多。

每个Socket都要维护到服务器的心跳。多个Channel在一个socket上，只需要一个心跳。

admin 后台，需要增加一个自己单独的Socket，跟用户的分隔开。

- 多个Channel 用一个Socket
- 如果应用有不同的认证需求，使用多个Socket （比如用户侧的登录，与管理员侧的登录）
