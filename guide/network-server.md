---
title: 网络服务器
description: 通过对 gnet 及 fiber 的扩展实现网络功能
published: true
date: 2024-07-27T05:24:17.105Z
tags: 
editor: markdown
dateCreated: 2024-07-22T10:44:39.287Z
---

# 自述

目前经过几轮重构及尝试来看，融入 HTTP、WebSocket、TCP、UDP 等网络服务器功能似乎比较困难，不是无法实现，而是没有找到比较优雅的实现方式，它总有或多或少的限制。

在 Minotaur 中，深度融入了 Actor 模型的思想，我们很希望将网络功能也融入进来，之前有尝试通过 Hook 的方式让用户去监听部分事件，但是经过本地模拟的测试发现，这种方式丧失了大量的灵活性。

目前一个新的包 `engine/stream` 正在进行这方面的尝试，敬请期待。

> 如果您有好的想法或建议，也希望可以告知我们。可以通过仓库 issus 或邮件、群聊等方式进行联系，感谢！
{.is-info}

# Stream Package

在 Minotaur 中将会在 stream 包中提供网络相关的内容，目前提供了一些实现，但是应该还有体验更佳的姿势来实现。

在使用 stream 包搭建服务器前，我们或许应该先了解一个方面以便快速上手。

## stream.Actor

在 `stream.NewStream` 函数中，我们会发现返回值是一个名为 `Actor` 的结构体。这也就意味着我们需要将其放入 `ActorSystem` 中进行运行。不过一般来说，如果我们不需要自行实现某种服务器的支持，那么也就无需这么做。

## stream.Configurator

在目前已有的实现中，会发现不管是接下来讲到的 `WebSocket` 服务器还是 `Socket` 服务器，均需要传入 `ActorSystem` 和一个 `Configurator` 作为参数。ActorSystem 不必解释，我们依赖它运行，而 Configurator 则是让我们能够处理 `Stream` 消息的核心，它允许编辑 `Stream` 的配置，而我们也需要通过配置来操作 Stream 的 Actor。

在 Configuration 中提供了 `WithPerformance` 函数，它将允许我们定义 Actor 的行为，就像正常使用 Actor 一样。

需要注意几点：
- 在 Stream Actor 中，我们可以处理 `*stream.Packet` 类型的消息，它将表示接收到的数据；
- 单独地向 Writer 或 Stream 发送 vivid.OnTerminate 消息都将导致连接的断开，它们是等效的；

> 当我们向 Stream Actor 或者其 Writer 发送类似于 `*vivid.OnTerminate` 这样的生命周期消息时，它将是生效的，如果关闭 Stream Actor，它会顺便销毁 Writer，而如果关闭 Writer，这个连接将无法写入数据！
{.is-warning}



# HTTP 服务器

Minotaur 中内置了 `fiber` 作为 HTTP 服务器组件，如果需要开启一个 HTTP 服务器，可参考 `fiber` 官方文档即可。目前暂时没有针对 HTTP 服务器的额外封装。

```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
	app := fiber.New()
	app.Get("/ping", func(ctx *fiber.Ctx) error {
		_, err := ctx.WriteString("pong")
		return err
	})
}
```

# WebSocket 服务器

在 Minotaur 中，WebSocket 的实现依赖了 `fiber`，我们在 `stream` 包中提供了用于开启基于 vivid 的 ActorSystem 实现的 `WebSocket` 服务器路由处理函数。

一个完整的示例如下：

```go 
package main

import (
	"github.com/gofiber/contrib/websocket"
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/engine/stream"
	"github.com/kercylan98/minotaur/engine/vivid"
	"github.com/kercylan98/minotaur/engine/vivid/behavior"
)

func main() {
	system := vivid.NewActorSystem()

	fiberApp := fiber.New()
	fiberApp.Get("/ws", stream.NewFiberWebSocketHandler(fiberApp, system, stream.FunctionalConfigurator(func(c *stream.Configuration) {
		c.WithPerformance(vivid.FunctionalStatefulActorPerformance(
			func() behavior.Performance[vivid.ActorContext] {
				var writer stream.Writer
				return vivid.FunctionalActorPerformance(func(ctx vivid.ActorContext) {
					switch m := ctx.Message().(type) {
					case stream.Writer:
						writer = m
						ctx.Tell(writer, stream.NewPacketDC([]byte("Hello!"), websocket.TextMessage))
					case *stream.Packet:
						ctx.Tell(writer, m) // echo
					}
				})
			}).
			Stateful(),
		)
	})))

	if err := fiberApp.Listen(":8888"); err != nil {
		panic(err)
	}
}

```

# Socket 服务器
在 Minotaur 中，Socket 服务器是基于 `gnet/v2` 实现的，与 WebSocket 类似，我们提供了一个基于 vivid 的 ActorSystem 对 gnet.EventHandler 的实现。

一个完整示例如下：

```go
package main

import (
	"github.com/kercylan98/minotaur/engine/stream"
	"github.com/kercylan98/minotaur/engine/vivid"
	"github.com/kercylan98/minotaur/engine/vivid/behavior"
	"github.com/panjf2000/gnet/v2"
)

func main() {
	system := vivid.NewActorSystem()

	if err := gnet.Run(stream.NewGNETEventHandler(system, stream.FunctionalConfigurator(func(c *stream.Configuration) {
		c.WithPerformance(vivid.FunctionalStatefulActorPerformance(
			func() behavior.Performance[vivid.ActorContext] {
				var writer stream.Writer
				return vivid.FunctionalActorPerformance(func(ctx vivid.ActorContext) {
					switch m := ctx.Message().(type) {
					case stream.Writer:
						writer = m
					case *stream.Packet:
						ctx.Tell(writer, m) // echo
					}
				})
			}).
			Stateful(),
		)
	})), "tcp://127.0.0.1:8080"); err != nil {
		panic(err)
	}
}
```