---
title: 快速开始
description: 搭建 Minotaur 的根基
published: true
date: 2024-07-01T07:22:47.668Z
tags: 
editor: markdown
dateCreated: 2024-06-19T10:29:43.149Z
---

# 介绍
Minotaur 的 `ActorSystem` 是其核心组件，通过 Actor 模型提供简便且高效的并发和分布式解决方案。Actor 模型是应对复杂并发问题的一种强大抽象，它避免了传统锁和回调的复杂性。每个 Actor 是一个独立的实体，通过消息传递进行通信，从而实现高并发和高可用性。

# 架构
在 Minotaur 中，每一个 `Actor` 都是一个父级单元，它可以包含多个子级单元，每一个 `Actor` 都有一个唯一的 `ActorId`，用于标识该 `Actor` 的唯一性。

`ActorSystem` 中默认包含了一个顶级的 `Actor`，它会对所有未被监管的 `Actor` 进行最终的执行。

我们可以通过这张图片看出来 `ActorSystem` 的整个层级关系是一个树状图。

![actor-system-tree.png](/actor-system-tree.png)

# ActorSystem 
`ActorSystem` 系统在 Minotaur 中扮演了维护所有 `Actor` 的角色，是一个重量级的数据结构，它在内部维护了一组本地进程地址信息及远程地址的解析，实现了

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


