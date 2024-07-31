---
title: 远程访问
description: 跨 ActorSystem 的网络访问
published: true
date: 2024-07-31T02:17:12.205Z
tags: actor, vivid, actor system, remoting
editor: markdown
dateCreated: 2024-07-29T11:04:21.448Z
---

# 介绍

在 Minotaur 的 Vivid 设计中，它是天然支持远程访问的，仅需要打开共享功能并提供一个远程的 `vivid.ActorRef` 即可拥有像本地使用一样的使用体验，这得益于 `prc` 进程资源控制器的实现。

我们在使用远程访问时，也是通过消息传递的方式来进行的，而由于多了网络层，这就意味着我们的消息必须是要可被序列化及反序列化的。默认情况下使用的是 `ProtoBuffer` 编解码器。

# 打开共享

当我们的 ActorSystem 需要与其他 ActorSystem 交互时，可通过选项 `WithShared` 来开启远程功能。

该函数也是来自 `ActorSystemConfiguration` 的一个选项，它是用于开启网络共享的关键，同时它还接受一个可变参数可用于定义编解码器：

```go
func (c *ActorSystemConfiguration) WithShared(address prc.PhysicalAddress, codec ...codec.Codec) *ActorSystemConfiguration
```

> 关于 `address` 它是用于绑定接受网络访问的地址，例如：`":8080"`
{.is-info}


## 使用示例

实际上的使用方式是非常简单的，例如：

```go
package vivid_test

import (
	"github.com/kercylan98/minotaur/engine/prc"
	"github.com/kercylan98/minotaur/engine/vivid"
	"sync"
	"testing"
)

func TestActorSystemConfiguration_WithSharedFutureAsk(t *testing.T) {
	var messageNum = 1000
	var receiverOnce, senderOnce sync.Once

	system1 := vivid.NewActorSystem(vivid.FunctionalActorSystemConfigurator(func(config *vivid.ActorSystemConfiguration) {
		config.WithShared(":8080")
	}))

	system2 := vivid.NewActorSystem(vivid.FunctionalActorSystemConfigurator(func(config *vivid.ActorSystemConfiguration) {
		config.WithShared(":8081")
	}))

	ref1 := system1.ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			switch v := ctx.Message().(type) {
			case *prc.ProcessId:
				receiverOnce.Do(func() {
					t.Log("receiver start", ctx.Ref().URL().String(), ctx.Sender().URL().String())
				})
				ctx.Reply(v)
			}
		})
	})

	ref2 := ref1.Clone() // 同一进程内，隔离开，避免通过缓存直接调用

	message := prc.NewProcessId("none", "none")
	for i := 1; i <= messageNum; i++ {
		if err := system2.FutureAsk(ref2, message, -1).Wait(); err != nil {
			panic(err)
		}
		senderOnce.Do(func() {
			t.Log("sender start")
		})
	}

	system1.Shutdown(true)
	system2.Shutdown(true)
}
```

可以看到我们创建了两个 ActorSystem，并且通过其中一个向另一个的 Actor 发送 1000 条 `FutureAsk` 消息并等待其回复。

这里由于是同一进程中运行，你可能会想，不同进程下怎么才能拿到另一个 ActorSystem 的 ActorRef 呢？这里提供了多种方式：
 - 最简单的便是通过 `vivid.NewActorRef` 函数来指定物理地址和逻辑地址以及 ActorSystem 名称的方式创建一个远程 `vivid.ActorRef`。
 - 当然，在我们不知道名称的情况下，也可以通过数据库例如 Redis 存储的方式来共享 ActorRef，它是支持序列化的。