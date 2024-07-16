---
title: ActorContext
description: 万物皆是 Actor
published: true
date: 2024-07-16T14:35:14.671Z
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

# 指定可选项
在创建 Actor 时，我们还可以通过可选项来设置一些额外的特征，仅需要在 `ActorOf` 函数中传入可选项编辑函数即可：
```go
system.ActorOf(func() vivid.Actor {
	return &HelloActor{}
}, func(options *vivid.ActorOptions) {
	// options...
})
```
就这样，将可选项编辑函数作为可变参传入即可。

> 传入多个编辑函数可能会导致靠后的可选项覆盖靠前的可选项，因为它们都会被执行。另外，通常建议通过 `With` 函数来进行编辑，如若直接赋值可能会发生不可预知的问题。
> {.is-warning}

## 指定名称
默认情况下，Actor 的名称将会是一个自增的计数，该计数来自于父 Actor 的子 Actor 生成次数（销毁不会减少），如果需要指定特定名称，那么可以使用该可选项：
```go
func (o *ActorOptions) WithName(name string) *ActorOptions
```

## 名称前缀
在 vivid 中，默认的 Actor 是不携带名称前缀的，如果指定了该可选项，将会包含一段名称前缀，并以 `-` 进行分隔。
```go
func (o *ActorOptions) WithNamePrefix(prefix string) *ActorOptions
```

## 指定父 Actor
在默认情况下，Actor 的父 Actor 将会来自调用 `ActorOf` 的 Actor，该可选项允许将父 Actor 特殊的指定为其他的 Actor。
```go
func (o *ActorOptions) WithParent(parent ActorRef) *ActorOptions 
```

> 对于 ActorSystem 的 `ActorOf` 调用，那么父 Actor 将会是内置的 `root(user) Actor`。
{.is-info}

## 持久化配置
在 vivid 中，Actor 是默认支持基于内存存储的持久化的，在 Actor 被重启时候，将通过快照结合事件回放将 Actor 的状态进行恢复。当我们需要调整存储器或触发快照的限制时，便可通过这两个可选项进行设置：

### 指定持久化存储器和持久化名称
```go
func (o *ActorOptions) WithPersistence(storage Storage, name string) *ActorOptions
```

### 指定触发快照所需事件数
```go
func (o *ActorOptions) WithPersistenceEventLimit(limit int) *ActorOptions 
```

## 设置邮箱
默认情况下，Actor 将会使用内置的 FIFO 事件驱动邮箱作为 Actor 的消息队列，如果需要特殊实现，可使用该可选项：
```go
func (o *ActorOptions) WithMailbox(producer MailboxProducer) *ActorOptions
```

## 设置调度器
默认的 Actor 调度器是基于 `ants` 实现的调度器，如果需要特殊实现，可使用该可选项：
```go
func (o *ActorOptions) WithDispatcher(producer DispatcherProducer) *ActorOptions
```

## 启用定时调度器
默认情况下，Actor 的调度器是不启用的，当 Actor 需要执行定时任务时，可通过该选项来启用调度器：
```go
func (o *ActorOptions) WithScheduler(enable bool) *ActorOptions
```
该调度器背后是基于 `github.com/RussellLuo/timingwheel` 仓库实现的 `chrono.Scheduler` 时间轮调度器，它不仅支持定时任务、循环任务、延迟任务，也支持根据 `cron` 表达式创建定时任务。

> 在为 Actor 指定生命周期或处理消息的生命周期后，也会启用调度器。
{.is-info}


## 指定生命期限
有一些 Actor 我们可能希望它仅运行一段时间，这时候就可以通过该可选项来完成。

该函数接受一个持续时间作为参数，Actor 的运行将在到达该时间后自动停止。
```go
func (o *ActorOptions) WithMessageDeadline(deadline time.Duration) *ActorOptions
```

## 指定消息消费期限
该可选项与 `WithMessageDeadline` 类似，但是它是以消费消息间隔作为期限的，当我们需要 Actor 空闲一段时间后自动销毁，那么便可以使用它。

它将在 Actor 接收消息时移除计时任务，在消费消息结束时创建计时任务，在这段期间如果没有新的消息，那么将销毁 Actor。

```go
func (o *ActorOptions) WithMessageDeadline(deadline time.Duration) *ActorOptions 
```

## 冲突复用
该可选项适用于类似单例 Actor 的场景，它会在 Actor 已存在的时候返回已有 Actor 的引用，例如在同一个父 Actor 下创建两个名称相同的 Actor，如果没有该可选项，将会引发 panic，否则将返回已有 Actor 的引用。

```go
func (o *ActorOptions) WithConflictReuse(enable bool) *ActorOptions
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
 OnTerminate  | 当收到该消息时表明 Actor 在处理完该消息后将被销毁。该阶段通常被用于释放或持久化 Actor 的运行时状态。       
 OnRestarting | 当 Actor 重启时将会收到该消息，在重启完成后将会收到 OnLaunch 消息。                      
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

# 状态持久化
一个 Actor 通常是有状态的，默认情况下，发生异常情况后状态便会通知，通过状态持久化，我们可以在 Actor 启动时进行状态的恢复，并且根据持久化存储器，将不限于从内存、硬盘还是网络进行恢复。

要实现状态的持久化，首先便是要确保 Actor 由事件来驱动状态的改变，除了一些基本类型外，目前需要使用 `protobuf` 来定义 Actor 的状态及事件（因为它们需要被序列化）。

## 事件记录
在 Actor 运行过程中，当我们改变状态后，可以通过 `ctx.StatusChanged` 函数记录该事件，在 Actor 启动时，将通过事件回放来恢复 Actor 的状态。

例如：
```go
system.ActorOf(func() vivid.Actor {
	state := 0
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case bool: // event
			state++
			ctx.StatusChanged(m)
		}
	})
})
```

## 快照持久化
为了避免事件过多造成内存膨胀，在事件数量达到一定数量后将会收到 `vivid.OnPersistenceSnapshot` 消息，我们可以通过监听该消息来调用 `ctx.PersistSnapshot` 函数持久化完整状态的快照，当然，也可以随时调用。

在调用 `PersistSnapshot` 函数后，旧的事件将被清理。在状态恢复时，将会发送一个快照类型消息后进行事件回放。

```go
system.ActorOf(func() vivid.Actor {
	state := &Snapshot{}
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case vivid.OnLaunch:
			ctx.PersistSnapshot(&Snapshot{
				Counter: 100,
			})
		case *Snapshot: // recover snapshot
			state = m
		}
	})
})
```

在这个例子中，我们在启动时持久化了 `Snapshot` 类型的快照，在 Actor 下次启动时，便会收到 `Snapshot` 类型的消息，用于恢复状态。