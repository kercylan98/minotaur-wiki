---
title: 让我们与 Actors 共舞
description: 从示例到深入，掌握 Actors 的使用
published: true
date: 2024-07-25T03:32:18.798Z
tags: actor, vivid, actor system
editor: markdown
dateCreated: 2024-07-11T09:55:53.912Z
---

# Vivid Actor
[Actor 模型](https://zh.wikipedia.org/wiki/%E6%BC%94%E5%91%98%E6%A8%A1%E5%9E%8B)为编写并发和分布式系统提供了更高级别的抽象。它减轻了开发人员处理显式锁定和线程管理的负担，使编写正确的并发和并行系统变得更加容易。

我们在之前聊过，Actor 的通讯是通过向邮箱投递消息来完成的，这似乎会有点抽象，我们来用一个例子说明一下！

依旧是第一步，我们创建一个系统，并用作接下来的展示：

```go
package main

import "github.com/kercylan98/minotaur/engine/vivid"

func main() {
	system := vivid.NewActorSystem()
	defer system.Shutdown(true)
}
```

让我们考虑一下，假设我们的 Actor 会在收到打招呼时会进行回复，并且它拥有自己的名称：

```go
type (
	GreetMessage      struct{ Name string }
	GreetMessageReply struct {
		Name    string
		Content string
	}
)

func NewGreetActor(wg *sync.WaitGroup, name string, neighbor ...vivid.ActorRef) *GreetActor {
	return &GreetActor{wg: wg, name: name, neighbor: neighbor}
}

type GreetActor struct {
	wg       *sync.WaitGroup
	name     string
	neighbor []vivid.ActorRef
}

func (g *GreetActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case *vivid.OnLaunch:
		g.onLaunch(ctx)
	case *GreetMessage:
		g.onGreet(ctx, m)
	case *GreetMessageReply:
		g.onGreetReply(ctx, m)
	case *vivid.OnTerminated:
		g.wg.Done()
	}
}

func (g *GreetActor) onLaunch(ctx vivid.ActorContext) {
	m := &GreetMessage{Name: g.name}
	for _, ref := range g.neighbor {
		ctx.Ask(ref, m)
	}
}

func (g *GreetActor) onGreet(ctx vivid.ActorContext, m *GreetMessage) {
	fmt.Println(fmt.Sprintf("%s 收到了来自 %s 的招呼", g.name, m.Name))
	ctx.Reply(&GreetMessageReply{Name: g.name, Content: fmt.Sprintf("Hi %s, I'm %s", m.Name, g.name)})
  
  ctx.Terminate(ctx.Ref(), true)
}

func (g *GreetActor) onGreetReply(ctx vivid.ActorContext, m *GreetMessageReply) {
	fmt.Println(fmt.Sprintf("%s 收到了来自 %s 的回复：%s", g.name, m.Name, m.Content))
  
  ctx.Terminate(ctx.Ref(), true)
}
```

这段代码定义了两种消息类型，一种用于命令 Actor 问候某人，另一种用于 Actor 确认已问候，并且它们在收到打招呼或回应时均会打印。

我们的 Actor 设计上，它可以拥有邻居，如果拥有，那么在启动时便会向邻居打招呼！这里的打招呼是使用 `Ask` 方法处理的，它是异步非阻塞的，并且当收到回复时将会作为消息一样发送过来，所以我们在消息类型里断言了 `*GreetMessageReply` 类型。另外我们还在 `onGreet` 方法内执行了一个 `Reply` 操作，这个函数是用于向发送人进行回复的，它是异步非阻塞的，并且是有可能会失败的，例如在发送人采用 `Tell` 方式时，将无法进行回复。

另外，我们还在 `onGreet` 和 `onGreetReply` 方法里调用了 `ctx.Terminate(ctx.Ref(), true)`，这是我们期望 Actor 在处理完打招呼或收到打招呼回应时就结束运行。

接下来，我们将创建两个 Actor，并让它们尝试我们的设想，继续编辑 main 函数：

```go
func main() {
	wg := new(sync.WaitGroup)
	wg.Add(2)
	system := vivid.NewActorSystem()
	defer system.Shutdown(true)

	actor1 := system.ActorOfF(func() vivid.Actor {
		return NewGreetActor(wg, "王老五")
	})

	system.ActorOfF(func() vivid.Actor {
		return NewGreetActor(wg, "李老四", actor1)
	})

	wg.Wait()
}
```

关于这里的 `sync.WaitGroup` 只是用来确保执行完成，因为 `Ask` 消息是不会等待收到消息的，如果消息还未进入邮箱，那么优雅停机是不会等待的。

接下来，运行便可以看到屏幕输入了以下内容：
```shell
王老五 收到了来自 李老四 的招呼
李老四 收到了来自 王老五 的回复：Hi 李老四, I'm 王老五
```

> 一个更为复杂的例子：[聊天室示例](/zh/guide/chat-room)
{.is-info}

# 风格化
在上一小节中，我们的 Actor 是通过结构体声明的方式创建的，而在 Vivid 中还提供了多种方式来生成 Actor，它不限于是函数式或者面向对象的方式，这取决于您的趋向，更多时候，对于功能复杂的 Actor 建议使用结构体定义，这样可以在代码上更为清晰！

首先我们需要先介绍 `ActorOf` 函数，它的函数签名如下：
```go
func (sys *ActorSystem) ActorOf(provider ActorProvider, configurator ...ActorDescriptorConfigurator) ActorRef
```

可以看到，它接收 `ActorProvider` 作为必填参数，同时接受 `ActorDescriptorConfigurator` 作为可选参数。
> 多个 `ActorDescriptorConfigurator` 的传入是会向前覆盖的，它们均会被执行。
{.is-info}

## ActorProvider
ActorProvider 是 Actor 示例的提供者，它是一个接口类型，接口定义如下：

```go
type ActorProvider interface {
	GetActorProviderName() ActorProviderName
  
	Provide() Actor
}
```

当实现这个接口时，便可以作为 Actor 的提供者，它用来实例化一个 Actor 对象，并且在重启时也会通过它进行状态的重置。

而 `GetActorProviderName` 函数是做什么的呢？首先它返回的 `ActorProviderName` 是 `string` 类型的别名，它被用在集群中，如果不考虑集群，可以简单的返回 `""` 字符串即可。

是不是感觉比较麻烦？创建一个对象还要做如此复杂的实现！对此，我们还提供了一个函数式的提供者 `vivid.FunctionalActorProvider`，它可以方便地进行 Actor 的实例化，例如：

```
system.ActorOf(vivid.FunctionalActorProvider(func() vivid.Actor {
	return nil
}))
```

它不仅可以方便地实现 Actor 的实例化，并且也对自动补全友好！

![actor-funcational-provider.gif](/actor-system/actor-funcational-provider.gif)

> 如果您有幸看到这句，您便会知道：在 Minotaur 中，大多数功能都包含了 FunctionalXXX 的设计，它将允许开发者以更高的自由度进行自己的风格实践。
{.is-info}



