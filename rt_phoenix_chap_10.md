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