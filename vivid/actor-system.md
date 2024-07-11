---
title: Actor System
description: 并行计算核心重量级结构
published: true
date: 2024-07-11T09:56:32.804Z
tags: actor, actor system
editor: markdown
dateCreated: 2024-07-11T09:55:53.912Z
---

# 介绍
在 Minotaur 中，ActorSystem 是一个核心的重量级结构，负责管理所有的 Actor 及其生命周期、消息传递和调度。ActorSystem 提供了一个运行时环境，使得开发者可以专注于 Actor 的业务逻辑，而无需关心底层的并发和通信细节。

目前网络、集群等功能的实现均使用了 Actor 模型进行设计，这样带来的好处是毋庸置疑的，例如在意外发送 panic 时自动重启 Actor 恢复服务，并通过持久化进行状态的恢复、与本地相同的跨网络通讯接口等。

接下来，我们来深入的了解 ActorSystem 的使用！

# 创建 ActorSystem
创建一个 ActorSystem 非常简单，甚至不需要启动它，只需要这样一行代码即可！
```go
vivid.NewActorSystem()
```

如此，ActorSystem 便创建并运行了，它会启动一个名为 `user` 的 Actor 来作为根 Actor，根 Actor 将会监管所有 Actor 的异常情况并重启它们。 

为了方便使用，ActorSystem 提供了创建 Actor 的函数 `ActorOf`、`KindOf`，以及消息相关的 `Tell`、`Ask`、`FutureAsk`、`Broadcast`、`AwaitForward` 函数，这几个函数实际上也是调用根 Actor 对应的函数来进行处理的，所以这里便不再讲述。可以在 Actor 部分的文章中阅读。 

> 在不需要使用的时候确保通过 `ShutdownGracefully` 或 `Shutdown` 函数来停止 ActorSystem 的运行，以确保每一个 Actor 得到销毁并正常处理退出逻辑，例如持久化等！
{.is-info}

# 停止 ActorSystem
停止 ActorSystem 是有必要的，这样可以确保所有 Actor 的生命周期正常结束，根据不同的场景，我们提供了两种方式来停止 ActorSystem 的运行。

```go
system.Shutdown()
system.ShutdownGracefully()
```

| **函数名**              | 描述                                                                                 |
|------------------------|-------------------------------------------------------------------------------------|
| **Shutdown**           | 向所有 Actor 递归的发送系统级终止消息，Actor 将立即进入销毁阶段                               |
| **ShutdownGracefully** | 向所有 Actor 递归的发送用户级终止消息，Actor 将等待终止消息前的消息处理完毕后进入销毁阶段          |

这两个函数都是阻塞的，它们会等待所有 Actor 被销毁后卸载所有模块并返回。

# 获取 ActorSystem 名称
虽然没什么用，但是我们提供了一个函数，可以用于获取到 ActorSystem 的名称，也许你会用它作为日志信息等。
```go
system.Name()
```

# 死信队列 DeadLetter（不健全的）
在 Vivid 中，Actor 是通过消息驱动的，我们有时会发生将消息投递到不存在的 Actor 的邮箱这种情况，这样这条消息便会进入到死信队列中，我们可以通过获取死信队列来处理这些消息。

通过 `DeadLetter` 函数，便可以获取到死信队列。
```go
system.DeadLetter()
```

# 类型化 ActorKind

在 Vivid 中，Actor 不仅可以通过 `ActorOf` 函数创建，也可以通过类型化后的 `Kind` 标识符进行创建，`Kind` 本质上就只是一个 `string` 类型，用于表示 Actor 的类型名称。

在通过 `Kind` 创建 Actor 之前，我们需要使用 `RegKind` 函数向 ActorSystem 注册 Actor 的工厂函数：

```go
package main

import "github.com/kercylan98/minotaur/core/vivid"

type Cat struct{}

func (c *Cat) OnReceive(ctx vivid.ActorContext) {}

func main() {
	system := vivid.NewActorSystem()
	system.RegKind("cat", func() vivid.Actor {
		return new(Cat)
	})
}
```

通过这样，我们便可以使用 "cat" 这个名称来创建 Actor 了，它背后的逻辑是将 Actor 的工厂函数与 Kind 进行索引，而后在使用的时候直接调用工厂函数进行创建。这种方式不够灵活，但是在集群的情况下会十分有用，它可以在远程节点上完成 Actor 的创建！

# 可选项
在创建 ActorSystem 的时候，我们可能会想要自定义一些信息，例如 ActorSystem 的名称等。为此，我们为 ActorSystem 的创建函数增加了一个可变参，它接受 `func(options *vivid.ActorSystemOptions)` 类型的参数作为可选项的编辑函数。

> 当传入多个编辑函数时，它们都会生效！
{.is-info}


## WithName 名称
该可选项将设置 ActorSystem 的名称。
```go
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithName("example")
	})
```

## WithModule 加载模块
模块是对 ActorSystem 的扩展，例如集群、网络等都是通过模块来注入的。
```go
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithModule(transport.NewFiber(":8080"))
	})
```

## WithLoggerProvider 日志记录器
ActorSystem 内部使用了 `toolkit/log` 包的日志记录器，你可以通过该可选项指定其使用的日志记录器，只要是 `slog.Logger` 就可以！
```go
	vivid.NewActorSystem(func(options *vivid.ActorSystemOptions) {
		options.WithLoggerProvider(slog.Default)
	})
```