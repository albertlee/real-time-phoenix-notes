# Real-Time Phoenix 10. Track Connected Carts with Presence

## By: hanzhupeng@gmail.com

构建一个管理后台,实时显示网站当前连接的用户.使用 Phoenix Tracker 和 Presence.

分布的状态是一个困难的问题,Phoenix Tracker让分布的进程列表管理较为容易.Tracker使用了一个高级的数据结构,在集群里高效准确的分布状态, 这让我们可以知道当前有多少Channel连接. 这个问题比较难处理,Tracker帮我们做到了.

这章最重要的功能是实现当前活跃的购物者用户列表.

## 规划Admin后台

### 需求

- 所有购物者的数量(去重)
- 购物者共打开多少页面
- 购物车中的商品数量
- 权限控制,限定管理员使用

### 把需求转化成计划

用一个专门的 Socke他给管理后台使用，增强安全性。

![架构图](images/chap_10_admin_arch.png)

Admin.DashboardController 负责认证和 Phoenix.Token 的创建。客户端用这个Token来连接到 Admin.Socket 。Admin.Socket 只允许管理员连接，因此后面的Channel就不需要加入topic的认证了。

用 Tracker 来知道当前有多少购物车连接。每一个 ShoppingCartChannel 在连接时都会在 CartTracker 中 track跟踪自己。

![Tracker架构图](images/chap_10_tracker.png)

CartTracker 来跟踪所有的客户端的购物车Channel，更新给 Admin客户端，并在多个服务器之间同步。

### 设置项目工程

略

### 用 Phoenix Tracker 来 Track

Phoenix Tracker 解决集群服务器之间多个进程的跟踪问题，这看上去简单，但是因为服务器之间复制信息的冲突会变得很困难。我们用 Tracker 来跟踪每一个连接的 ShoppingCartChannel 进程，以及每一个购物车的元数据。

#### Phoenix Tracker 的设计

Tracker 使用一个特殊的数据类型来跨集群复制信息。我们将研究这个数据结构，看看它提供了什么样的保证。

Phoenix Tracker 在集群跨多态服务器之间，保持一个精确的准时的在场列表。它不使用一个单一可信数据源（如数据库），每一个服务器都贡献已知状态。在遇到分布式状态时，时间不是我们的朋友，它让这个问题充满了挑战。分发状态的状态的改变，很容易遇到各种边际情况。冲突的数据，丢失的更新，低效率等。Phoenix Tracker 使用 CRDT (Conflict-free Replicated Data Type)。

CRDT在多个服务器之间复制状态，用独立的、并发的更新底层的数据，不需要请求其他的copy用于保证。CRDT有很多种类型，Phoenix Tracker用的是 ORSWOT (Observe-Remove-Set-Without-Tombstones)管理它的状态。（本书没有深入讲解这个数据结构的实现细节，拿来用就好）

Tracker 的进程结构：

![Tracker架构图](images/chap_10_tracker_2.png)

Phoenix.Tracker 模块是一群 Tracker.Shard 进程的 facade 门面接口。我们调用 Phoenix.Tracker 模块的函数，实际的数据由底层的shard 分片进程提供。这样设计避免了单进程的性能瓶颈。（跟数据库分库分表读一样）。Tracker 根据它跟踪的topic 字符串来进行分片，所以一个topic连接多个进程依然会遇到单进程的性能瓶颈。

每一个 Tracker.Shard 进程收集它自己状态的改变，然后通过 Phoenix PubSub广播到集群里其他的结点上。状态分布广播可以配置一个延迟时间来批量发送，不超过两秒钟。这使得Tracker 是最终一致性的，写操作不会立即反映到整个集群。

#### 在管理后台上使用 Phoenix Tracker

Tracker 最早用来在一个聊天app里用来实现“谁在线？”这样的问题，每个用户的Channel连接时进行跟踪，记录一些元数据，如他们的名字、uid。聊天室的每个客户端，实时的收到在线状态列表更新。后面看到如何用Presence(一个特殊类型的Tracker)来解决这个问题。

每一个 ShoppingCartChannel 在连接时会被跟踪，管理员可以看到谁在线。我们把一些元数据放到tracker里，这样就能知道各个购物车里有什么。Admin Dashboard会实时的汇集展示这些信息。

> Tracker in a Pipeline
> Tracker不止用于用户相关的场景，Tracker在数据流水线里也很有用。数据流水线在获取数据时经常消耗很大，要访问数据库或第三方的API，在调用数据访问前，我们可以先用Tracker来回答“谁在线？“

### 在应用程序中使用Tracker

创建一个模块来实现 Phoenix.Tracker behaviour，这个模块隐藏Tracker的函数调用，提供一个简单的接口。当我们需要的Channel进程join的时候进行跟踪。最后用 Phoenix.Tracker.list/2 函数来获得所有跟踪的数据。

Tracker 的样板代码如下，启动一个Tracker的 supervisor后，它启动一组 Phoenix.Tracker.Shard 进程。init/1 函数会在每个 Shard进程启动时调用，需要提供 pubsub_server key，否则Tracker将 crash。

```elixir
defmodule HelloSocketsWeb.UserTracker do
  @behaviour Phoenix.Tracker

  def child_spec(opts) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [opts]},
      type: :supervisor
    }
  end

  def start_link(opts) do
    opts =
      opts
      |> Keyword.put(:name, __MODULE__)
      |> Keyword.put(:pubsub_server, HelloSockets.PubSub)

    Phoenix.Tracker.start_link(__MODULE__, opts, opts)
  end

  def init(opts) do
    server = Keyword.fetch!(opts, :pubsub_server)
    {:ok, %{pubsub_server: server}}
  end
end
```

Tracker 需要实现一个 handle_diff/2 函数，这是用来放状态更新逻辑的地方。

```elixir
  require Logger
  def handle_diff(changes, state) do
    Logger.info inspect({"tracked changes", changes})
    {:ok, state}
  end

  def track(%{channel_pid: pid, topic: topic, assigns: %{user_id:   user_id}}) do
    metadata = %{
      online_at: DateTime.utc_now(),
      user_id: user_id
    }
    Phoenix.Tracker.track(__MODULE__, pid, topic, user_id, metadata)
  end

  def list(topic \\ "tracked") do
    Phoenix.Tracker.list(__MODULE__, topic)
  end
```

Phoenix.Tracker.track/5 是最重要的调用，它给定pid 和 topic。可以添加任意的元数据metadata，可以用来记录”谁“，”什么时候”joined的。

把 UserTrack加入到应用的监督树里：

```elixir
    children = [
        #....
      HelloSocketsWeb.Endpoint,
      {HelloSocketsWeb.UserTracker, [pool_size: :erlang.system_info(:schedulers_online)]}
    ]
```

Tips:

> Tracker 基于Topic来分片，因此相同topic的消息最终会是同一个进程来处理，在大量消息的情况下，还是可能会有性能瓶颈。

接下来创建新的TrackedChannel， 并在socket中添加路由。

```elixir
defmodule HelloSocketsWeb.TrackedChannel do
  use Phoenix.Channel

  alias HelloSocketsWeb.UserTracker

  def join("tracked", _payload, socket) do
    send(self(), :after_join)
    {:ok, socket}
  end

  def handle_info(:after_join, socket) do
    {:ok, _} = UserTracker.track(socket)
    {:noreply, socket}
  end
end
```

注意这里的 send(self(), :after_join) ，能快速的进行响应，然后异步的进行处理。这是elixir/erlang 里的惯用方法。

在前端 socket.js 里添加上 socket 和 channel相关的代码：

```js
const trackedSocket = new Socket("/auth_socket", {
  params: { token: window.authToken }
})
trackedSocket.connect()

const trackerChannel = trackedSocket.channel("tracked")
trackerChannel.join()
```

实际测试，打开一个 app 结点，一个back 结点，用 Node.connect 组成集群。
在浏览器上打开 http://localhost:5200/tracked?user_id=1
在 back结点上：

```elixir
iex(back@127.0.0.1)7> Node.connect(:"app@127.0.0.1")
true
iex(back@127.0.0.1)8> HelloSocketsWeb.UserTracker.list()
[]
iex(back@127.0.0.1)9> HelloSocketsWeb.UserTracker.list()
[]
iex(back@127.0.0.1)10> Node.list()
[:"app@127.0.0.1"]
iex(back@127.0.0.1)11> [info] {"tracked changes", %{"tracked" => {[{"1", %{online_at: ~U[2020-08-18 09:10:01.326849Z], phx_ref: "t3ileu4tEOc=", user_id: "1"}}], []}}}
 
nil
iex(back@127.0.0.1)12> 
nil
iex(back@127.0.0.1)13> HelloSocketsWeb.UserTracker.list()
[
  {"1",
   %{
     online_at: ~U[2020-08-18 09:10:01.326849Z],
     phx_ref: "t3ileu4tEOc=",
     user_id: "1"
   }}
]
```

尝试用不同的 user_id 参数打开多个tab，关闭其中的几个，可以看到 tracked changes 的 log信息， UserTracker.list() 的结果也会响应改变。Tracker把他的状态分布到整个集群上。

当 app结点关闭时， back结点上需要过一会改变状态。
当 app结点重启时，back结点需要重新 connect 一下，然后整个crdt 结构会全部重新发送：

```elixir
iex(back@127.0.0.1)56> Node.connect(:"app@127.0.0.1")
true
iex(back@127.0.0.1)57> [debug] {:"back@127.0.0.1", 1597741829729266}: falling back to sending entire crdt
```

在各个场景下，Tracker总能最终显示正确，但它可能会用一些时间（最多30秒），大部分改变感觉上立刻能生效。

handle_diff/2 函数在各个节点上都执行，你可以在里面做任何事情。

## Phoenix Tracker 与 Presence

