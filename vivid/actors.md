---
title: 与 Actors 共舞
description: 从示例到深入，掌握 Actors 的使用
published: true
date: 2024-08-03T07:57:21.141Z
tags: actor, vivid, actor system, supervision
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
ActorProvider 是 Actor 实例的提供者，它是一个接口类型，接口定义如下：

```go
type ActorProvider interface {
	Provide() Actor
}
```

当实现这个接口时，便可以作为 Actor 的提供者，它用来实例化一个 Actor 对象，并且在重启时也会通过它进行状态的重置。

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

还有没有更方便快捷的办法？我们还提供了 `ActorOfF` 函数，他将提供更简单的方式来实例化：

```go
system.ActorOfF(func() vivid.Actor {
	return nil
})
```

> 当我们使用函数式方式创建 Actor 时，我们的 Actor 将不再支持集群内创建，因为集群内需要注册名称，声明这个节点能提供什么 Actor！如果需要支持集群的函数式 Actor，可以考虑 `vivid.NewShortcutActorProvider`：
> ```go
> system.ActorOf(vivid.NewShortcutActorProvider("calc", func() vivid.Actor {
> 	return nil
> }))
> ```

## ActorDescriptorConfigurator
ActorDescriptorConfigurator 是 Actor 描述符配置器，它是一个接口类型，接口定义如下：

```go
type ActorDescriptorConfigurator interface {
	Configure(descriptor *ActorDescriptor)
}
```

与 `ActorProvider` 一样，`ActorDescriptorConfigurator` 同样提供了它的函数式接口，这里便不过多赘述：

```go
system.ActorOf(
	vivid.FunctionalActorProvider(func() vivid.Actor {
		return nil
	}), vivid.FunctionalActorDescriptorConfigurator(func(descriptor *vivid.ActorDescriptor) {
		descriptor.WithName("hi")
	}),
)
```

```go
system.ActorOfF(
	func() vivid.Actor {
		return nil
	}, func(descriptor *vivid.ActorDescriptor) {
		descriptor.WithName("hi")
	},
)
```

## 面向对象 Actor
在我们上述的例子中，我们采用的便是面向对象的 Actor 设计，通过将 Actor 定义为一个结构体来进行使用，它通常能让代码逻辑更清晰，并且适合有状态或者功能复杂的 Actor 构建：
```go
type MyActor struct {}

func (m *MyActor) OnReceive(ctx vivid.ActorContext) {
	// none
}
```

## 函数式 Actor
另一种定义 Actor 的方式便是通过函数式接口来定义，在一些简单功能或者无状态的 Actor，为了快速实现，我们便可以通过这样来进行：
```go
vivid.FunctionalActor(func(ctx vivid.ActorContext) {
	// none
})
```

# 生命周期
由于 Actor 是一种有状态的资源，它从启动到结束均会经历多个阶段。

比较特殊的是，子 Actor 的生命周期不会超过其父 Actor，所以当一个 Actor 在销毁或 ActorSystem 销毁时，所有的子 Actor 均会被销毁，并且 Actor 是可以停止自身或者被其它 Actor 停止的。

## 创建 Actor
Actor 可以创建任意数量的子 Actor，而子 Actor 又可以生成自己的子 Actor，从而形成 Actor 层次结构。在这个层次结构中只会存在一个根 Actor，即层次结构的顶部的 Actor。子 Actor 的生命周期与父 Actor 紧密相关。

在创建 Actor 之后，我们会收到 `*vivid.OnLaunch` 消息，表示 Actor 已经开始工作，并且一次的消息处理均会携带贯穿 Actor 整个生命周期的 `ActorContext`。

> 关于 `ActorContext` 我们则需要注意，它并不是一个并发安全的结构，除非你知道你在做什么，否则它不应该传递给外部进行使用。
{.is-warning}

```go
system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch ctx.Message().(type) {
		case *vivid.OnLaunch:
      // do something
		}
	})
})
```

> `*vivid.OnLaunch` 消息是可以通过 `Reply` 函数进行回复的，这样它的父级 Actor 将会收到回复的消息，这在需要监听初始化完成的情况下非常有用！
{.is-info}


这个简短的示例便让我们创建了一个 Actor，并且处理 `*vivid.OnLaunch` 生命周期事件，对于其他的事件，处理方式也是相同的。

## 生成子代 Actor
在我们上述的例子，我们知道生成 Actor 是通过 `ActorOf` 或 `ActorOfF` 函数实现的，那么如果要生成子代仅需要在不同的 `ActorContext` 上调用即可（即便是远程 Actor，依旧如此！）

> 接下来，生成函数均表示 `ActorOf` 或 `ActorOfF` 函数。
{.is-info}


如果我们通过 `ActorSystem` 调用生成函数，实际上调用的也是我们层次结构的顶级根 Actor 的生成函数，也就是除了顶级 Actor，每一个 Actor 都会有父 Actor。

当子代创建时，父 Actor 不会收到任何消息，因为这是已知的。当生成函数返回引用时，子 Actor 便已准备就绪。

## 停止 Actor
我们的 Actor 都可以通过 `ActorContext` 的 `Terminate` 函数来停止自身或其他 Actor，该函数的签名如下：

```go
func (ctx *actorContext) Terminate(target ActorRef, gracefully bool)
```

可以看到，`target` 作为要停止的目标 Actor 的引用，如果是自身，那么便会停止自身。而另一个 `gracefully` 参数则表示了是否优雅的停止，如果是，那么停止消息将会作为用户消息进行发送，等待前方用户消息处理完毕后进行停止，否则将会立即停止。

> 优雅的停止是会向下覆盖的，当子 Actor 被销毁时，会选择继承父 Actor 的停止参数。
{.is-info}

在 Actor 收到停止消息时候，我们可以通过监听 `*vivid.OnTerminate` 消息来进行释放逻辑，例如持久化、记录日志等。

在子 Actor 被停止时，还会向父 Actor 发送 `*vivid.OnTerminated` 消息以告知自身已停止，这个消息会携带其引用，另外，Actor 自身的停止，Actor 本身也会收到该消息。

```go
system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch ctx.Message().(type) {
		case *vivid.OnLaunch:
			ctx.Terminate(ctx.Ref(), true)
		}
	})
})
```

## 观察 Actor
当我们需要观察一个非自己子 Actor 是否已结束的状态时，便可以通过 ActorContext 的 `Watch` 函数进行观测，它会在目标停止后向观察者推送 `*vivid.OnTerminated` 消息。当然，也可以通过 `UnWatch` 函数取消观察。

```go
func main() {
	wait := new(sync.WaitGroup)
	wait.Add(1)
	system := vivid.NewActorSystem()
	refA := system.ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			// none
		})
	})

	system.ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			switch m := ctx.Message().(type) {
			case *vivid.OnLaunch:
				ctx.Watch(refA)
			case *vivid.OnTerminated:
				if m.TerminatedActor.Equal(refA) {
					wait.Done()
				}
			}
		})
	})

	system.Terminate(refA, false)

	wait.Wait()
	system.Shutdown(true)
}
```

## 重启 Actor
在 Actor 处理消息发送事故（panic）或 Actor 自身报告事故时，将会根据其监管者的定义决定是否重启，当 Actor 重启时，将会停止自身所有的子 Actor，并依次行走 `*vivid.OnRestarting`、`*vivid.OnTerminate`、`*vivid.OnTerminated`、`*vivid.OnRestarted`、`*vivid.OnLaunch` 多个生命周期，并在 `*vivid.OnLaunch` 收到时确保状态已重置。

> 如果存在持久化策略，那么重启后将会对其状态进行恢复。
{.is-info}

# 消息传递
了解完 Actor 的基本使用及生命周期后，让我们来详细聊聊消息的传递过程。

在 Vivid 中，消息的传递是依赖于消息类型的，而不再是函数的调用，也就是所谓的协议。我们需要确保消息类型的正确，否则这条消息将会石沉大海。

消息类型的定义很简单，如果只是简单的单机应用，任何类型都可以作为消息进行传递，消息的类型本质上是 `any` 类型，但是如果是需要跨网络通讯，那么需要确保消息是能够被序列化以及反序列化的。在 vivid 的内部，远程消息是依赖于 `prc` 进行实现的，默认的编解码器使用的是 `ProtoBuffer` 编解码器，如果需要其他编解码器，可自行实现。

Actor 的消息交换有几种模式，我们首先定义一个仅用于示例的消息结构，然后逐一讲解：

```go
type ExampleMessage struct {
	Content string
}
```

## Tell 仅告知
在该模式下，我们的消息非阻塞地被发送到目标 Actor，并且该消息无法被回复。

Actor 的协议及行为定义：
```go
receiver := system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case *ExampleMessage:
			fmt.Println(m.Content)
		}
	})
})
```

发送消息：
```go
system.Tell(receiver, &ExampleMessage{
	Content: "Hi",
})
```

## Ask 可寻址的告知
在该模式下，我们的消息同样是非阻塞地被发送到目标 Actor，但是不同的是，该消息是可以被回复的，并且回复的消息也是可以循环回复的，也就是交谈！

另外，有用的是，消息是可以多次回复的，消息的回复实际上与 `Ask(ctx.Sender(), "xxx")` 是等效的。

接收者 Actor 的协议及行为定义：
```go
receiver := system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case *ExampleMessage:
			switch m.Content {
			case "你好":
				ctx.Reply(&ExampleMessage{Content: "你好"})
			case "能帮我一下吗？":
				ctx.Reply(&ExampleMessage{Content: "不能，谢谢！"})
			}
		}
	})
})
```

发送者 Actor 的协议及行为定义：
```go
system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case *vivid.OnLaunch:
			ctx.Ask(receiver, &ExampleMessage{Content: "你好"})
		case *ExampleMessage:
			switch m.Content {
			case "你好":
				ctx.Reply(&ExampleMessage{Content: "能帮我一下吗？"})
			}
		}
	})
})
```

在这个示例下，发送者在创建后便向接收者发送第一条消息，之后它们来回交流，直到发送者被拒绝。

## FutureAsk 阻塞式的请求响应
在该模式下发送消息将会返回一个 `future.Future` 对象，它可以被用来阻塞的等待消息。

Actor 的协议及行为定义：
```go
receiver := system.ActorOfF(func() vivid.Actor {
	return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
		switch m := ctx.Message().(type) {
		case *ExampleMessage:
			ctx.Reply(m)
		}
	})
})
```

发送消息：
```go
future := system.FutureAsk(receiver, &ExampleMessage{
	Content: "Hi",
})

if _, err := future.Result(); err != nil {
	panic(err)
}
```

在 `FutureAsk` 函数中，还包含了一个可变参数，它将允许设置超时事件，如果这个值 <= 0，那么这个等待将会是无限期的，直至消息的到来。
> 默认的超时时间为 1 秒
{.is-info}

## Broadcast 非阻塞的 Ask
正如标题所属，`Broadcast` 是对所有子 Actor 发起 `Ask` 请求，没有其他的特殊内容。

# 监管
在 Vivid 中，默认所有 Actor 均会受到顶级 Actor 的监管，我们将它叫做 `GuardActor`。

监督允许您以声明方式描述当 Actor 内部抛出某些类型的异常时应该发生什么。

默认的监督策略是，如果抛出异常，则会根据指数退避算法进行至多 10 次的重启。在许多情况下，您需要进一步自定义此行为。

在我们为 Actor 指定监管策略后，将会由它的父 Actor 来执行，假若父 Actor 无法执行，那么将继续向上升级，直至顶级 Actor。

> 默认的监管策略目前是非确定性的，这可能在经过测试或反馈后进行调整。
{.is-warning}

## 指定监管策略

当我们需要自定义监管行为时，我们可以通过 `vivid.ActorOf` 函数的可变参数来定义可选项，在 `*vivid.ActorDescriptor` 中可通过 `WithSupervisionStrategyProvider` 来指定监管策略提供者。它接收一个 `supervision.StrategyProvider` 作为参数，并且接受一个可变参数作为日志记录选项。函数签名如下：

```go
func (d *ActorDescriptor) WithSupervisionStrategyProvider(provider supervision.StrategyProvider, loggers ...supervision.Logger) *ActorDescriptor
```
接口 `supervision.StrategyProvider` 的定义：
```go
type StrategyProvider interface {
	Provide() Strategy
}
```

为了更方便的指定监管策略，我们还提供了函数式的提供者：
```go
supervision.FunctionalStrategyProvider
```

如果采用函数式的方式指定，我们仅需要如下进行即可：
```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return ...
}))
```

当然，不仅仅是监管策略提供者，对于监管策略，也支持函数式创建：
```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return supervision.FunctionalStrategy(func(record *supervision.AccidentRecord) {
		record.Supervisor.Stop(record.Victim)
	})
}))
```

## 监管者 Actor
作为监管者，如果实现了 `supervision.Strategy` 接口，那么在没有为子 Actor 指定监管策略时，将会通过实现的策略进行决策。

例如 `guard` Actor，便是通过匿名组合的方式实现了监管策略接口：

```go
package vivid

import (
	"github.com/kercylan98/minotaur/engine/vivid/supervision"
	"github.com/kercylan98/minotaur/toolkit/log"
	"time"
)

type guard struct {
	supervision.Strategy
}

func (g *guard) OnReceive(ctx ActorContext) {
	switch ctx.Message().(type) {
	case *OnLaunch:
		g.Strategy = supervision.OneForOne(10, time.Millisecond*200, time.Second*3, supervision.FunctionalDecide(func(record *supervision.AccidentRecord) supervision.Directive {
			ctx.System().Logger().Warn("ActorSystem", log.String("info", "unsupervised accidents"), log.String("actor", record.Victim.LogicalAddress()), log.Any("reason", record.Reason))
			if len(record.Stack) > 0 {
				ctx.System().Logger().Error("trace", log.StackData("stack", record.Stack))
			}
			return supervision.DirectiveRestart
		}))
	}
}

```

## OneForOne 一对一策略
内置的监管策略中，包含一个较为复杂一对一监管策略，它将对发生意外的 Actor 执行重启、停止、升级或继续几个类型的指令。

```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return supervision.OneForOne(3, time.Millisecond*100, time.Millisecond*100, supervision.FunctionalDecide(func(record *supervision.AccidentRecord) supervision.Directive {
		return supervision.DirectiveRestart
	}))
}))
```
在这个例子中，当 Actor 发生意外时，则会通过退避指数算法进行最多 3 次的重启。

## StopStrategy 停止策略
当采用该策略时，发生意外时将会对其执行停止。

```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return supervision.StopStrategy()
}))
```

## RestartStrategy 重启策略
当采用该策略时，发生意外时将会立即重启。

```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return supervision.RestartStrategy()
}))
```

## ResumeStrategy 继续策略
当采用该策略时，发生意外时将会忽略错误继续运行。

```go
descriptor.WithSupervisionStrategyProvider(supervision.FunctionalStrategyProvider(func() supervision.Strategy {
	return supervision.ResumeStrategy()
}))
```