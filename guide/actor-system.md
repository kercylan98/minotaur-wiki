---
title: Actor 模型
description: 搭建 Minotaur 的根基
published: true
date: 2024-07-04T08:15:00.700Z
tags: 
editor: markdown
dateCreated: 2024-06-19T10:29:43.149Z
---

# 介绍
自从分布式计算（Distributed computing）被提出后，业界就广泛开始了对这种新计算模式的研究和应用。在分布式系统中，传统的多线程和锁机制常常带来复杂的并发问题和性能瓶颈。而 `Actor` 模型作为一种并发编程模型，为解决这些问题提供了一种新颖且有效的方法。

# 什么是 Actor 模型？
`Actor` 模型是一种数学模型，用于处理并发计算。它由 Carl Hewitt、Peter Bishop 和 Richard Steiger 于 1973 年提出。
该模型的核心思想是将计算单元抽象为 `Actor`，每个 `Actor` 是独立的实体，拥有自己的状态和行为，并且通过消息传递来进行通信。

`Actor` 模型主要有以下几个关键概念：
- 一个 `Actor` 是一个独立的计算单元，具有自己的状态、行为和消息队列。每个 `Actor` 只负责处理自己的消息，不直接共享状态，从而避免了传统并发编程中的竞争条件问题。
- `Actor` 之间的通信是通过消息传递实现的。每个 `Actor` 具有一个唯一的地址（或引用），其他 `Actor` 可以通过该地址向它发送消息。消息传递是异步的，这意味着发送方不需要等待消息被处理。
- 当一个 `Actor` 接收到一条消息时，它可以根据当前的状态和消息内容决定自己的行为。具体来说，一个 `Actor` 在接收到消息后可以执行以下操作：
  - 创建新的 `Actor`。
  - 发送消息给其他 `Actor`。
  - 修改自己的状态。
- 不可变状态：`Actor` 模型提倡使用不可变状态，这意味着 `Actor` 在处理消息时不会直接修改其他 `Actor` 的状态，而是通过消息传递来间接影响它们。

# Actor 模型的优势
`Actor` 模型在处理并发和分布式计算时具有以下几个优势：
- 简化并发控制：由于 `Actor` 之间不共享状态，并发控制变得更加简单，不需要使用复杂的锁机制来避免竞争条件。
- 可扩展性：`Actor` 模型天然支持分布式计算和负载均衡，可以轻松扩展以应对大规模并发请求。
- 容错性：通过消息传递和 `Actor` 的独立性，可以实现高容错性和自愈能力。某个 `Actor` 出现故障时，不会影响其他 `Actor` 的正常运行。
- 模块化：`Actor` 模型提倡将系统分解为独立的 `Actor`，每个 `Actor` 专注于特定的任务，从而实现高度模块化和松耦合的系统设计。

# Actor 模型在游戏领域的影响
在游戏开发领域，`Actor` 模型因其处理并发和分布式系统的优势，得到了广泛的应用和认可。

## 并发处理与高性能
- 在线游戏通常需要处理大量用户的并发请求，例如实时多人游戏。在传统的线程模型中，需要通过复杂的锁机制来管理共享资源，容易导致性能瓶颈和死锁问题。而在 `Actor` 模型中，每个游戏实体（例如玩家、NPC、物品等）可以独立地作为一个 `Actor`，避免了复杂的并发控制，从而提升了系统的响应速度和可靠性。

- 游戏中的任务可以被分解成多个独立的 `Actor`，每个 `Actor` 负责处理特定的任务，例如路径计算、碰撞检测和 AI 逻辑等。这样可以更好地利用多核处理器的并行计算能力，提高游戏的整体性能。

## 高可扩展性
- 在大规模多人在线游戏（MMO）中，玩家分布在不同的区域或服务器上。`Actor` 模型的消息传递机制使得动态负载均衡变得更加容易。例如，可以根据玩家的分布情况动态地调整服务器资源，将繁忙区域的负载分配到空闲服务器上，从而提升系统的可扩展性和稳定性。

- 游戏服务器通常采用分布式架构，以应对大量的并发连接和请求。`Actor` 模型的天然分布式特性使得游戏服务器可以更容易地扩展和维护。通过 `Actor` 之间的消息传递，可以实现跨服务器的通信和协作，提升系统的扩展能力。

## 容错与恢复
- 在游戏服务器中，硬件故障、网络问题和软件错误都是常见的挑战。`Actor` 模型通过将系统分解为独立的 `Actor`，可以实现更高的容错性。例如，当某个 `Actor` 发生故障时，可以通过消息机制自动重启或迁移到其他服务器，确保系统的稳定性和连续性。

- `Actor` 模型支持自愈能力，当某个 `Actor` 出现异常时，可以自动重启并恢复到正常状态。这样可以大大减少人为干预和系统停机时间，提高游戏服务器的可靠性和用户体验。

## 模块化与可维护性
- 游戏开发通常涉及大量的功能模块，例如物理引擎、AI 系统和网络通信等。`Actor` 模型提倡将每个功能模块独立为 `Actor`，从而实现高度模块化的设计。这样可以使得各个模块之间的耦合度降低，提升系统的可维护性和可扩展性。

- 通过 `Actor` 模型，可以实现代码的高度复用。例如，可以将通用的功能封装为独立的 `Actor`，并在不同的游戏场景中重复使用，从而减少开发工作量和错误率。

# 架构
在 Minotaur 中，每一个 `Actor` 都是一个父级单元，它可以包含多个子级单元，每一个 `Actor` 都有一个唯一的 `ActorId`，用于标识该 `Actor` 的唯一性。

`ActorSystem` 中默认包含了一个顶级的 `Actor`，它会对所有未被监管的 `Actor` 进行最终的执行。

我们可以通过这张图片看出来 `ActorSystem` 的整个层级关系是一个树状图。

![actor-system-tree.png](/actor-system-tree.png)

# ActorSystem 
`ActorSystem` 在 Minotaur 中是维护所有本地及远程 `Actor` 的角色，是一个重量级的数据结构。



通常来说，每个应用程序中有且仅应该拥有至多一个 `ActorSystem`，它应该是作为全局的存在，并且在关闭时应确保通过 `ActorSystem.Shutdown` 函数安全的退出。

`ActorSystem.Shutdown` 函数被执行时，将会是优雅的，它会导致所有 `Actor` 依次退出。

## 创建 ActorSystem

通过以下代码即可创建一个 `ActorSystem`：

```go
package main

import "github.com/kercylan98/minotaur/core/vivid"

func main() {
	vivid.NewActorSystem()
}
```

## 第一个 Actor
```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/vivid"
)

type MyActor struct {
}

func (m *MyActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case string:
		fmt.Println(m)
	}
}

func main() {
	system := vivid.NewActorSystem()
	ref := system.ActorOf(func() vivid.Actor {
		return &MyActor{}
	})

	system.Context().Tell(ref, "Hi, minotaur")
	system.ShutdownGracefully()
}
```

# Actor
`Actor` 是基本的并发单元，独立处理其内部状态，并通过消息传递进行通信。正常情况每个 Actor 都会拥有一个自己的邮箱，用于接收和存储消息。

每个 `Actor` 均会需要实现 `vivid.Actor` 的 `OnReceive` 函数，所有的消息均会通过该函数进行处理。

> `OnReceive` 函数是线性的，每一条消息均会按照进入邮箱的顺序进行处理，Minotaur 不保证多个 `Actor` 同时向一个 `Actor` 发送消息的顺序，但是可以确保单个发送消息的 `Actor` 的消息是有序的。

> 在 Minotaur 中包含一种特殊的消息，它通过可选项 `vivid.WithInstantly(true)` 来触发，这一类消息将会越过所有正常消息进行最高优先级的处理，但是它依旧需要等待当前消息处理完成后才会进行。
{.is-warning}

## 定义 Actor 对象
当我们准备开始使用 `ActorSystem` 时，第一步就是定义一个 `Actor` 对象。定义的方式很简单，仅需要实现 `vivid.Actor` 接口即可，例如：

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type HelloActor struct {}

func (h *HelloActor) OnReceive(ctx vivid.MessageContext) {
	
}

```

## 创建 Actor 并获取到其引用

我们在刚才定义了一个 `HelloActor`，这个 `Actor` 不会做任何事情，但是当我们创建它的时候可以开启一个 `[goroutinue](https://go.dev/doc/effective_go#goroutines)`。

如何创建并运行我们定义的 `Actor` 呢？可以通过 `ActorOf[I|F|T|IT|FT]` 来进行创建，关于为什么有那么多可选项，我们后面再详解。首先我们回到正题，上代码：

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

func main() {
	system := vivid.NewActorSystem("example")
	defer system.Shutdown()
	
	ref := vivid.ActorOf[*HelloActor](&system)
}
```

这样简单的代码就将刚才的 `HelloActor` 创建并运行了，同时我们拿到了它的 `ActorRef`，通过这个 `ActorRef` 我们便可以控制向其发送消息了。

## 向 Actor 发送消息
我们的 `HelloActor` 的目的是当收到任一字符串时，在控制台打印 "Hello, XXX!"，并且将该字符串回复给发送方。接下来我们将通过两种方式向 `HelloActor` 发送消息。

### 通过 Tell 发送非阻塞消息
我们可以通过 `Tell` 方法向 `HelloActor` 发送非阻塞消息。非阻塞消息是指发送方不会等待接收方的回复，而是直接返回。这种方式适用于发送方不关心接收方的回复的场景。

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

func main() {
	system := vivid.NewActorSystem("example")
	defer system.Shutdown()

	ref := vivid.ActorOf[*HelloActor](&system)
	ref.Tell("World")
}
```

### 通过 Ask 发送阻塞消息
我们可以通过 `Ask` 方法向 `HelloActor` 发送阻塞消息。阻塞消息是指发送方会等待接收方的回复，直到接收到回复或者超时。这种方式适用于发送方需要接收方的回复的场景。

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

func main() {
	system := vivid.NewActorSystem("example")
	defer system.Shutdown()

	ref := vivid.ActorOf[*HelloActor](&system)
	ref.Ask("World")
}
```

## 接收并处理消息
当我们像之前那样向 `HelloActor` 发送 `Tell` 或 `Ask` 消息时，可以发现没有任何输出，并且 `Ask` 消息将会超时。这是因为我们还没有实现 `HelloActor` 的消息处理逻辑。接下来我们将实现 `HelloActor` 的消息处理逻辑。

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type HelloActor struct {}

func (h *HelloActor) OnReceive(ctx vivid.MessageContext) {
	switch m := ctx.GetMessage().(type) {
	case string:
		format := fmt.Sprintf("Hello, %s!", m)
		fmt.Println(format)
		ctx.Reply(format)
	}
}
```

在 `HelloActor` 的 `OnReceive` 方法中，我们通过 `GetMessage` 方法获取消息内容，并根据消息内容的类型进行处理。在这里我们只处理字符串类型的消息，当收到字符串类型的消息时，我们将字符串格式化为 "Hello, XXX!"，并且将格式化后的字符串打印到控制台，并回复给发送方。

接下来我们再次尝试向 `HelloActor` 发送消息，你会发现控制台打印了 "Hello, World!"，并且 `Ask` 消息也收到了回复。

你可能会有疑问，为什么很明确知道只会有 `string` 类型消息的情况下还需要通过 `switch` 语句判断消息类型呢？这是因为 `Actor` 不仅仅会接收到我们发送的消息，还会接收到系统发送的消息，例如 `Actor` 的生命周期消息。在这种情况下，我们需要通过 `switch` 语句判断消息类型，以区分不同类型的消息。

## 完整示例

```go

package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type HelloActor struct{}

func (h *HelloActor) OnReceive(ctx vivid.MessageContext) {
	switch m := ctx.GetMessage().(type) {
	case string:
		format := fmt.Sprintf("Hello, %s!", m)
		fmt.Println(format)
		ctx.Reply(format)
	}
}

func main() {
	system := vivid.NewActorSystem("example")
	defer system.Shutdown()

	ref := vivid.ActorOf[*HelloActor](&system)
	ref.Ask("World")
}
```


