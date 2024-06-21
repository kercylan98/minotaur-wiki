---
title: Actor 生命周期
description: 了解 Actor 的各阶段生命周期及监听方式
published: true
date: 2024-06-21T03:53:36.579Z
tags: actor, life-cycle
editor: markdown
dateCreated: 2024-06-20T10:11:37.769Z
---

# 简介
`Actor` 模型是一种用于并发计算的编程模型，在分布式系统中尤其常见。在 `Actor` 模型中，`Actor` 是独立的计算实体，它们通过消息传递进行通信。每个 `Actor` 有自己的状态和行为，并且在接收到消息时才会进行计算。理解 `Actor` 的生命周期对于有效地利用这个模型至关重要。

# 生命周期
在 Minotaur 中，如果希望监听生命周期，仅需要在 `OnReceive` 函数中通过类型断言特定的生命周期消息类型即可。例如：

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type MyActor struct{}

func (h *MyActor) OnReceive(ctx vivid.MessageContext) {
	switch ctx.GetMessage().(type) {
	case vivid.OnBoot:

	}
}
```

> 不同的生命周期消息中可能会携带额外的参数。
{.is-info}

## vivid.OnInit
这是 `Actor` 生命周期的第一个阶段，该阶段虽然叫初始化，但是并非为 `Actor` 的初始化，而是对于 `ActorOptions` 的初始化。

> 在此刻，`Actor` 并不可用，通常被用于对于 `ActorOptions` 的检查及控制需求。
{.is-warning}

## vivid.OnBoot
当 `Actor` 进入到该阶段时，表示 `Actor` 已经初始化完成，并已经可以开始处理消息。

`vivid.OnBoot` 消息作为第一个消息发送给 `Actor`，该消息的执行是阻塞的，这意味着如果 `Actor` 在处理该消息完成之前，`ActorOf` 等创建 `Actor` 的函数将会被阻塞。

通常，该阶段我们可以完成对 `Actor` 对象属性的初始化等行为。

## vivid.OnRestart
该阶段并非一定发生，而是在 `Actor` 重启时会紧跟在 `OnBoot` 之后发送，同样的，该消息的执行也是阻塞的。

## vivid.OnActorTyped
这是一个特殊的生命周期，当 `Actor` 通过类型化接口被创建时，将会触发该生命周期，该阶段会紧跟着 `OnBoot` 或 `OnRestart` 之后触发。

> 当 `Actor` 需要获取到自身的类型化接口时，该生命周期十分有用。
{.is-info}

## vivid.OnTerminate
这是 `Actor` 的终止消息，当一个 `Actor` 收到该消息后，应该完成对自身资源的清理工作。

当主动向 `Actor` 发送该消息时，也会造成 `Actor` 的终止，`ActorRef.Terminate()` 函数的原理就是向 `Actor` 发送该消息。

> 当 `Actor` 是由于意外情况关闭时，并且其父级的监管策略将其执行了重启行为时，该 `Actor` 的所有子级 `Actor` 都会跟随着进行重启。而关闭的次序是由子到父，重启的次序则是由父到子。
{.is-info}

> `Actor` 收到该消息后，`ActorSystem` 会将其邮箱关闭，拒绝新消息的进入，并且会递归的向其子 Actor 发送该消息，直到所有子级 Actor 都被终止后，该 `Actor` 将会被释放。
{.is-warning}

