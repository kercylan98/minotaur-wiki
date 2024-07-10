---
title: Socket 服务器
description: 创建基于 gnet 的包含 ActorSystem 的 Socket 服务器
published: true
date: 2024-07-10T09:07:49.780Z
tags: actor, gnet, tcp, udp, unix domain, actor system
editor: markdown
dateCreated: 2024-07-10T09:07:49.780Z
---

# 介绍

如果您阅读过 [WebSocket 服务器](/guide/websocket-server) 这篇文章，那么会发现。创建 Socket 服务器的过程与基于 gnet 的 WebSocket 服务器创建过程几乎一致，只是我们是创建不同类型的网络而已。

# 创建并运行服务器

这里就不再赘述，直接上代码：

```go
package main

import (
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
)

type SocketService struct {
}

func (s *SocketService) OnInit(kit *transport.GNETKit) {
	kit.ConnectionPacketHook(s.onConnectionPacket)
}

func (s *SocketService) onConnectionPacket(kit *transport.GNETKit, conn *transport.Conn, packet transport.Packet) error {
	conn.WritePacket(packet) // echo
	return nil
}

func main() {
	system := vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(
			transport.NewTCP(":8000").BindService(new(SocketService)),
			transport.NewTCP4(":8001").BindService(new(SocketService)),
			transport.NewTCP6(":8002").BindService(new(SocketService)),
			transport.NewUDP(":8003").BindService(new(SocketService)),
			transport.NewUDP4(":8004").BindService(new(SocketService)),
			transport.NewUDP6(":8005").BindService(new(SocketService)),
			transport.NewWebSocket(":8006", "/ws").BindService(new(SocketService)),
			transport.NewUnix("./unix.sock").BindService(new(SocketService)),
		)
	})
}

```