---
title: 发布与订阅
description: 简单灵活且高效的消息总线
published: true
date: 2024-08-07T11:22:07.623Z
tags: actor, vivid, actor system, cluster, message-bus, pub-sub
editor: markdown
dateCreated: 2024-08-07T11:22:07.623Z
---

# 发布与订阅
在 Vivid 中提供了一项功能，它可让您将消息从发布者广播到多个订阅者。在发布者端，消息发布到特定 `Topic`，而在订阅者端可以通过 `Topic` 订阅主题。

发布者和订阅者不仅可以在本地，如果您的消息是通过 `Protobuf` 描述的，那么也可以散播在网络或集群中。当然不同的是，在单独使用 `WithShared` 打开网络功能时，发布与订阅并不会穿透本地节点，如果需要扩散到网络节点，那么需要先为其建立连接。建立连接的方式也很简单，发送一条跨节点消息即可。

如果是在集群中使用，那么不必做任何额外的工作，仅需要正常使用集群即可。 Vivid 会在节点感知到时自动为其建立联系，并且函数签名也是完全相同的！

> Topic 尚不具备唯一性，这意味着相同的 `Topic` 可能会收到不同类型的消息。
{.is-warning}

# 发布消息
消息发布是通过 ActorContext 和 ActorSystem 的 `Publish` 函数来进行的，它支持发布单条消息，但是如果有多条消息，在经过网络时将会自动被整合为消息批。

```go
package main

import "github.com/kercylan98/minotaur/engine/vivid"

func main() {
	vivid.NewActorSystem().Publish("chat", "hello")
}
```

> 订阅者在收到消息时是可以对其进行回复的。
{.is-info}

# 订阅消息
订阅消息通过 ActorContext 的 `Subscribe` 即可进行订阅，这样您便可以在 `OnReceive` 函数中处理订阅消息：

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/engine/vivid"
)

func main() {
	type ChatMessage string
	
	vivid.NewActorSystem().ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			switch m := ctx.Message().(type) {
			case *vivid.OnLaunch:
				ctx.Subscribe("chat")
				ctx.Publish("chat", ChatMessage("hi"))
			case ChatMessage:
				fmt.Println(m)
			}
		})
	})
}
```

# 取消订阅
取消订阅通过 ActorContext 的 `Unsubscribe` 即可进行完成，比较特殊的是，您需要传入的是 `Subscribe` 函数返回的句柄，而不是 `Topic`：

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/engine/vivid"
)

func main() {
	type ChatMessage string

	vivid.NewActorSystem().ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			switch m := ctx.Message().(type) {
			case *vivid.OnLaunch:
				sub := ctx.Subscribe("chat")
				ctx.UnSubscribe(sub)
				ctx.Publish("chat", ChatMessage("hi"))
			case ChatMessage:
				fmt.Println(m) // not print
			}
		})
	})
}
```

> 既然有取消，那势必会存在忘记取消而导致内存泄漏的可能，在 Vivid 中，当 Actor 重启或终止时，都会对其所有订阅进行取消，以最大限度的避免内存泄漏!
{.is-info}
