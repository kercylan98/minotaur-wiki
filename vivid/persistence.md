---
title: 状态持久化
description: 为 Actors 创建记忆点，在启动时恢复
published: true
date: 2024-07-26T11:00:17.757Z
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
		a.counter = &CounterSnapshot{}
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
	case string:
		// 测试当前值
		fmt.Println("当前值", a.counter.Count)
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
	system.Tell(ref, "test")

	system.Shutdown(true)
}
```

这个例子仔细看其实十分简单，如此我们便能够完成 Actor 状态的持久化，接下来我们展开来针对这个例子详细聊聊！

# 持久化
在持久化状态恢复时，是通过快照恢复和事件回放来进行的，它们是作为消息传入进来的，所以我们这里收到 `*CounterChangeEvent` 消息时应用状态的变更，收到 `*CounterSnapshot` 时则使用快照覆盖当前状态。

首先我们需要先了解一下几个关键函数。

## StateChanged 状态已改变

该函数用来定义状态已改变，它接受一个事件作为参数，这个事件将会被持久化，而后在 Actor 启动或重启时进行事件回放。另外，它还将返回一个 `int` 值表示当前的事件数量，以便在特殊场景下主动进行持久化。函数签名如下：

```go
StateChanged(event Message) int
```

> 在执行该函数后，当事件数量达到阈值时，还会发送一条 `*vivid.OnPersistenceSnapshot` 消息以提醒应该执行快照的更新了，我们可以在收到该消息时主动调用 `ActorContext.SaveSnapshot` 函数来保存快照。这个阈值默认值将会是 `1000`，你可以通过创建 `Actor` 时的配置器更改它。
{.is-info}


## StateChangeEventApply 应用状态事件

这个函数的功能很简单，它会在调用的时候将传入的事件作为当前正在处理的消息并调用 `Actor.OnReceive` 函数来进行二次处理。函数签名如下：

```go
StateChangeEventApply(event Message)
```

## SaveSnapshot 保存快照

快照是 Actor 完整的状态，当执行保存快照时，将会丢弃所有的事件以节省空间，而后在状态恢复时会先恢复快照，而后事件回放。函数签名如下：

```go
SaveSnapshot(snapshot Message)
```

***

在了解几个关键函数后，我们回到之前的例子。你可能会想，为什么不直接收到 `int` 时直接调用 `ctx.StateChanged` 函数呢？还要兜一圈来处理。如果直接执行状态改变，由于事件是已经成功的象征，在回放时就会经过一系列不必要的业务逻辑校验，而如果将事件独立出来，那么便可以直接改变状态。

# Storage 存储器

存储器是一个接口类型，它定义了我们的持久化数据应该存储在哪里。假如是远程的数据库中，那么你的 Actor 将可以在任一节点上恢复其状态。

接口定义如下：

```go
type Storage interface {
	Save(name Name, snapshot Snapshot, events []Event) error

	Load(name Name) (snapshot Snapshot, events []Event, err error)

	Clear(name Name) error
}
```

在默认的情况下，Actor 将会使用 `MemoryStorage` 作为默认的持久化存储器，它会将状态存储在整个进程内的一个全局 Map 中。

# 持久化选项

关于状态持久化，我们目前提供了一些可供配置的选项，它可以在创建 `Actor` 时被指定。

## 设置持久化名称

持久化名称是持久化的关键，它类似于数据库的主键。在未指定的情况下，默认会是 Actor 的逻辑地址。在状态恢复时，将会通过持久化名称去匹配对应的持久化数据。

```go
func (d *ActorDescriptor) WithPersistenceName(name persistence.Name) *ActorDescriptor 
```

## 设置持久化事件数量阈值

我们之前有说过，当事件数量到达阈值便会触发快照的持久化，具体的数值便是可以通过该选项来更改的。

```go
func (d *ActorDescriptor) WithPersistenceEventThreshold(threshold int) *ActorDescriptor 
```

> 阈值只是用作推送 `*vivid.OnPersistenceSnapshot` 消息，事件的丢弃则是在 `SaveSnapshot` 后执行的，如果始终不进行快照的持久化，那么内存里的事件将会持续膨胀直至崩溃。
{.is-warning}

## 设置持久化存储器

当我们需要更改默认的存储器时，仅需要提供一个 `persistence.StorageProvider` 接口实现即可，它与其他的提供者类似，同样提供了函数式的定义方式。

```go
func (d *ActorDescriptor) WithPersistenceStorageProvider(provider persistence.StorageProvider) *ActorDescriptor 
```
