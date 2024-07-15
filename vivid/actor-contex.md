---
title: ActorContext
description: 万物皆是 Actor
published: true
date: 2024-07-15T04:08:08.726Z
tags: actor, actor system, actor context
editor: markdown
dateCreated: 2024-07-11T10:08:05.267Z
---

# 介绍
ActorContext 是 Actor 对象的上下文，也是开发过程中交互最为频繁的结构体。它涵盖了消息的处理、发送等内容。

# 定义 Actor
获取到 ActorContext 的方法只有一种，那便是创建一个 Actor，创建 Actor 我们需要先定义一个 Actor：

```go
type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
    // todo
}
```

如此，Actor 便定义好了，它实现了 `vivid.Actor` 接口：
```go
type Actor interface {
	OnReceive(ctx ActorContext)
}
```

# 运行 Actor
在定义好 Actor 后，我们便需要运行它，我们可以通过 `ActorSystem` 或者 `ActorContext` 的 `ActorOf` 函数来实例化 Actor 并运行：
```go
func main() {
	system := vivid.NewActorSystem()
	system.ActorOf(func() vivid.Actor {
		return &HelloActor{}
	})
}
```

在 Actor 被运行后，便可以通过 `OnReceive` 函数来获取到该 Actor 的上下文了。

## 函数式 Actor
在一些简单的场景下，我们可能不太希望以定义一个结构体这样的复杂方式来创建 Actor，那么在 vivid 中，也提供了另一种函数式的创建方式：
```go
package main

import (
	"github.com/kercylan98/minotaur/core/vivid"
)

func main() {
	system := vivid.NewActorSystem()
	system.ActorOf(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			
		})
	})
}
```

# 处理消息
为了赋予 Actor 能力，我们需要使得其能够处理消息，由于我们实现了 vivid.Actor 接口的 OnReceive 函数，所有消息均会通过该函数进行传入，包括生命周期。

一般来说，我们可以监听生命周期消息来进行一些特殊的处理，或者接收用户消息来进行自定义消息逻辑的处理。

在 vivid 中，推荐通过 `switch` 关键字进行类型断言来处理消息，例如：
```go
type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case string:
		ctx.Reply(fmt.Sprintf("hello, %s", m))
	}
}
```

这样，我们便实现了对 `string` 类型消息的处理。

# 生命周期
在之前，我们提到过 Actor 的声明周期也会作为消息传入，那包含哪些生命周期呢？具体如下：

 生命周期       | 描述                                                              
--------------|-----------------------------------------------------------------
 OnLaunch     | Actor 收到的第一条消息，表明 Actor 已经准备就绪，可以处理消息。该阶段通常被用于初始化 Actor 的运行时状态。 
 OnRestarting | 当 Actor 重启时将会收到该消息，在重启完成后将会收到 OnLaunch 消息。                      
 OnTerminate  | 当收到该消息时表明 Actor 在处理完该消息后将被销毁。该阶段通常被用于释放或持久化 Actor 的运行时状态。       
 OnTerminated | 当 Actor 已被销毁时将会收到该消息，通常用于记录日志、计数等情况。                            

如何监听生命周期呢？其实和处理消息一样，只需要对类型断言是否为生命周期类型即可：

```go
type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case vivid.OnLaunch:
		fmt.Println("Hello world!")
	}
}
```

