---
title: Actor Lifecycle
description: 
published: true
date: 2024-06-21T03:53:42.825Z
tags: 
editor: markdown
dateCreated: 2024-06-21T03:53:42.825Z
---

# Introduction
The `Actor` model is a programming model used for concurrent computing, particularly common in distributed systems. In the `Actor` model, an `Actor` is an independent computational entity that communicates through message passing. Each `Actor` has its own state and behavior, and computation occurs only upon receiving messages. Understanding the lifecycle of an `Actor` is crucial for effectively utilizing this model.

# Lifecycle
In Minotaur, if you want to listen for lifecycle events, you only need to type assert specific lifecycle message types in the `OnReceive` function. For example:

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type MyActor struct{}

func (h *MyActor) OnReceive(ctx vivid.MessageContext) {
	switch ctx.GetMessage().(type) {
	case vivid.OnBoot:

	}
}
```

> Different lifecycle messages may carry additional parameters.
{.is-info}

## vivid.OnInit
This is the first stage of the `Actor` lifecycle. Although it is called initialization, it is not the initialization of the `Actor` itself, but rather the initialization of `ActorOptions`.

> At this moment, the `Actor` is not available. It is usually used for checking and controlling the `ActorOptions` requirements.
{.is-warning}

## vivid.OnBoot
When the `Actor` enters this stage, it means the `Actor` has been fully initialized and is ready to start processing messages.

The `vivid.OnBoot` message is sent as the first message to the `Actor`. The execution of this message is blocking, which means that functions like `ActorOf`, which create the `Actor`, will be blocked until the `Actor` completes processing this message.

Usually, at this stage, we can complete the initialization of the `Actor` object's properties.

## vivid.OnRestart
This stage does not necessarily occur but is sent right after `OnBoot` when the `Actor` is restarted. Similarly, the execution of this message is also blocking.

## vivid.OnActorTyped
This is a special lifecycle stage. When the `Actor` is created through a typed interface, this lifecycle will be triggered, occurring immediately after `OnBoot` or `OnRestart`.

> This lifecycle is very useful when the `Actor` needs to obtain its own typed interface.
{.is-info}

## vivid.OnTerminate
This is the termination message of the `Actor`. When an `Actor` receives this message, it should complete the cleanup of its resources.

When this message is actively sent to the `Actor`, it will also cause the termination of the `Actor`. The principle of the `ActorRef.Terminate()` function is to send this message to the `Actor`.

> When the `Actor` is closed due to unexpected situations, and its parent's supervision strategy restarts it, all child `Actors` of this `Actor` will also be restarted. The order of shutdown is from child to parent, and the order of restart is from parent to child.
{.is-info}

> After the `Actor` receives this message, the `ActorSystem` will close its mailbox, reject new messages, and recursively send this message to its child `Actors`. After all child `Actors` are terminated, the `Actor` will be released.
{.is-warning}
