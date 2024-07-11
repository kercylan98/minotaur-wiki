---
title: WebSocket 服务器
description: 快速搭建基于 gnet 或 Fiber 的包含 ActorSystem 的 WebSocket 服务器
published: true
date: 2024-07-11T06:52:39.733Z
tags: actor, fiber, gnet, websocket, actor system
editor: markdown
dateCreated: 2024-07-10T08:59:10.905Z
---

# 介绍
在 Minotaur 中目前包含了两种搭建 WebSocket 服务器的方式，分别是基于 [Fiber](https://gofiber.io) 和 [gnet](https://gnet.host) 实现的 WebSocket 服务器。

当需要考虑与 HTTP 结合时，推荐使用基于 Fiber 的实现，而当需要更纯粹的 Socket 体验，那么推荐使用基于 gnet 的实现。

> 目前建议使用基于 Fiber 的 WebSocket 实现。
{.is-warning}

接下来，让我们开始搭建一个基于 WebSocket 的回响服务器！

# 定义 Service
如果您阅读了上一篇搭建 [HTTP 服务器](/zh/guide/http-server) 的文档，那么一切都会十分熟悉，

## 基于 Fiber
首先我们通过实现 tranposrt.FiberService 接口来接收 transport.FiberKit 的结构实例：

```go
import (
	"github.com/kercylan98/minotaur/core/transport"
)

type WebSocketService struct {
}

func (f *WebSocketService) OnInit(kit *transport.FiberKit) {

}
```

## 基于 gnet
与基于 Fiber 的方式相同，只是我们将实现另一个不同的接口：

```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
)

type WebSocketService struct {
}

func (s *WebSocketService) OnInit(kit *transport.GNETKit) {
	
}
```

# 设置数据包 Hook 并写入回响数据
通过 ConnectionPacketHook 函数设置一个 Hook 用于截取到连接 Actor 所收到的数据包并进行处理：

## 基于 Fiber
```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
)

type WebSocketService struct {
}

func (f *WebSocketService) OnInit(kit *transport.FiberKit) {
	kit.WebSocket("/ws").ConnectionPacketHook(f.onConnectionPacket)
}

func (f *WebSocketService) onConnectionPacket(kit *transport.FiberKit, ctx *transport.FiberContext, conn *transport.Conn, packet transport.Packet) error {
	conn.WritePacket(packet) // echo
	return nil
}
```

## 基于 gnet
```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
)

type WebSocketService struct {
}

func (s *WebSocketService) OnInit(kit *transport.GNETKit) {
	kit.ConnectionPacketHook(s.onConnectionPacket)
}

func (s *WebSocketService) onConnectionPacket(kit *transport.GNETKit, conn *transport.Conn, packet transport.Packet) error {
	conn.WritePacket(packet) // echo
	return nil
}
```

# 启动 WebSocket 服务器
最后我们通过与 HTTP 服务器类似的方式，启动我们的服务器即可：

## 基于 Fiber
```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
)

type WebSocketService struct {
}

func (f *WebSocketService) OnInit(kit *transport.FiberKit) {
	kit.WebSocket("/ws").ConnectionPacketHook(f.onConnectionPacket)
}

func (f *WebSocketService) onConnectionPacket(kit *transport.FiberKit, ctx *transport.FiberContext, conn *transport.Conn, packet transport.Packet) error {
	conn.WritePacket(packet) // echo
	return nil
}

func main() {
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(transport.NewFiber(":8877").BindService(new(WebSocketService)))
	})
}
```

## 基于 gnet
```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
)

type WebSocketService struct {
}

func (s *WebSocketService) OnInit(kit *transport.GNETKit) {
	kit.ConnectionPacketHook(s.onConnectionPacket)
}

func (s *WebSocketService) onConnectionPacket(kit *transport.GNETKit, conn *transport.Conn, packet transport.Packet) error {
	conn.WritePacket(packet) // echo
	return nil
}

func main() {
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(transport.NewWebSocket(":8877", "/ws").BindService(new(WebSocketService)))
	})
}
```

就是如此简单！

***

# 扩展阅读
如果你希望更深入的了解相关信息，可以查阅一下内容

- [🎈 暂无 *敬请期待。*](#)
{.links-list}