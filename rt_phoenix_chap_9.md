# Real-Time Phoenix 9. Build a Real-Time Shopping Cart

## By: hanzhupeng@gmail.com

本章一步步构造一个购物车，应用上书里讲过的各种概念：Channels, PubSub, Channel state, Javascript, Session state。还要考虑到各种可能的失效情况，如服务器崩溃，客户端断线，浏览器多窗口支持。最后进行验收测试。

## Plan Your Shopping Cart

购物车概念上很简单，就是把商品放入、移除，以及下单付款。我们的购物车要实现实时提醒功能，在商品售罄时能通知到购物者，让他们能尽快选择其他的尺码。

### 需求

- 添加删除多个商品到购物车
- 每种鞋只有一个尺码可以添加
- 当购物车中某个商品售罄时通知到购物者
- 页面刷新时，购物车能保持
- 在多个页面之间，一个购物者只有一个单独的购物车
- 购物者不用购物车无法下单
- 管理员可以看到不同的购物车里有什么商品（下一章）

### 设计应用架构

最重要的特性是商品售罄时及时通知用户，我们用 Phoenix PubSub来通知 Channel，每个Channel将更新数据通知给连接的客户端。

![架构图](images/chap_9_arch.png)

PubSub 有一个动态订阅的功能，进程可以订阅或取消订阅一个topic。一个Channel进程可以监听任意的PubSub topic，即使与它连接到的topic不同也可以。在购物车里添加或删除商品时，我们动态的添加或删除PubSub的订阅。这样一个Channel就不会收到不在购物车中的那些商品的消息了。

我们用 CartChannel 来处理用户的添加删除消息，这个Channel进程的状态里保存当前的购物车内容。

我们的购物车需要在多个页面或者在页面reload的时候能保持，而不会消失。有几种方法，如用数据库，或者用Elixir的进程来保存购物者的购物车。我们的需求没有要求将购物车保持一个很长的时间，因此用简单的方法，存在浏览器的本地存储里。

如果用户开了多个tab，那么就有多个连接和多个Channel。当某个tab的购物车有更新操作时，它相应的Channel就进行广播，相同用户的其他channel收到消息后就更新各自的数据和客户端。

### 设置项目，构建商店购物车Channel的架子

把业务逻辑、数据结构放在功能内核里，与用户交互的部分独立开。分离界面与逻辑，可以曾庆可维护性，两边都可以改变而不需要完全的重写。

我们编写一个 ShoppingCart 数据结构来保存购物车的数据，将这些代码加入 Checkout context。

### 构造一个功能内核

开始编写一个新特性时，首先从最核心的部分开始。Checkout.ShoppingCart 是业务逻辑内核(Model)，包含数据结构（items 列表），添加、删除，以及序列号、反序列化，这里使用 Token来进行验证防止被伪造:

```elixir
defmodule Sneakers23.Checkout.ShoppingCart do
  defstruct items: []

  def new(), do: %__MODULE__{}

  def add_item(cart = %{items: items}, id) when is_integer(id) do
    if id in items do
      {:error, :duplicate_item}
    else
      {:ok, %{cart | items: [id | items]}}
    end
  end

  def remove_item(cart = %{items: items}, id) when is_integer(id) do
    if id in items do
      {:ok, %{cart | items: List.delete(items, id)}}
    else
      {:error, :not_found}
    end
  end

  def item_ids(%{items: items}), do: items

  @base Sneakers23Web.Endpoint
  @salt "shopping cart serialization"
  @max_age 86400 * 7

  def serialize(cart = %__MODULE__{}) do
    {:ok, Phoenix.Token.sign(@base, @salt, cart, max_age: @max_age)}
  end

  def deserialize(serialized) do
    case Phoenix.Token.verify(@base, @salt, serialized, max_age: @max_age)
    do
      {:ok, data} ->
        items = Map.get(data, :items, [])
        {:ok, %__MODULE__{items: items}}

      e = {:error, _reason} ->
        e
    end
  end
end
```

通过Checkout Context模块，将内部的函数代理出来，提供调用接口。

```elixir
defmodule Sneakers23.Checkout do
  alias __MODULE__.{ShoppingCart}

  defdelegate add_item_to_cart(cart, item), to: ShoppingCart, as: :add_item
  defdelegate cart_item_ids(cart), to: ShoppingCart, as: :item_ids
  defdelegate export_cart(cart), to: ShoppingCart, as: :serialize
  defdelegate remove_item_from_cart(cart, item), to: ShoppingCart, as: :remove_item

  def restore_cart(nil), do: ShoppingCart.new()
  def restore_cart(serialized) do
    case ShoppingCart.deserialize(serialized) do
      {:ok, cart} -> cart
      {:error, _} -> restore_cart(nil)
    end
  end
  @cart_id_length 64
  def generate_cart_id() do
    :crypto.strong_rand_bytes(@cart_id_length)
    |> Base.encode64()
    |> binary_part(0, @cart_id_length)
  end
end
```

### 准备HTML

用户打开多个窗口，要都保持同步，每一个窗口是一个不同的Channel 实例，我们要用一种方法把他们都连接起来。最简单的办法是通过Channel topic，用 "cart:<cart_id>" 。我们用一个随机数生成 cart_id 存到cookie中.

我们希望每一个页面上都有cart_id 功能，我们可以通过在Controller里复制粘贴同样的代码块，但是有更简单的方法，就是使用 Plug库，创建下面的 Plug模块，加入到 router里：

```elixir
defmodule Sneakers23Web.CartIdPlug do
  import Plug.Conn
  def init(_), do: []

  def call(conn, _) do
    {:ok, conn, cart_id} = get_cart_id(conn)
    assign(conn, :cart_id, cart_id)
  end

  defp get_cart_id(conn) do
    case get_session(conn, :cart_id) do
      nil ->
        cart_id = Sneakers23.Checkout.generate_cart_id()
        {:ok, put_session(conn, :cart_id, cart_id), cart_id}

      cart_id ->
        {:ok, conn, cart_id}

    end
  end
end
```

在 layout 里加入购物车相关的前端代码，这样每个页面都包含了cart_id .

这样浏览器多个tab里， window.cartId 都是同一个值。

### 构建购物车Channel

在购物车Channel进程里，保存购物车的数据，处理添加，删除商品操作，与客户端同步，实时更新商品库存。（这里把购物车的数据放在Channel进程的状态里，如果有多个页面，就会有多个Channel进程，购物车数据会复制多份。如果用单独的一个GenServer来保存和处理更好，数据不需要重复，也不需要复制）

用 "cart:*" topic 来连接channel。在ProductSocket中添加路由：

```elixir
defmodule Sneakers23Web.ProductSocket do
  use Phoenix.Socket

  ## Channels
  channel "product:*", Sneakers23Web.ProductChannel
  channel "cart:*", Sneakers23Web.ShoppingCartChannel
  
  def connect(_params, socket, _connect_info) do
    {:ok, socket}
  end

  def id(_socket), do: nil
end
```

ShoppingCartChannel

```elixir
defmodule Sneakers23Web.ShoppingCartChannel do
  use Phoenix.Channel

  alias Sneakers23.Checkout
  def join("cart:" <> id, params, socket) when byte_size(id) == 64do
    cart = get_cart(params)
    socket = assign(socket, :cart, cart)
    {:ok, socket}
  end

  defp get_cart(params) do
    params
    |> Map.get("serialized", nil)
    |> Checkout.restore_cart()
  end

end
```

将客户端本地保存到cart数据恢复到Channel的状态中。

把购物车数据里的id列表，变成具体的数据结构，这部分功能与Channel无关，放在单独的View模块里：

```elixir
defmodule Sneakers23Web.CartView do
  def cart_to_map(cart) do
    {:ok, serialized} = Sneakers23.Checkout.export_cart(cart)
    {:ok, products} = Sneakers23.Inventory.get_complete_products()
    item_ids = Sneakers23.Checkout.cart_item_ids(cart)
    items = render_items(products, item_ids)
    %{items: items, serialized: serialized}
  end

  defp render_items(_, []), do: []

  defp render_items(products, item_ids) do
    for product <- products,
        item <- product.items,
        item.id in item_ids do
      render_item(product, item)
    end
    |> Enum.sort_by(& &1.id)
  end

  @product_attrs [
    :brand, :color, :name, :price_usd, :main_image_url, :released
  ]

  @item_attrs [:id, :size, :sku]

  defp render_item(product, item) do
    product_attributes = Map.take(product, @product_attrs)
    item_attributes = Map.take(item, @item_attrs)
    product_attributes
    |> Map.merge(item_attributes)
    |> Map.put(:out_of_stock, item.available_count == 0)
  end
end
```

这段代码根据 item id，把商品重要属性取出来，然后加入当前是否售罄的状态信息。


