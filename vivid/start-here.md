---
title: 从这里开始
description: 了解并开始使用 Vivid
published: true
date: 2024-08-05T10:57:59.826Z
tags: actor, vivid, actor system
editor: markdown
dateCreated: 2024-06-21T06:13:28.417Z
---

# 什么是 Vivid？

如果你已经阅读过[网络服务器](/zh/guide/network-server)和[聊天室示例](/zh/guide/chat-room)这两篇文章，那么你应该已经大概了解 Minotaur 的使用方法了，接下来我们就该了解一下它们的依靠 vivid 了！

Vivid 是 Minotaur 的核心实现之一，提供了对 Actor 模型的实现。如果您不熟悉 Actor 模型，可以先阅读以下相关文章：
- [📖 Wikipedia: *讲述 Actor 模型的背景及基本概念。*](https://zh.wikipedia.org/wiki/%E6%BC%94%E5%91%98%E6%A8%A1%E5%9E%8B)
- [📖 Rocketeer.be: *Concurrency in Erlang & Scala: The Actor Model。*](https://rocketeer.be/articles/concurrency-in-erlang-scala/)
{.links-list}

首先，Vivid 是一个 Go 语言的包名，位于 `github.com/kercylan98/minotaur/engine/vivid` 路径下。

在 Vivid 中提供了完整的 ActorSystem 实现，使您可以专注于业务需求的编写，而不必编写大量繁杂的可靠性代码来实现所谓的可靠性、容错和高性能。

长期的 Go 语言游戏服务端开发实践中，我们发现，许多问题是由于内存泄漏、死锁、并发等问题造成的。而从经验来看，底层代码的实现正是事故的高发点。而为了解决这些问题，Vivid 采用了一系列优化技术和设计模式，帮助开发者有效避免常见的并发和内存管理问题，从而提高系统的稳定性和性能：

- Vivid 通过 Actor 模型的无锁并发机制，实现了高效的并发控制。每个 Actor 独立处理自己的消息队列，避免了传统并发编程中复杂的锁定和同步问题。通过合理设计 Actor 之间的消息传递，开发者可以轻松实现高性能的并发应用。
- 通过消息传递机制，Vivid 实现了无锁并发，这大大提高了系统的性能。与传统的多线程编程相比，无锁并发避免了上下文切换和锁竞争的开销，能够更高效地利用多核 CPU 的计算能力。此外，Vivid 对消息队列进行了优化，确保了消息传递的低延迟和高吞吐量。

除此之外，Vivid 还提供了网络及集群方面的支持，使得编写并行的分布式程序更为简单轻松。

> 在 Minotaur 中，vivid 的使用场景是非常多见的，基本大部分的实现均是 vivid 作为底部支撑，掌握 vivid，您将获得一把强大的分布式利剑（万剑归宗？逃~ 🙈）
{.is-info}

# 简单速览

我们可以从一个示例简单的看一下使用过程，以打消潜意识内认为它很复杂的疑惑：

```go
package main

import (
	"fmt"
	"github.com/kercylan98/minotaur/engine/vivid"
)

func main() {
	system := vivid.NewActorSystem()
	defer system.Shutdown(true)

	system.ActorOfF(func() vivid.Actor {
		return vivid.FunctionalActor(func(ctx vivid.ActorContext) {
			switch ctx.Message().(type) {
			case *vivid.OnLaunch:
				fmt.Println("Hello, World!")
			}
		})
	})
}
```

这样便完成了 ActorSystem 以及 Actor 的创建，并且我们可以在屏幕上看到打印了 "Hello World!"。

> 为了更好的体验，我们还提供了很多达到相同目的函数，例如提供器接口、函数式声明等，以便可以结合自身完成更高自由度的扩展。
{.is-info}

# 基准测试

如果需要快速了解 vivid 的网络传输性能，那么运行基准测试是最方便的。

vivid 提供了非常快，非常非常快的远程处理。目前两个 Actor 在节点间每秒能够传输超过 600 万条消息！

![vivid-shared-speed.gif](/actor-system/vivid-shared-speed.gif)

```shell
ax benchmark network
```