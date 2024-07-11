---
title: ActorContext
description: 万物皆是 Actor
published: true
date: 2024-07-11T10:08:05.267Z
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

# 处理消息
... 未完待续