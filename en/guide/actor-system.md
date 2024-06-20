---
title: Actor System
description: Building the foundation of Minotaur
published: true
date: 2024-06-20T16:05:01.085Z
tags: 
editor: markdown
dateCreated: 2024-06-20T16:05:01.085Z
---

# Introduction
Minotaur's `ActorSystem` is its core component, providing a simple and efficient concurrent and distributed solution through the Actor model. The Actor model is a powerful abstraction for addressing complex concurrency issues, avoiding the complexities of traditional locks and callbacks. Each Actor is an independent entity that communicates through message passing, enabling high concurrency and high availability.

# Architecture
In Minotaur, each `Actor` is a parent unit that can contain multiple child units. Every `Actor` has a unique `ActorId` to identify its uniqueness.

The `ActorSystem` includes two top-level `Actors` by default, which are defined and created when you create the `ActorSystem`:
- `UserGuardActor`: Serves as the top-level user guardian `Actor`, overseeing every `Actor` created by the user.
- `EventBusActor`: An independent top-level `Actor` that maintains the event bus within the `ActorSystem`.

We can visualize the entire hierarchy of the `ActorSystem` as a tree structure using this image:

![actor-system-tree.png](/actor-system/actor-system-tree.png)

The `UserGuardActor` is primarily used for supervision purposes. It takes final action in cases where supervision boundaries are exceeded. Currently, actors are restarted unless they are explicitly destroyed.

# ActorSystem

The `ActorSystem` in Minotaur serves as the overseer of all `Actors`, functioning as a heavyweight data structure that starts running upon completion of its creation.

Typically, each application should have at most one `ActorSystem`, which should exist as a global entity and should be safely shut down using `ActorSystem.Shutdown` upon termination.

When `ActorSystem.Shutdown` is executed, it gracefully terminates, causing all `Actors` to sequentially exit.

> Actor termination is asynchronous: upon receiving an exit message, an Actor first stops processing new messages, sends exit messages to all its child Actors, waits for them to complete their exit behavior, and then finishes its own message processing before exiting.

## Creating an ActorSystem

In Minotaur, despite the encapsulation of `ActorSystem` within the `minotaur` package, you can still instantiate an `ActorSystem` independently.

Here's how you can create an `ActorSystem`:

```go
package vivid_test

import "github.com/kercylan98/minotaur/minotaur/vivid"

func ExampleNewActorSystem() {
	vivid.NewActorSystem("example")

	// Output:
	//
}
```

> Example code: [https://github.com/kercylan98/minotaur/blob/develop/minotaur/vivid/actor_system_example_test.go](https://github.com/kercylan98/minotaur/blob/develop/minotaur/vivid/actor_system_example_test.go)

It's quite straightforward, isn't it?

In the `vivid.NewActorSystem` function, it also accepts optional parameters that allow us to customize behaviors of the `ActorSystem`, such as setting up loggers or defining names. For more details, please refer to: [ActorSystem Options](/guide/actor-system/actor-system-options)

# Actor

An `Actor` is a fundamental unit of concurrency in Minotaur, handling its internal state independently and communicating through message passing. Typically, each Actor has its own mailbox for receiving and storing messages.

Every `Actor` needs to implement the `OnReceive` function of `vivid.Actor`, where all messages are processed.

> The `OnReceive` function operates linearly, processing messages in the order they arrive in the mailbox. Minotaur does not guarantee the order of messages sent by multiple Actors to one Actor simultaneously, but ensures that messages from a single sending Actor are processed in order.

> Minotaur includes a special type of message triggered by the optional parameter `vivid.WithInstantly(true)`. These messages are processed with the highest priority, bypassing all normal messages. However, they still wait for the current message processing to complete.
{.is-warning}

## Defining an Actor Object

When preparing to use the `ActorSystem`, the first step is defining an `Actor` object. This is straightforward: you only need to implement the `vivid.Actor` interface. For example:

```go
package main

import (
	"github.com/kercylan98/minotaur/minotaur/vivid"
)

type HelloActor struct {}

func (h *HelloActor) OnReceive(ctx vivid.MessageContext) {
	
}

```

## Creating an Actor and Obtaining its Reference

After defining a `HelloActor` earlier, which doesn't perform any specific actions, we can create it and start a [goroutine](https://go.dev/doc/effective_go#goroutines) when it's instantiated.

How do we create and run our defined Actor? We can use `ActorOf[I|F|T|IT|FT]` for instantiation. We'll delve into the reasons for these options later. First, let's get to the code:

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
This simple code snippet has created and started the `HelloActor` we defined earlier. Additionally, we obtained its `ActorRef`, which allows us to send messages to it.

## Sending Messages to an Actor

The purpose of our `HelloActor` is to print "Hello, XXX!" to the console and reply with the received string whenever it receives any string message. Next, we'll demonstrate two ways to send messages to the `HelloActor`.

### Sending Non-blocking Messages via Tell

We can use the `Tell` method to send non-blocking messages to `HelloActor`. Non-blocking messages mean that the sender does not wait for a reply from the receiver but returns immediately. This method is suitable when the sender does not need to wait for a response from the receiver.

Here's how to do it:
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

### Sending Blocking Messages via Ask

We can use the `Ask` method to send blocking messages to `HelloActor`. Blocking messages mean that the sender will wait for a reply from the receiver until it receives a response or times out. This method is suitable when the sender needs a response from the receiver.

Here's how to do it:

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

## Receiving and Handling Messages

When we send `Tell` or `Ask` messages to `HelloActor` as we did before, you may notice that there is no output and the `Ask` message will time out. This is because we haven't implemented the message handling logic for `HelloActor` yet. Next, we will implement the message handling logic for `HelloActor`.

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

In the `OnReceive` method of `HelloActor`, we use the `GetMessage` method to retrieve the content of the message and process it based on its type. Here, we handle messages of type `string` only. When receiving a string message, we format it as "Hello, XXX!" and print it to the console. Additionally, we reply with the formatted string to the sender.

Next, let's try sending messages to `HelloActor` again. You will see "Hello, World!" printed on the console, and the `Ask` message will receive a reply.

You might wonder why we use a `switch` statement to check the message type when we already know we are expecting only `string` messages. This is because an `Actor` not only receives messages sent by us but also system-generated messages, such as lifecycle messages of the `Actor`. In such cases, we use a `switch` statement to differentiate between different types of messages.

## Complete Example

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


