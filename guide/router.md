---
title: 客户端消息路由
description: 通过 router.Multistage 路由客户端消息
published: true
date: 2024-07-10T09:43:42.733Z
tags: router, message, server, proto
editor: markdown
dateCreated: 2024-07-10T09:43:42.733Z
---

# 介绍
在 Socket 开发过程中，一定会面临消息路由的问题，最简单的方案便是通过 switch 来 case 每一个消息号。在 Minotaur 中，提供了 router.Multistage 来对消息路由进行处理。

> Multistage 是一个支持多级路由的路由器，它并不是开箱即用，而是需要对处理函数进行定义后才可使用。
{.is-info}

# 快速开始
通常，Multistage 会被用在数据包处理的函数中，我们需要通过解析数据包得到消息号及相关数据后再通过路由器来匹配到对应的处理器。

首先，我们直接看完整代码，然后我们再一步步解释：

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
	"github.com/kercylan98/minotaur/toolkit"
	"github.com/kercylan98/minotaur/toolkit/router"
)

type Reader func(dst any)

type Message struct {
	Code int
	Data []byte
}

type WebSocketService struct {
	router *router.Multistage[func(conn *transport.Conn, reader Reader)]
}

func (f *WebSocketService) OnInit(kit *transport.FiberKit) {
	f.router = router.NewMultistage[func(conn *transport.Conn, reader Reader)]()

	kit.WebSocket("/ws").ConnectionPacketHook(f.onConnectionPacket)

	f.router.Route(1, f.onPrint)
	f.router.Sub(1).Route(1, f.onPrint)
	f.router.Register(1, 2).Bind(f.onPrint)
}

func (f *WebSocketService) onConnectionPacket(kit *transport.FiberKit, ctx *transport.FiberContext, conn *transport.Conn, packet transport.Packet) error {
	var msg Message
	toolkit.UnmarshalJSON(packet.GetBytes(), &msg)

	handler := f.router.Match(msg.Code)
	if handler != nil {
		handler(conn, func(dst any) {
			toolkit.UnmarshalJSON(msg.Data, dst)
		})
	}

	return nil
}

func (f *WebSocketService) onPrint(conn *transport.Conn, reader Reader) {
	var str string
	reader(&str)
	fmt.Println(str)
}

func main() {
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(transport.NewFiber(":8877").BindService(new(WebSocketService)))
	})
}
```

## 定义消息处理函数类型
从代码中我们可以看到，路由器是接收一个消息处理函数类型的泛型结构。在这个示例中，我们期待的处理函数是这样的：
```go
func(conn *transport.Conn, reader Reader)
```
这是一个非常简单的处理函数，将直接传入 transport.Conn 和一个 Reader。

> 一般情况下，处理函数可能还需要校验获取到 Player 或 User 等结构后再将其传入。
{.is-info}

## 创建路由器并注册消息
在 OnInit 函数里，有这样一段代码：
```go
f.router = router.NewMultistage[func(conn *transport.Conn, reader Reader)]()

f.router.Route(1, f.onPrint)
f.router.Sub(1).Route(1, f.onPrint)
f.router.Register(1, 2).Bind(f.onPrint)
```

其中第一行我们使用指定的处理函数类型创建了一个路由器，而后续的则是几种不同的注册方式。我们看看几种注册方式的匹配命中情况是怎么样的：


| 路由                                     |      命中          |
|-----------------------------------------|:------------------:|
| f.router.Route(1, f.onPrint)            | router.Match(1)    |
| f.router.Sub(1).Route(1, f.onPrint)     | router.Match(1, 1) |
| f.router.Register(1, 2).Bind(f.onPrint) | router.Match(1, 2) |

## 消息结构与匹配

在代码里，我们还定义了消息类型：

```go
type Message struct {
	Code int
	Data []byte
}
```

而在之后我们又在 onConnectionPacket 函数中解析了原始的数据包，得到了这个 Message 类型：

```go
var msg Message
toolkit.UnmarshalJSON(packet.GetBytes(), &msg)
```

解析得到外层的消息后，我们便可以拿到消息的 Code，通过该 Code 匹配处理函数，当处理函数不存在时，将会得到空指针：

```go
handler := f.router.Match(msg.Code)
```

---

> 消息结构是允许自行定义的，不限制使用任何协议，你可以使用 Protobuf、Json 等。
{.is-info}
