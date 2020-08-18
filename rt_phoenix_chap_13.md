# Real-Time Phoenix 13. Phoenix LiveView

## by: hanzhupeng@gmail.com 2020-8-19

前面我们看到 Phoenix Channel 让实时应用的服务端开发轻松了很多，但是依然需要写大量的前端代码。如果能不写那些JavaScript前端呢？看看 Phoenix LiveView。

LiveView动摇了传统的实时应用的开发流程。在典型的web应用里，你把实时特性集成进一个标准的HTML + JS 的界面里。在前面第7章，我们通过船体HTML片段给前端，以及发送JSON数据给客户端。这两种情况下，都是前端收到Channel的消息并基于它的内容来修改界面。

LiveView 改变了这个范式，它用 Elixir 代码来定义应用的用户界面。通过从服务端向客户端发送内容差异，来让界面自动保持更新。几行JS代码来初始化LiveView，接下来由LiveView来处理DOM的更新。用LiveView，你可以不用写js代码来实现实时web应用。

## Getting Started with LiveView

### LiveView Overview

LiveView基于你已经熟悉的技术： Channel 和 Socket。它向前走了一步，提供了富客户端及服务端渲染引擎，提供了一个全栈的开发体验。

![LiveView](images/chap_13_liveview.png)

流程开始于一个web 请求，先渲染一个静态版本的LiveView，发给前端。然后客户端连接到后端服务器，把静态页面变成实时的LiveView。

当前端连接到后端的LiveView Socket，后端启动新的LiveView进程，后端进程基于当前状态渲染一个 Live EEx模板。当LiveView进程的状态改变时，LiveView将发送一个包含改变的最小的payload到前端。然后前端实时的显示正确的HTML。前端界面只要LiveView连接着，就会保持到最新的状态。前端也会发送事件（点击或输入等）到后端LiveView。

Live EEx 模板，就是普通的 EEx模板带了一个特殊的引擎，来高效的跟踪改变。LiveView本身实现了 Phoenix.Channel behaviour。

### 一个快速的 LiveView 例子

```elixir
defmodule LiveViewDemoWeb.CounterLive do
  use Phoenix.LiveView

  def render(assigns) do
    ~L"""
    Current count: <%= @count %>
    <button phx-click="dec">-</button>
    <button phx-click="inc">+</button>
    """
  end

  def mount(%{count: initial}, socket) do
    {:ok, assign(socket, :count, initial)}
  end

  def handle_event("dec", _value, socket) do
    {:noreply, update(socket, :count, &(&1 - 1))}
  end

  def handle_event("inc", _value, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end
end
```

（这个例子跟十几年前看的Seaside的例子好像啊，都是服务端执行逻辑，进行渲染。底层实现机制不同，但本质目的一样.对比一下上下两端代码多么像啊）

[Seaside](http://seaside.st)

[Seaside Counter Demo](http://www.seaside.st/about/examples/counter)

```smalltalk
initialize
    super initialize.
    count := 0

renderContentOn: html
    html heading: count.
    html anchor 
        callback: [ count := count + 1 ];
        with: '++'.
    html space.
    html anchor
        callback: [ count := count - 1 ];
        with: '--'
```

![LiveView](images/chap_13_counter.png)

可以看到WebSocket里的通讯，在点击按钮后向服务端发送事件，服务端发回 {diff: {"0": "87088"}}

我们定义了一个模板，初始状态，以及点击事件的处理函数。代码看上去像是GenServer，因为它就是一个GenServer！通过observer界面观察一下。

