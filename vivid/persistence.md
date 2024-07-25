---
title: 状态持久化
description: 为 Actors 创建记忆点，在启动时恢复
published: true
date: 2024-07-25T16:27:09.041Z
tags: actor, vivid, actor system, persistence
editor: markdown
dateCreated: 2024-07-25T16:27:09.041Z
---

# 介绍
Vivid Persistence 提供了对于 Actor 状态持久化的实现，以便在 Actor 重新启动（例如事故崩溃后）、由监管者或手动停止启动时恢复状态。

它被背后的实现是基于事件溯源及快照的方式进行存储的，当我们的某个消息导致状态变化时，将其记录下来，并在到达一定数量后对快照进行持久化，以此在启动时对快照进行恢复后对事件进行回放来恢复 Actor 的状态。

虽然持久化会使得编码变得更为复杂，但是带来的好处是值得的。它在游戏更新、状态迁移等场景均能表现出众！

# 速览

话不多说，让我们从一个例子开始了解状态持久化：

```go
package main

import (
	"errors"
	"fmt"
	"github.com/kercylan98/minotaur/engine/vivid"
)

type CounterChangeEvent struct {
	Delta int
}

type CounterSnapshot struct {
	Count int
}

type MyActor struct {
	counter *CounterSnapshot
}

func (a *MyActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case *vivid.OnLaunch:
		// 初始化状态
		if a.counter == nil {
			a.counter = &CounterSnapshot{}
		}
		fmt.Println("启动完成", a.counter.Count)
	case *CounterSnapshot:
		a.counter = m
	case *CounterChangeEvent:
		a.counter.Count += m.Delta // 状态改变
		ctx.StateChanged(m)
	case int:
		// 模拟业务逻辑校验
		if m == 0 {
			return
		}
		// 校验通过，状态改变
		ctx.StateChangeEventApply(&CounterChangeEvent{Delta: m})
	case *vivid.OnPersistenceSnapshot:
		ctx.SaveSnapshot(a.counter)
	case error:
		ctx.ReportAbnormal(m) // 模拟崩溃
	}
}

func main() {
	system := vivid.NewActorSystem()
	ref := system.ActorOfF(func() vivid.Actor {
		return new(MyActor)
	})

	for i := 0; i < 10000; i++ {
		system.Tell(ref, i)
	}
	system.Tell(ref, errors.New("panic"))

	system.Shutdown(true)
}

```

待续...
