# Real-Time Phoenix 5. Dive Deep into Phoenix Channels

## By: 韩祝鹏 2020-8-12

本章讨论如下问题：

- 不可靠的网络连接，网络异常、应用bug、服务重启等问题
- Channel在多服务器上的使用
- 测试Socket和Channel 代码。

## 为不可靠的网络连接设计

各种原因都会导致连接不稳定或中断，如断网、电脑休眠、客户端bug、服务端升级等。

### Channel Subscriptions

Client连接到特定的Channel，这个记录放在内存里。如果client断开，重连时需要重新subscript各个topic。官方的JavaScript客户端已经自动处理了这种情况。如果是自己写的客户端，需要处理这种情况。

### 保持关键数据存活

客户端断开后，与连接相关的Process就关闭了，里面的数据也没了。因此当要在一个进程里保存状态时，要考虑到进程什么情况会被关闭。*所有业务相关的数据应该存储在持久化存储中，可以经受系统的重启。* （我的理解是最坏的情况能恢复。重要的数据在持久化存储里保存一份，性能考虑在内存的Process里保存最新的状态，只要把变化的写操作日志持久化出去，能恢复就行）

> *注意* 每个Channel 都是一个 GenServer，每个client join进一个Channel，就会创建这个连接相应的 Channel 进程。客户端连接中断后，相应的channel进程就结束。

- Channel与一个业务进程进行交互，业务进程使用持久化的数据源，保存数据。
- 创建一个功能内核，在通讯层与业务逻辑层之间保持边界。
  
Channel聚焦在实时通讯的职责上，避免在channel里实现业务逻辑。
把关键的数据可以放在内存里，这样会极大的提高性能，要考虑的是进程和服务器重启的情况，数据状态能恢复。

### Message Delivery

Phoenix Channel 采用 *at-most-once* 策略发送消息到客户端。意味着客户端收到0次或1次消息。另一种不同的策略是*at-least-once* ，消息会收到一次或多次。由于分布式系统的不确定性，不会有 exactly-once 策略。

At-most-once 策略是一个设计上的取舍，我们不需要去实现一个保证每个消息都可能处理多次的系统，而那个的潜在复杂度更大。

At-most-once在下面的应用场景比较好：

- 丢失消息不会影响打断应用的流程
- 愿意做取舍，用可能丢失数据来换取开发的低成本
- 应用和客户端可以手动的从丢失的消息中恢复

（由于我们现在的业务需求不涉及到重要的支付财务等业务，因此 at-most-once够用，可以容忍消息的丢失）

在PubSub里，向远程节点广播时，只发送一次，没有确认和重发机制。

Phoenix在给客户端发消息时，也没有任何确认。

如果需要保证 at-least-once ，就需要自己写代码实现确认与重发机制，这个复杂性就比较大了。

## 集群中使用 Channel

横向扩展比垂直扩展方便，加机器数量比换CPU快。垂直扩展也会遇到极限。
Phoenix用PubSub 在集群里广播消息。

### 连接本地集群

启动服务结点，和另一个节点。 --name 设定node的名字。

```shell
$iex --name server@127.0.0.1 -S mix phx.server
$iex --name remote@127.0.0.1 -S mix
```

在remote节点上，进行node连接：

```elixir
iex(remote@127.0.0.1)1> Node.list()
[]
iex(remote@127.0.0.1)2> Node.connect(:"server@127.0.0.1")
true
iex(remote@127.0.0.1)3> Node.list()
[:"server@127.0.0.1"]
```

在remote节点上进行了 Node.connect 后，其他的结点自动连接，在server结点上可以看到：

```elixir
iex(server@127.0.0.1)5> Node.list()
[:"remote@127.0.0.1"]
```

尝试启动第3个结点，然后连接到之前的两个节点的集群里：

```elixir
$ iex --name third@127.0.0.1 -S mix
Erlang/OTP 23 [erts-11.0.2] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1] [hipe] [dtrace]

Interactive Elixir (1.10.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(third@127.0.0.1)1> Node.list
[]
iex(third@127.0.0.1)2> Node.connect(:"remote@127.0.0.1")
true
iex(third@127.0.0.1)3> Node.list
[:"remote@127.0.0.1", :"server@127.0.0.1"]
```

third连接了 remote后，在 server节点上，也自动连接上了third：

```elixir
iex(server@127.0.0.1)6> Node.list()
[:"remote@127.0.0.1", :"third@127.0.0.1"]
```

remote和third结点没有运行web服务器，没有Socket连接，我们在 remote(或 third)结点上broadcast一个消息，客户端都可以接收到。从客户端的消息，可以看到消息来源自  "server@127.0.0.1" 。

```elixir
iex(remote@127.0.0.1)6> HelloSocketsWeb.Endpoint.broadcast("ping", "request_ping", %{})
:ok
```

在remote结点上广播消息，通过PubSub在集群里广播，server结点接到消息发给客户端。

在集群的任意结点上都可以发送消息。

在实际中，remote结点也可能提供Socket连接服务，整个系统部署在负载均衡后面。

修改配置文件，将 PORT改成读取环境变量：

```elixir
config :hello_sockets, HelloSocketsWeb.Endpoint,
  http: [port: String.to_integer(System.get_env("PORT") || "5000")],
```

重新启动 remote服务器：

```shell
$PORT=5001 iex --name remote@127.0.0.1 -S mix phx.server
```

### 分布式Channel的挑战

分布式系统在扩展性上很有好处，但是比单节点的应用要复杂。内部系统，往往单节点是合适的选择，但是大量用户的场景就需要考虑分布式的问题了。

- 我们无法完全精确的知道远程节点的状态，用技术和算法可以降低不确定性，但无法完全避免。
- 消息可能无法按预期发送给远程节点，完全丢失情况少，但经常会延迟。
- 在各种场景下进行测试很复杂
- 客户端可能会断开连接并连到另一个节点上，需要一个中心来做数据的参考，最常用的就是共享一个数据库。

### Customize Chnnael Behavior 定制Channel的行为

Phoenix Channel 基于 GenServer，因此可以接收消息并存储状态。通过定制可以做到标准的消息广播难以做到的事情，比如向单独一个客户发送消息。

#### 发送循环的消息

周期向客户端发送消息（比如定期刷新token），避免用户同一时间请求服务器。

Channel 用 Process.send_after/3 可以定时向自身发送消息。可以在进程启动时开始定时，也可以随时启动（比如在 handle_in方法中）。

下面的例子，Channel中通过send_after 定时发送token给客户端：

recurring_channel.ex

```elixir
defmodule HelloSocketsWeb.RecurringChannel do
  use Phoenix.Channel

  @send_after 5_000

  def join(_topic, _payload, socket) do
    schedule_send_token()
    {:ok, socket}
  end

  defp schedule_send_token do
    Process.send_after(self(), :send_token, @send_after)
  end

  def handle_info(:send_token, socket) do
    schedule_send_token()
    push(socket, "new_token", %{token: new_token(socket)})
    {:noreply, socket}
  end

  defp new_token(socket = %{assigns: %{user_id: user_id}}) do
    Phoenix.Token.sign(socket, "salt identifier", user_id)
  end
end
```

socket.js

```js
let recurringChannel = authSocket.channel("recurring")

recurringChannel.on("new_token", (payload) => {
  console.log("received new auth token:", payload)
})
recurringChannel.join()
```

#### 重复的外发消息

要阻止重复的外发消息，解决方案尽量离用户端近，Channel是单个用户与服务器间最低层级的进程，因此在Channel级做这件事。

在本例子里，我们用 Socket.assigns 保存与Channel相关的状态。

在一个Channel里对 Socket.assigns 的数据，不会影响到其他的channel，即使是用的同一个socket。因为Elixir是函数式的，channel启动时，socket复制进来不变了。

（因为 Channel 是一个  GenServer，这里的socket 其实就是进程的 state， 见上面的 handle_info 函数）

（例子里，往 buffer 列表里加入新的消息，往列表的头上加，消耗时间是常数级别的，因为是链表。这是Erlang/Elixir 的惯用法）

```elixir
defmodule HelloSocketsWeb.DedupeChannel do
  use Phoenix.Channel
  intercept ["number"]

  def join(_topic, _payload, socket) do
    {:ok, socket}
  end

  def handle_out("number", %{number: number}, socket) do
    buffer = Map.get(socket.assigns, :buffer, [])
    next_buffer = [number | buffer]

    next_socket =
      socket
      |> assign(:buffer, next_buffer)
      |> enqueue_send_buffer()

    {:noreply, next_socket}
  end

  defp enqueue_send_buffer(socket = %{assigns: %{awaiting_buffer?: true}}), do: socket

  defp enqueue_send_buffer(socket) do
    Process.send_after(self(), :send_buffer, 1_000)
    assign(socket, :awaiting_buffer?, true)
  end

  def handle_info(:send_buffer, socket = %{assigns: %{buffer: buffer}}) do
    buffer
    |> Enum.reverse()
    |> Enum.uniq()
    |> Enum.each(&push(socket, "number", %{value: &1}))

    next_socket =
      socket
      |> assign(:buffer, [])
      |> assign(:awaiting_buffer?, false)

    {:noreply, next_socket}
  end

  def broadcast(numbers, times) do
    Enum.each(1..times, fn _ ->
      Enum.each(numbers, fn number ->
        HelloSocketsWeb.Endpoint.broadcast!("dupe", "number", %{
          number: number
        })
      end)
    end)
  end
end
```

实验下：

```elixir
iex(server@127.0.0.1)3> HelloSocketsWeb.DedupeChannel.broadcast([1,2,3], 100)
:ok
```

客户端只收到3条消息，而不是300条。

## 写测试

Phoeniex 框架里提供了对 Socket 和Channel 测试的方法，不需要操心 WebSocket 或 Long Polling。

### 测试 Sockets

mix phx.new 创建Phoenix项目后，会包含一些辅助测试的模块，在 test/support 下， ChannelCase.

```shell
$mix test
```

UserSocketTest:

```elixir
defmodule HelloSocketsWeb.UserSocketTest do
  use HelloSocketsWeb.ChannelCase
  alias HelloSocketsWeb.UserSocket

  describe "connect/3" do
    test "can be connected to without parameters" do
      assert {:ok, %Phoenix.Socket{}} = connect(UserSocket, %{})
    end
  end

  describe "id/1" do
    test "an identifier is not provided" do
      assert {:ok, socket} = connect(UserSocket, %{})
      assert UserSocket.id(socket) == nil
    end
  end
end
```

测试AuthSocket ：

```elixir
defmodule HelloSocketsWeb.AuthSocketTest do
  use HelloSocketsWeb.ChannelCase
  import ExUnit.CaptureLog
  alias HelloSocketsWeb.AuthSocket

  defp generate_token(id, opts \\ []) do
    salt = Keyword.get(opts, :salt, "salt identifier")
    Phoenix.Token.sign(HelloSocketsWeb.Endpoint, salt, id)
  end

  describe "connect/3 success" do
    test "can be connect to with a valid token" do
      assert {:ok, %Phoenix.Socket{}} =
        connect(AuthSocket, %{"token" => generate_token(1)})
      assert {:ok, %Phoenix.Socket{}} =
        connect(AuthSocket, %{"token" => generate_token(2)})
    end
  end

  describe "connect/3 error" do
    test "cannot be connected to with an invalid salt" do
      params = %{"token" => generate_token(1, salt: "invalid")}

      assert capture_log(fn ->
        assert :error = connect(AuthSocket, params)
      end) =~ "[error] #{AuthSocket} connect error :invalid"
    end

    test "cannot be connected to without a token" do
      params = %{}

      assert capture_log(fn ->
        assert :error = connect(AuthSocket, params)
      end) =~ "[error] #{AuthSocket} connect error missing params"
    end

    test "cannot be connected to with an fake token" do
      params = %{"token" => "nonsense"}

      assert capture_log(fn ->
        assert :error = connect(AuthSocket, params)
      end) =~ "[error] #{AuthSocket} connect error :invalid"
    end
  end

  describe "id/1" do
    test "an identifier is based on the connected ID" do
      assert {:ok, socket} =
        connect(AuthSocket, %{"token" => generate_token(1)})

      assert AuthSocket.id(socket) == "auth_socket:1"

      assert {:ok, socket} =
        connect(AuthSocket, %{"token" => generate_token((2))})
      assert AuthSocket.id(socket) == "auth_socket:2"
    end
  end
end
```

### 测试 Channels

Channel 比 Socket 有跟多的业务逻辑，因此测试的需求更大。对Channel的测试核心是消息的传递，测试要验证测试进程与Channel进程正确的发送和接受消息。

#### WildcardChannelTest:

测试代码里，connect/3 函数返回一个 Phoenix.Socket 结构，可以方便的初始化一个状态，不需要实际去连接Socket。

用 subscribe_and_join/3 来join到给定的topic里。

错误的topic格式导致 WildcardChannel 崩溃，通过 capture_log 捕捉错误信息。

assert_reply/3 用于判断发送的回应消息是否正确

例子里用了 ^reply 的方法，而不是模式匹配的方式，以排除 %{ping: "pong", extra: true}这种错误通过测试的情况。

```elixir
defmodule HelloSocketsWeb.WildcardChannelTest do
  use HelloSocketsWeb.ChannelCase
  import ExUnit.CaptureLog
  alias HelloSocketsWeb.UserSocket

  describe "join/3 success" do
    test "ok when numbers in the format a:b when b = 2a" do
      assert {:ok, _, %Phoenix.Socket{}} =
        socket(UserSocket, nil, %{})
        |> subscribe_and_join("wild:2:4", %{})
    end
  end

  describe "join/3 error" do
    test "error when b is note exactly twice a" do
      assert socket(UserSocket, nil, %{})
        |> subscribe_and_join("wild:1:3", %{}) == {:error, %{}}
    end
    test "error when 3 numbers are provided" do
      assert socket(UserSocket, nil, %{})
        |> subscribe_and_join("wild:1:2:3", %{}) == {:error, %{}}
    end
  end

  describe "join/3 error causing crash" do
    test "error with an invalid format topic" do
      assert capture_log(fn ->
        socket(UserSocket, nil, %{})
          |> subscribe_and_join("wild:invalid", %{})
      end) =~ "[error] an exception was raised"
    end
  end

  describe "handle_in ping" do
    test "a pong response is provided" do
      assert {:ok, _, socket} =
        socket(UserSocket, nil, %{})
        |> subscribe_and_join("wild:2:4", %{})

      ref = push(socket, "ping", %{})
      reply = %{ping: "pong"}
      assert_reply ref, :ok, ^reply
    end
  end
end
```

#### 测试 DedupeChannel

Tips
> Elixir Pipeline 的惯用写法，函数的第一个参数，然后再返回这个参数，就可以把它放入pipeline里串起来用。见下面代码。把 socket 作为第一个参数，并返回socket，这样多个函数就可以串在一起。

用 :sys.get_state/1 获取一个指定进程的状态。这种方法要谨慎使用，放测试里或调试时用，业务逻辑一般不要使用。

refute_push/2 确定没有向client发送消息。
assert_push/2 确定发送了消息。

assert_push 在大部分情况下适用，但是不能检查消息的顺序。可以手动检查进程里的消息，以确定消息发送的顺序。

```elixir
defmodule HelloSocketsWeb.DedupeChannelTest do
  use HelloSocketsWeb.ChannelCase
  alias HelloSocketsWeb.UserSocket

  defp broadcast_numbers(socket, number) do
    assert broadcast_from!(socket, "number", %{number: number}) == :ok
    socket
  end

  defp validate_buffer_contents(socket, expected_contents) do
    assert :sys.get_state(socket.channel_pid).assigns == %{
      awaiting_buffer?: true,
      buffer: expected_contents
    }
    socket
  end

  defp connect() do
    assert {:ok, _, socket} =
      socket(UserSocket, nil, %{})
      |> subscribe_and_join("dupe", %{})
    socket
  end

  test "a buffer is maintained as numbers are broadcasted" do
    connect()
    |> broadcast_numbers(1)
    |> validate_buffer_contents([1])
    |> broadcast_numbers(1)
    |> validate_buffer_contents([1, 1])
    |> broadcast_numbers(2)
    |> validate_buffer_contents([2, 1, 1])

    refute_push _, _

  end

  test "the buffer is drained 1 second after a number is first added" do
    connect()
    |> broadcast_numbers(1)
    |> broadcast_numbers(1)
    |> broadcast_numbers(2)

    Process.sleep(1050)

    assert_push "number", %{value: 1}, 0
    refute_push "number", %{value: 1}, 0
    assert_push "number", %{value: 2}, 0
  end

  test "the buffer drains with unique values in the correct order" do connect()
    |> broadcast_numbers(1)
    |> broadcast_numbers(2)
    |> broadcast_numbers(3)
    |> broadcast_numbers(2)

    Process.sleep(1050)
    assert {:messages, [
      %Phoenix.Socket.Message{
        event: "number",
        payload: %{value: 1}
      },
      %Phoenix.Socket.Message{
        event: "number",
        payload: %{value: 2}
      },
      %Phoenix.Socket.Message{
        event: "number",
        payload: %{value: 3}
      }
    ]} = Process.info(self(), :messages)
  end
end
```

## 要写测试啊
