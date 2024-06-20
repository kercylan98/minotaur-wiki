---
title: Network Services
description: Explaining how to build a network server using Minotaur
published: true
date: 2024-06-20T16:12:23.839Z
tags: 
editor: markdown
dateCreated: 2024-06-20T16:12:23.839Z
---

> 本教程旨在通过创建一个简单的 WebSocket 回响服务器来介绍如何使用 Minotaur 提供的网络功能。
{.is-info}

> This tutorial aims to introduce how to use Minotaur's networking capabilities by creating a simple WebSocket echo server.
{.is-info}

# Creating a WebSocket Server

In Minotaur, common networking implementations are built on the ActorSystem design. As assembling these functionalities can be complex, we provide the `minotaur` package to rapidly build applications. Next, we'll use the `minotaur` package to construct a WebSocket echo server.

First, let's look at how we approach this:

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

In the [previous section](#creating-a-websocket-server), we've created a basic WebSocket server that listens on port `8080` but currently only establishes connections without performing any additional actions.

> Sometimes, we may need to listen on a specific `url` to establish connections. The `network.WebSocket` function provides a variadic parameter to specify the URL to listen on, for example:
>```go
>network.WebSocket(":8080", "/ws")
>```
{.is-info}

> `WebSocket` is currently built on `gnet`'s TCP implementation and does not support sharing with `HTTP`.
{.is-warning}

## Defining the ServerActor's OnReceiveHandler

In the previous section, we've only created a basic server. Now, we need to handle events for connection opening, receiving data from connections, and writing messages back. We can modify the `minotaur.Application.Launch` function to obtain the context of `minotaur.applicationActor` to make these changes.

Next, let's continue with the previous code and complete the implementation of the echo server:

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

**This completes the implementation of a simple echo server. Let's analyze it:**

In the callback function of `Launch`, `minotaur.Application` passes an instance of the application and `minotaur.applicationActor`'s `vivid.MessageContext`.

**Why `vivid.MessageContext`?**

This is because this callback function effectively overrides the `Actor.OnReceive` function. In `Actor.OnReceive`, the context of the current message is received, not the `Actor` itself's `vivid.ActorContext`. However, in Minotaur, `vivid.MessageContext` can be considered as equivalent to `vivid.ActorContext`, with additional comprehensive functionality.

Returning to the point, we listen for `Actor` lifecycle events in the callback function. When the `Actor` starts, we subscribe to the server's connection open event and specify the subscriber as `app`. This allows us to continue listening for `transport.ServerConnectionOpenedEvent` messages within the same function to retrieve open connections.

> Because `minotaur.Application` internally implements the `vivid.Subscriber` interface using `ctx`, and the `vivid.Subscriber` interface is essentially a reference to an `Actor`, specifically `vivid.ActorRef`, passing `app` and `ctx` in the code is equivalent.
{.is-info}

Finally, we complete the implementation of the echo server by using `m.Conn.SetPacketHandler` to specify writing upon receiving connection messages.

> Example code: [https://github.com/kercylan98/minotaur/blob/develop-v2/examples/internal/websocket-each-server/main.go](https://github.com/kercylan98/minotaur/blob/develop-v2/examples/internal/websocket-each-server/main.go)
{.is-success}
