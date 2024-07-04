---
title: 快速开始
description: 快速上手 Vivid ActorSystem
published: false
date: 2024-07-04T09:19:24.823Z
tags: actorsystem, actor, vivid
editor: markdown
dateCreated: 2024-06-21T06:13:28.417Z
---

# 创建 ActorSystem
ActorSystem 是 Actor 的容器，也是管理 Actor 生命周期的中心点。它负责创建、和维护 Actor。

首先我们来创建一个 ActorSystem：
```go
package main

import "github.com/kercylan98/minotaur/core/vivid"

func main() {
	vivid.NewActorSystem()
}
```

就是如此简单，运行后会发现程序立刻结束了，这是因为 ActorSystem 的运行是非阻塞的。

# 创建一个 Actor
Actor 是参与计算的基本单元，实现一个 Actor 我们仅需要定义一个结构体并实现 `OnReceive(vivid.Message)` 函数即可。

## 定义 Actor
首先我们来定义一个 Actor，我们需要这个 Actor 收到 `string` 消息时打印 `Hello world`，并且回复一个 `Hello world`。

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/vivid"
)

type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case string:
		fmt.Println("Hello world")
		ctx.Reply(m)
	}
}
```

## 运行 Actor
定义完成 Actor 后接下来我们便可以通过 `ActorOf` 函数来启动 Actor，当前首先是要先创建 `ActorSystem`。

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/vivid"
)

type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case string:
		fmt.Println("Hello world")
		ctx.Reply(m)
	}
}

func main() {
	system := vivid.NewActorSystem()
	system.ActorOf(func() vivid.Actor {
		return &HelloActor{}
	})
}
```

## 发送消息并接收回复
消息的传递依赖Actor的引用（ActorRef），而引用则可以在 `ActorOf` 之后通过返回值获取到，通过引用我们就可以向 Actor 发送消息了。

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/core/vivid"
)

type HelloActor struct{}

func (m *HelloActor) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case string:
		fmt.Println("Hello world!")
		ctx.Reply(m)
	}
}

func main() {
	system := vivid.NewActorSystem()
	ref := system.ActorOf(func() vivid.Actor {
		return &HelloActor{}
	})

	reply := system.Context().FutureAsk(ref, "Hey, sao ju~").AssertResult()
	fmt.Println(reply)
}
```

运行后便可以看到屏幕上输出了如下内容：

```shell
Hello world!
Hey, sao ju~
```

就这样！您已成功创建了一个基本的示例。
