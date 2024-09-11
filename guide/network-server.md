---
title: 网络服务器
description: 支持各种 Socket 实现的网络服务器
published: true
date: 2024-09-11T08:50:26.796Z
tags: 
editor: markdown
dateCreated: 2024-07-22T10:44:39.287Z
---

# 介绍

网络传输是应用服务器的基石，负责处理客户端与服务器之间的网络通信。尤其在需要高并发、低延迟的场景中，基于 Socket 的网络服务器能够实现有效的双向通信，并为现代应用提供强大的数据传输能力。

在 Minotaur 中提供了便利的 Socket 支持，它将能够方便快捷地创建基于 Actor 模型的 Socket 连接。

> 由于我们允许任何第三方的网络层实现，所以这并不是开箱即用的，但是这也不是十分困难的~
{.is-info}

# 前言

在 Minotaur 中不限制所使用的网络层实现，在本文中则会以 `fiber` 作为 `WebSocket` 服务器来讲述如何与 Actor 进行结合！

> Minotaur 不提供 HTTP 服务器的支持，但是这并不意味着您不能在 HTTP 服务器中使用 Actor，您仅需要在合适（需要）的时候通过引用进行消息的收发即可。
{.is-warning}

# Socket 工厂

在 Minotaur 中创建基于 Actor 的网络服务器的第一步便是创建一个 `socket.Factory`，它的用途是维护所有 Socket 产生的 Actor，并确保生成 Actor 的过程是线程安全的。

首先，我们会依赖 `vivid.ActorSystem（如果您不理解它是什么，可在稍后的章节中进行了解）`，并通过其创建 `socket.Factory`：

```go
package main

import "github.com/kercylan98/minotaur/engine/vivid"

func main() {
	system := vivid.NewActorSystem()
  factory := socket.NewFactory(system)
}
```

如此，我们便得到了 Socket 工厂，它的定义是这样的：
```go
type Factory interface {
    Produce(actor Actor, writer Writer, closer Closer) Socket
}
```

# 使用 Fiber 创建服务器

由于我们需要使用 `fiber` 实现服务器功能，所以先下载对应的包：
```shell
go get -u github.com/gofiber/fiber/v2
go get -u github.com/gofiber/contrib
```

之后在 `main` 函数中我们继续：

```go
package main

import (
	"github.com/gofiber/contrib/websocket"
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/engine/socket"
	"github.com/kercylan98/minotaur/engine/vivid"
)

func main() {
	system := vivid.NewActorSystem()
	factory := socket.NewFactory(system)
	fiberApp := fiber.New()
  
	// TODO
	
	if err := fiberApp.Listen(":8080"); err != nil {
		panic(err)
	}
}
```

如此，我们便搭建了一个 WebSocket 服务器，但是它并没有任何效果。

# 与 Actor 结合

在开始将 `fiber` 与 `vivid` 相结合之前，我们需要先定义一个我们自己的 Actor，它仅需要实现 `socket.Actor` 接口即可：

```go
type Conn struct {
	
}

func (c *Conn) OnReceive(ctx vivid.ActorContext) {
	
}

func (c *Conn) OnPacket(ctx vivid.ActorContext, socket socket.Socket, packet *socket.Packet) {
	socket.WritePacket(packet)
}

```

可以看到这个结构体我们在 `OnPacket` 函数中额外增加了写入数据包的代码，这是用于在接收到数据包时进行回响写入。

接下来，便是关键时刻，我们将在之前的 TODO 部分编写相关代码，以实现将 WebSocket 连接转换为支持 Actor 的 Socket：

```go
fiberApp.Get("/ws", websocket.New(func(conn *websocket.Conn) {
	actorSocket := factory.Produce(new(Conn), func(packet []byte, ctx any) error {
	return	conn.WriteMessage(websocket.TextMessage, packet)
	}, func() error {
		// 由于 Fiber 的 WebSocket 实现，需要手动发送关闭消息
		if err := conn.WriteMessage(websocket.CloseMessage, nil); err != nil {
			return err
		}
		return conn.Close()
	})

	var (
		typ int
		data []byte
		err error
	)

	defer actorSocket.Close(err)
	for {
		typ, data, err = conn.ReadMessage()
		if err != nil {
			return
		}
		actorSocket.React(data, typ)
	}
}))
```

如此，我们便实现了一个支持 Actor 功能的 Socket 服务器了！

# 完整代码

```go
package main

import (
	"github.com/gofiber/contrib/websocket"
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/engine/socket"
	"github.com/kercylan98/minotaur/engine/vivid"
)

func main() {
	system := vivid.NewActorSystem()
	factory := socket.NewFactory(system)
	fiberApp := fiber.New()
	fiberApp.Get("/ws", websocket.New(func(conn *websocket.Conn) {
		actorSocket := factory.Produce(new(Conn), func(packet []byte, ctx any) error {
			return conn.WriteMessage(websocket.TextMessage, packet)
		}, func() error {
			// 由于 Fiber 的 WebSocket 实现，需要手动发送关闭消息
			if err := conn.WriteMessage(websocket.CloseMessage, nil); err != nil {
				return err
			}
			return conn.Close()
		})

		var (
			typ  int
			data []byte
			err  error
		)

		defer actorSocket.Close(err)
		for {
			typ, data, err = conn.ReadMessage()
			if err != nil {
				return
			}
			actorSocket.React(data, typ)
		}
	}))

	if err := fiberApp.Listen(":8080"); err != nil {
		panic(err)
	}
}

type Conn struct{}

func (c *Conn) OnReceive(ctx vivid.ActorContext) {}

func (c *Conn) OnPacket(ctx vivid.ActorContext, socket socket.Socket, packet *socket.Packet) {
	socket.WritePacket(packet)
}
```