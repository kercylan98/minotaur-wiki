---
title: 网络服务
description: 阐述如何通过 Minotaur 搭建网络服务器
published: true
date: 2024-06-20T12:14:33.227Z
tags: 
editor: markdown
dateCreated: 2024-06-19T03:10:28.352Z
---

> 本教程旨在通过创建一个简单的 WebSocket 回响服务器来介绍如何使用 Minotaur 提供的网络功能。
{.is-info}

# 创建 WebSocket 服务器

在 Minotaur 中，内置了常用的网络实现，它们都是基于 ActorSystem 进行设计的，由于组建各个功能的过程是繁杂的，所以我们提供了 `minotaur` 包来快速构建应用程序，接下来我们会使用 `minotaur` 包来构建基于 `WebSocket` 的回响服务器。

首先我们先看看使用姿势是怎么样的：

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur"
	"github.com/kercylan98/minotaur/minotaur/transport/network"
)

func main() {
	minotaur.NewApplication(
		minotaur.WithNetwork(network.WebSocket(":8080")),
	).Launch()
}
```

上述代码中，我们创建了一个 `WebSocket` 服务器并监听 `8080` 端口，它仅支持连接的建立，并不会做其他任何事情。

> 有时我们或许需要监听某个 `url` 来建立连接，`network.WebSocket` 提供了一个可变参数，可以用于指定监听的 `url`，例如：
>```go
>network.WebSocket(":8080", "/ws")
>```
{.is-info}

> `WebSocket` 目前是基于 `gnet` 的 `TCP` 实现构建，暂无法支持与 `HTTP` 共用。
{.is-warning}

# 定义 ServerActor 的 OnReceiveHandler

在[上一节](#%E5%88%9B%E5%BB%BA-websocket-%E6%9C%8D%E5%8A%A1%E5%99%A8)中我们仅仅只是创建了一个服务器，我们还需要能够获取到连接打开的事件，以及收到来自连接数据的消息和写入消息，我们可以通过 `minotaur.Application.Launch` 函数来获取到 `minotaur.applicationActor` 的上下文后来进行这些修改。

接下来我们继续之前的代码，完成回响服务器的实现：

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur"
	"github.com/kercylan98/minotaur/minotaur/transport"
	"github.com/kercylan98/minotaur/minotaur/transport/network"
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

func main() {
	minotaur.NewApplication(
		minotaur.WithNetwork(network.WebSocket(":8080")),
	).Launch(func(app *minotaur.Application, ctx vivid.MessageContext) {
		switch m := ctx.GetMessage().(type) {
		case vivid.OnBoot:
			app.GetServer().SubscribeConnOpenedEvent(app)
		case transport.ServerConnectionOpenedEvent:
			m.Conn.SetPacketHandler(func(ctx vivid.MessageContext, conn transport.Conn, packet transport.ConnectionReactPacketMessage) {
				conn.Write(packet)
			})
		}
	})
}

```

**这样我们就完成了一个简单回响服务器的实现，我们来分析一下：**

在 `Launch` 的回调函数中 `minotaur.Application` 传入的应用程序的实例和 `minotaur.applicationActor` 的 `vivid.MessageContext`。

**为什么会是 `vivid.MessageContext` 呢？**

这是因为这个回调函数相当于是重写了 `Actor.OnReceive` 函数，在 `Actor.OnReceive` 中收到的是本次消息的上下文，而不是 `Actor` 本身的 `vivid.ActorContext`，但是在 Minotaur 中，可以将 `vivid.MessageContext` 视为 `vivid.ActorContext`，且功能覆盖与之相比更全面。

回到正题，我们通过在回调函数中监听了 `Actor` 的生命周期事件，在 `Actor` 启动时订阅了服务器的连接打开事件，并且指定接收事件的订阅人是 `app`，这样，我们就可以在同一个函数中，继续监听 `transport.ServerConnectionOpenedEvent` 消息来拿到打开的连接了。

> 由于 `minotaur.Application` 内部使用 `ctx` 实现了 `vivid.Subscriber` 接口，而 `vivid.Subscriber` 接口实际上就是一个 `Actor` 的引用，即 `vivid.ActorRef`，所以代码中传入 `app` 和 `ctx` 是等效的。
{.is-info}

在最后我们通过 `m.Conn.SetPacketHandler` 函数指定了获取到连接消息后进行写入，这样就完成了回响服务器的实现。

> 示例代码：[https://github.com/kercylan98/minotaur/blob/develop-v2/examples/internal/websocket-each-server/main.go](https://github.com/kercylan98/minotaur/blob/develop-v2/examples/internal/websocket-each-server/main.go)
{.is-success}
