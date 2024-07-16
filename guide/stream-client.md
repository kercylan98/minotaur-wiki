---
title: Stream 客户端
description: 流式传输客户端
published: true
date: 2024-07-16T04:02:34.668Z
tags: websocket, socket, stream, client
editor: markdown
dateCreated: 2024-07-16T03:58:49.462Z
---

# 介绍
通常，我们在开发一些功能时会需要用到不限于 TCP、WebSocket 等的客户端，例如在网关开发、压力测试、机器人模拟等。

Minotaur 中提供了基于 Actor 模型设计的通用 `StreamClient` 结构，它目前包含三个可选配置，用于处理连接的打开、关闭及数据包的处理：

- ConnectionOpenedHandler：当连接打开时
- ConnectionPacketHandler：当收到数据包时
- ConnectionClosedHandler：当连接关闭时

> StreamClient 还将在连接异常断开时根据退避指数算法进行至多 10 次的重连，重连的实现是依赖 Actor 的监管策略实现的，如果需要禁用重连或进行更细节的控制，仅需要指定监管策略即可！
{.is-info}


# 使用
使用方式非常简单，我们仅需要通过 `ActorSystem` 创建它的 Actor 实例即可。

接下来，我们将以 `WebSocket` 为例，创建一个简单的 `StreamClient` 并处理它的三个事件：

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
	"os"
)

func main() {
	system := vivid.NewActorSystem()
	system.ActorOf(func() vivid.Actor {
		return transport.NewStreamClient(&transport.StreamClientWebsocketCore{
			Url: "ws://localhost:8877/ws",
		}, transport.StreamClientConfig{
			ConnectionOpenedHandler: func(ctx vivid.ActorContext) {
				fmt.Println("connection opened")
			},
			ConnectionPacketHandler: func(ctx vivid.ActorContext, packet transport.Packet) {
				fmt.Println("packet received", string(packet.GetBytes()))
			},
			ConnectionClosedHandler: func(ctx vivid.ActorContext, err error) {
				fmt.Println("connection closed", err)
			},
		})
	})

	system.Signal(func(system *vivid.ActorSystem, signal os.Signal) {
		system.ShutdownGracefully()
	})
}
```

# 发送数据
每一个函数中都会传入 `vivid.ActorContext`，这个便是 `StreamClient` Actor 的上下文。我们如果需要写入数据，只需要向其发送 `transport.Packet` 类型的数据包即可！
```go
ConnectionOpenedHandler: func(ctx vivid.ActorContext) {
    fmt.Println("connection opened")
    ctx.Tell(ctx.Ref(), transport.NewPacket([]byte("hello")).SetContext(websocket.BinaryMessage))
},
```

在这个示例里，我们修改了连接打开的处理函数，在连接打开时发送 `"hello"`。

> 为什么还需要 `SetContext(websocket.BinaryMessage)` 这段繁杂的内容呢？这是根据不同的流客户端实现来决定的，在内置的 `WebSocket` 实现中，会根据上下文来决定发送的消息类型是 `Text` 还是 `Binary`。
{.is-info}

# 关闭连接
如果需要关闭连接，可以直接调用上下文的 `Terminate` 或 `TerminateGracefully` 函数即可，客户端的实现是完全兼容 `vivid` 的。

# StreamClientCore
如果需要基于该客户端实现不同的传输协议，仅需要实现 `transport.StreamClientCore` 接口即可，这个接口是这样的：
```go
type StreamClientCore interface {
	// OnConnect 打开连接
	OnConnect() error

	// OnRead 读取数据包
	OnRead() (Packet, error)

	// OnWrite 写入数据包
	OnWrite(Packet) error

	// OnClose 关闭连接
	OnClose() error
}
```

## WebSocket
目前基于 `fasthttp` 实现了 `WebSocket` 的流客户端核心，它包含了几个对外公开的字段：
```go
type StreamClientWebsocketCore struct {
	Url    string
	Dialer *websocket.Dialer // 默认将使用 websocket.DefaultDialer
	Header http.Header
}
```

在使用时，仅需要通过 `&` 进行实例化即可：
```go
wsc := &transport.StreamClientWebsocketCore{
	Url: "ws://localhost:8877/ws",
}
```