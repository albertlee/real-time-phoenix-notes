# Real-Time Phoenix 6. Avoid Performance Pitfalls

## By: 韩祝鹏 2020-8-13

系统设计阶段就要考虑到一些扩展时可能遇到的坑。

我们关注下面三方面性能陷阱：

- 不知道应用的健康状况，用 StatsD监控系统运行状况
- Channel吞吐量受限， channel使用单进程来收发信息，可能被耗时的请求阻塞，用Phoenix内置的函数解决
- 设计不良好的数据管道，GenStage

## 测量一切

复杂的应用，有很多环节，经常一个环节出问题会影响其他部分，甚至让整个应用不可用。（这方面我有很惨痛的经历）。

APM (Application Performance Monitoring)是市场上提供的性能监控工具，往往是收费的。

有一些开源的工具可以用。

### 测量的类型

对各种事件和操作进行测量：

- 出现次数计数 计算Channel收到消息的次数，或Socket连接失败的次数等
- 某个时间点上的数值，如当前连接的Socket 和 Channel的数量，当前在线用户数等， gauge
- 操作的用时

组合使用多种监控测量的值，帮助判断性能问题。

### 用StatsD 收集测量值

用 Statix 库：

```elixir
      {:statix, "~> 1.4"},
      {:statsd_logger, "~> 1.1", only: [:dev, :test]}
```

配置好端口，

```elixir
config :statsd_logger, port:  8126
config :statix, HelloSockets.Statix, port: 8126
```

添加服务模块

```elixir
defmodule HelloSockets.Statix do
  use Statix
end
```

在 application.ex 里启动 Statix

```elixir
  def start(_type, _args) do
    :ok = HelloSockets.Statix.connect()
```

创建一个 Socket， 在connect的时候发出一个打点数据：

```elixir
defmodule HelloSocketsWeb.StatsSocket do
  use Phoenix.Socket

  channel "*", HelloSocketsWeb.StatsChannel

  def connect(_params, socket, _connect_info) do
    HelloSockets.Statix.increment("socket_connect", 1,
      tags: ["status:success", "socket:StatsSocket"])

    {:ok, socket}
  end

  def id(_socket), do: nil
end
```

StatsChannel , 对 join 进行计数， 测量 handle_in 的耗时:

```elixir
defmodule HelloSocketsWeb.StatsChannel do
  use Phoenix.Channel

  def join("valid", _payload, socket) do
    channel_join_increment("success")
    {:ok, socket}
  end

  def join("invalid", _payload, _socket) do
    channel_join_increment("fail")
    {:error, %{reason: "always fails"}}
  end

  defp channel_join_increment(status) do
    HelloSockets.Statix.increment("channel_join", 1,
      tags: ["status:#{status}", "channel:StatsChannel"]
    )
  end

  def handle_in("ping", _payload, socket) do
    HelloSockets.Statix.measure("stats_channel.ping", fn ->
      Process.sleep(:rand.uniform(1000))
      {:reply, {:ok, %{ping: "pong"}}, socket}
    end)
  end
end
```

measure/2 函数接受一个函数，它执行并计时。

socket.js 里添加：

```js
let statsSocket = new Socket("/stats_socket", {})
statsSocket.connect()

let statsChannelInvalid = statsSocket.channel("invalid")
statsChannelInvalid.join()
  .receive("error", () => statsChannelInvalid.leave())

let statsChannelValid = statsSocket.channel("valid")
statsChannelValid.join()

for (let i = 0; i < 5; i++) {
  statsChannelValid.push("ping")
}
```

服务端显示：

```elixir
StatsD metric: socket_connect 1|c|#status:success,socket:StatsSocket
StatsD metric: channel_join 1|c|#status:fail,channel:StatsChannel
StatsD metric: channel_join 1|c|#status:success,channel:StatsChannel
iex(server@127.0.0.1)6> StatsD metric: stats_channel.ping 897|ms
StatsD metric: stats_channel.ping 586|ms
StatsD metric: stats_channel.ping 453|ms
iex(server@127.0.0.1)7> StatsD metric: stats_channel.ping 918|ms
StatsD metric: stats_channel.ping 428|ms
```

### 可视化测量信息

有很多收费或开源的系统可以收集分析这些监控信息，能够

- 图形展示监控数据
- 创建Dashboard
- 问题警报
- 异常检测，基于统计等方法，检测异常，例如某些事件数值超出一定标准差，预示着潜在问题

识别问题，改进问题

## 让 Channels 保持异步

(性能改进的几个常用手段：异步、Cache、减少IO)

Elixir是并行执行的，多个Channel可以并行执行，但是单个Channel是串行的。如果一个Channel 处理一个缓慢的消息，那它就被阻塞了。

给 StatsChannel 添加一个慢处理函数：

```elixir
  def handle_in("slow_ping", _payload, socket) do
    Process.sleep(3_000)
    {:reply, {:ok, %{ping: "pong"}}, socket}
  end
```

实际中，数据库的访问，网络调用，文件操作等都可能拖慢响应。

Phoenix里用 Task， 创建工作进程来并发的处理这些耗时的请求。

> *注意*
> 用 socket_ref 来避免大量的内存copy！socket 里可能包含大量的状态数据。
> 实际应用里，把处理消息放在其他的GenServer里，并通过 socket_ref 来传递。
> 注意另外的坑：客户端可能发起太多的请求，服务端如果不做限制会给一个用户开太多的处理进程。需要做限流，限制一个Channel同时开的任务进程。

```elixir
  def handle_in("parallel_slow_ping", _payload, socket) do
    ref = socket_ref(socket)

    Task.start_link(fn ->
      Process.sleep(3_000)
      Phoenix.Channel.reply(ref, {:ok, %{ping: "pong"}})
    end)
    {:noreply, socket}
  end
```

## 构造一个可扩展的数据流水线 Data Pipeline

用户需要尽快获取最新的数据，要有意识的设计应用的数据流。处理外发的实时数据的机制就是 data pipeline数据管道。

### 数据流水线的特质

#### 向所有相关的客户端发送消息

> 实时事件要向所有连接的结点上广播，这些结点可以处理所有连接的Channel。PubSub帮我们做了，我们需要考虑数据管道跨多个服务器，注意不要给用户发送错误的数据。

#### 快速的数据发送

> 越快越好。用户能尽快收到消息更新状态，数据的生产方也不用担心发的太多影响性能。

#### 耐用

> 满足不同场景需求

#### 满足并发要求

> 可以限制并发性，避免应用被消息吞没。

#### 可测量

> 性能监控

### 用 GenStage 做 Pipeline

[GenStage](https://github.com/elixir-lang/gen_stage) 不是一个开箱即用的data pipeline，它提供了一个数据如何传递的规范。GenStage提供两个主要的类型来构造pipeline：

- Producer 生产者，可以从数据库里，也可以完全从内存里生产数据。并传给消费者
- Consumer 消费者，从之前的生产者stage请求并接收数据。

一个消费者也可以是其他消费者的生产者，pipeline就是把众多的生产者消费者串在一起。

```elixir
defmodule HelloSockets.Pipeline.Producer do
  use GenStage
  def start_link(opts) do
    {[name: name], opts} = Keyword.split(opts, [:name])
    GenStage.start_link(__MODULE__, opts, name: name)
  end

  def init(_opts) do
    {:producer, :unused, buffer_size: 10_000}
  end

  def handle_demand(_demand, state) do
    {:noreply, [], state}
  end

  def push(item = %{}) do
    GenStage.cast(__MODULE__, {:notify, item})
  end

  def handle_cast({:notify, item}, state) do
    {:noreply, [%{item: item}], state}
  end
end
```

init/1 返回 {:producer, state}，表明我们写的是一个生产者。

consumer：

```elixir
defmodule HelloSockets.Pipeline.Consumer do
  use GenStage

  def start_link(opts) do
    GenStage.start_link(__MODULE__, opts)
  end

  def init(opts) do
    subscribe_to =
      Keyword.get(opts, :subscribe_to, HelloSockets.Pipeline.Producer)

    {:consumer, :unused, subscribe_to: subscribe_to}
  end

  def handle_events(items, _from, state) do
    IO.inspect(
      {__MODULE__, length(items), List.first(items), List.last(items)}
    )
    {:noreply, [], state}
  end
end
```

指定一个 Producer，可以通过opts动态配置。每个消费者必须有一个 handle_events 回调函数。

在application 中将 Producer, Consumer 挂到监督树下启动：(放在Endpoint之前)

```elixir
  children = [
      # ...
      {Producer, name: Producer},
      {Consumer,
        subscribe_to: [{Producer, max_demand: 10, min_demand: 5}]
      },
      HelloSocketsWeb.Endpoint
```

min_demand，max_demand 的设置，在用到外部数据的时候可以调大点，减少IO操作。根据这个设置，Consumer会把消息切成一批批处理。

#### 增加并发和Channels

可扩展的数据管道需要同时处理多个数据项，因此必须是并发的。GenStage用 ConsumerSupervisor模块增加并发特性。这个模块让我们专注在定义pipeline，让库去处理并发。

ConsumerSupervisor 是GenStage 的一种消费者类型，它为收到的每个数据项spawn 子进程，这些子进程不会被复用，不过在 Elixir(Erlang)里进程的开销非常小。（转变下思维，“浪费”点创建进程的时间，比重用它来的划算，不仅简单可靠，而且从性能上来说没准更好）

consumer_supervisor:

```elixir
defmodule HelloSockets.Pipeline.ConsumerSupervisor do
  use ConsumerSupervisor
  alias HelloSockets.Pipeline.{Producer, Worker}

  def start_link(opts) do
    ConsumerSupervisor.start_link(__MODULE__, opts)
  end

  def init(opts) do
    subscribe_to = Keyword.get(opts, :subscribe_to, Producer)
    supervisor_opts = [strategy: :one_for_one, subscribe_to: subscribe_to]

    children = [
      %{id: Worker, start: {Worker, :start_link, []}, restart: :transient}
    ]
    ConsumerSupervisor.init(children, supervisor_opts)
  end
end
```

worker：

```elixir
defmodule HelloSockets.Pipeline.Worker do
  def start_link(item) do
    Task.start_link(fn ->
      process(item)
    end)
  end

  defp process(item) do
    IO.inspect(item)
    Process.sleep(1000)
  end
end
```

将 worker模块的 process 替换，向特定用户push消息：

```elixir
  defp process(%{item: %{data: data, user_id: user_id}}) do
    #IO.inspect(item)
    Process.sleep(1000)
    HelloSocketsWeb.Endpoint.broadcast!("user:#{user_id}", "push", data)
  end
```

Pipeline 里，将消息广播到特定的topic里，Channel会把消息发送到客户端。

socket.js

```js
let authUserChannel = authSocket.channel(`user:${window.userId}`)
authUserChannel.on("push", (payload) => {
  console.log("received auth user push: ", payload)
})
authUserChannel.join()
```

### 监测Pipeline性能

要确定我们的软件功能正常，监控系统的健康情况，我们对 Worker的处理时间，以及广播消息花费的时间进行监控。

我们手动触发一个计时，统计从消息的产生到推送总共的时间。

很容易就可以增加Worker处理事件的测量功能：

```elixir
defmodule HelloSockets.Pipeline.Worker do
  def start_link(item) do
    Task.start_link(fn ->
      HelloSockets.Statix.measure("pipeline.worker.process_time", fn ->
        process(item)
      end)
    end)
  end
```

要测量总的投递用时有点复杂，因为没办法把整个pipeline包起来，我们可以在数据项入队时记录一个时间，在Channel的外发事件上进行拦截，再算一个时间差。

判断时间差用 erlang去系统时间，因为分布式系统，有可能要测量的事件分布到多台机器上，时间可能有些不准确。

消息从 Producer里入队时记录当前时间, 在 push_timed 方法里开始记录时间，而不是 handle_cast ，因为Producer有可能阻塞，处理延迟，这个时间需要记录下来。

```elixir
  def push_timed(item = %{}) do
    GenStage.cast(__MODULE__, {:notify_timed, item, Timing.unix_ms_now()})
  end
  def handle_cast({:notify_timed, item, unix_ms}, state) do
    {:noreply, [%{item: item, enqueued_at: unix_ms}], state}
  end
```

修改 Worker，把 enqueued_at 放到广播里

```elixir
  defp process(%{item: %{data: data, user_id: user_id}, enqueued_at: unix_ms}) do
    HelloSocketsWeb.Endpoint.broadcast!("user:#{user_id}", "push_timed",
      %{data: data, at: unix_ms})
  end
```

做时间测量时，经常用 Statix.histogram 汇总统计。

### 测试数据流水线

我们可以分开测试流水线里的每个组件，也可以集成测试整个流水线。

测试文件：

```elixir
defmodule Integration.PipelineTest do
  use HelloSocketsWeb.ChannelCase, async: false
  alias HelloSocketsWeb.AuthSocket
  alias HelloSockets.Pipeline.Producer

  defp connect_auth_socket(user_id) do
    {:ok, _, %Phoenix.Socket{}} =
      socket(AuthSocket, nil, %{user_id: user_id})
      |> subscribe_and_join("user:#{user_id}", %{})
  end

  test "event are pushed from begining to end correctly" do
    connect_auth_socket(1)

    Enum.each(1..10, fn n ->
      Producer.push_timed(%{data: %{n: n}, user_id: 1})
      assert_push "push_timed", %{n: ^n}
    end)
  end

  test "an event is not delivered to the wrong user" do
    connect_auth_socket(2)
    Producer.push_timed(%{data: %{test: true}, user_id: 1})
    refute_push "push_timed", %{test: true}
  end

  test "events are timed on delivery" do
    assert {:ok, _} = StatsDLogger.start_link(port: 8127, formatter: :send)
    connect_auth_socket(1)
    Producer.push_timed(%{data: %{test: true}, user_id: 1})
    assert_push "push_timed", %{test: true}
    assert_receive {:statsd_recv, "pipeline.push_delivered", _value}
  end
end
```

async: false ， 使用同步的方式，避免随机的测试失败。
从 Producer 创建事件开始，到Channel 发出特定消息，整个流程测试。
用 StatsDLogger 来测试 StatsD 的统计数据被正确发送。

### GenStage 的力量

我们的应用随着时间会增长与变化。GenStage可以随着我们的应用成长，从简单开始，逐步增加需要的部分。

> 我的想法：
> 程序是长出来的，这是我一直以来的观点。一个编程语言，框架，系统设计需要容易并鼓励演化。
> 《SICP》序言里有很深刻的讲解。




