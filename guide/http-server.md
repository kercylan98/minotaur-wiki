---
title: HTTP 服务器
description: 快速搭建基于 Fiber 的包含 ActorSystem 的 HTTP 服务器
published: true
date: 2024-07-10T08:47:32.680Z
tags: actorsystem, actor, http, fiber
editor: markdown
dateCreated: 2024-07-10T08:31:35.795Z
---

# 介绍
在 Minotaur 中内置了基于 [Fiber](https://gofiber.io) 设计的 HTTP 服务器模块，我们仅需要创建一个 FiberService 便可以得到一个支持 ActorSystem 的服务器。

# 定义 FiberService
首先我们通过实现 tranposrt.FiberService 接口来接收 transport.FiberKit 的结构实例：

```go
package main

import (
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/core/transport"
)

type PingService struct {
}

func (f *PingService) OnInit(kit *transport.FiberKit) {

}
```

# 设置 /ping 路由
在 FiberService 的 OnInit 函数中可以获得我们需要的 FiberKit 实例，FiberKit 实例包含几个函数：

- App() *fiber.App
- System() *vivid.ActorSystem
- WebSocket(path string, rulePath ...string) *transport.FiberWebSocket

我们通过 App 函数便可得到原始的 Fiber 实例。其中还包括了其他的例如用于创建 WebSocket 服务的辅助结构，以及获取到 ActorSystem 的 System 函数。

接下来我们定义一个函数用来处理 ping 请求，并将其注册。这个函数将会响应一个 pong 字符串：

```go
package main

import (
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/core/transport"
)

type PingService struct {
}

func (f *PingService) OnInit(kit *transport.FiberKit) {
	kit.App().Get("/ping", f.onPing)
}

func (f *PingService) onPing(ctx *fiber.Ctx) error {
	_, err := ctx.WriteString("pong")
	return err
}
```

# 启动 HTTP 服务器
最后，我们通过 ActorSystem 来启动 HTTP 服务器：

```go
package main

import (
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/core/transport"
	"github.com/kercylan98/minotaur/core/vivid"
)

type PingService struct {
}

func (f *PingService) OnInit(kit *transport.FiberKit) {
	kit.App().Get("/ping", f.onPing)
}

func (f *PingService) onPing(ctx *fiber.Ctx) error {
	_, err := ctx.WriteString("pong")
	return err
}

func main() {
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(transport.NewFiber(":8877").BindService(new(PingService)))
	})
}
```

这样，我们的 HTTP 服务器就完成了。
