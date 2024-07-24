---
title: 简单的聊天室
description: 简单高效、并发安全的聊天室
published: true
date: 2024-07-24T09:18:46.575Z
tags: actor, websocket, actor system, chat-room
editor: markdown
dateCreated: 2024-07-24T09:18:46.575Z
---

# 介绍

在 Socket 服务器示例中，最经典的莫过于聊天室场景了。在 Minotaur 中同样的也提供了一个基于 WebSocket 和 vivid 实现的聊天室示例，它包含了简单的加入房间、发送消息、离开房间、释放房间的处理。

在这个示例中，可以了解到通过 Actor 模型，我们能够搭建线程安全且高效的房间类应用。

> 示例代码不包含客户端，但是你可以通过任一 WebSocket 客户端进行访问测试，这里推荐使用 [http://www.websocket-test.com](http://www.websocket-test.com)
{.is-info}


# 房间管理器

首先，关于房间类的实现，我们应该有一个房间管理器，它用来维护所有房间，并且提供已有的房间。

由于我们退出房间准备通过断开 WebSocket 连接来实现，所以这一块我们不需要定义这个请求的消息，这样下来，我们的房间管理器仅需要一个 `查找或创建房间` 的消息便可以开始工作！

让我们开始编写业务代码：

```go
package manager

import (
	"github.com/kercylan98/minotaur/engine/prc"
	"github.com/kercylan98/minotaur/engine/vivid"
	"github.com/kercylan98/minotaur/examples/internal/chat-room/internal/manager/room"
	"github.com/kercylan98/minotaur/toolkit/log"
)

type (
	FindOrCreateRoomMessage struct {
		RoomId string
	}

	FindOrCreateRoomMessageReply struct {
		Room vivid.ActorRef
	}
)

func New() *Manager {
	return &Manager{
		rooms:   map[string]vivid.ActorRef{},
		roomMap: make(map[prc.LogicalAddress]string),
	}
}

type Manager struct {
	rooms   map[string]vivid.ActorRef
	roomMap map[prc.LogicalAddress]string
}

func (m *Manager) OnReceive(ctx vivid.ActorContext) {
	switch msg := ctx.Message().(type) {
	case *FindOrCreateRoomMessage:
		m.onFindOrCreateRoom(ctx, msg)
	case *vivid.OnTerminated:
		m.onTerminated(ctx, msg)
	}
}

func (m *Manager) onFindOrCreateRoom(ctx vivid.ActorContext, msg *FindOrCreateRoomMessage) {
	r, exist := m.rooms[msg.RoomId]
	if !exist {
		r = ctx.ActorOfF(func() vivid.Actor {
			return room.New(msg.RoomId)
		})
		m.roomMap[r.LogicalAddress()] = msg.RoomId
		m.rooms[msg.RoomId] = r
		ctx.System().Logger().Info("RoomManager", log.String("status", "create"), log.String("room_id", msg.RoomId))
	}

	ctx.Reply(&FindOrCreateRoomMessageReply{Room: r})
}

func (m *Manager) onTerminated(ctx vivid.ActorContext, msg *vivid.OnTerminated) {
	roomId, exist := m.roomMap[msg.TerminatedActor.LogicalAddress()]
	if exist {
		delete(m.roomMap, msg.TerminatedActor.LogicalAddress())
		delete(m.rooms, roomId)
		ctx.System().Logger().Info("RoomManager", log.String("status", "destroy"), log.String("room_id", roomId))
	}
}

```

这段代码可能稍显繁杂，让我们一步步道来！

在 `onFindOrCreateRoom` 函数中很明显，我们会去检查这个房间 id 是否存在，如果不存在，那么我们创建它并且绑定记录。而为什么需要绑定房间的逻辑地址呢？因为我们需要监听房间的销毁来完成内存的清理工作。

可能大家还会问到，为什么对 map 的读写不进行加锁？这便是通过 Actor 模型带来的好处，在房间管理器中，任何消息执行都是串行的，也就意味着不存在并发问题。

在 `onTerminated` 函数中便是我们释放房间的过程，当子 Actor 被销毁时，父 Actor 均会收到消息，通过比较唯一的逻辑地址，我们便可以完成内存的清理。

# 聊天室
聊天室便是房间管理器创建的子 Actor，它其中需要记录聊天室内有哪些用户，以及支持进入房间并发送消息，所以我们得到下面这段代码：

```go
package room

import (
	"github.com/kercylan98/minotaur/engine/prc"
	"github.com/kercylan98/minotaur/engine/vivid"
)

type (
	JoinRoomMessage struct {
		UserId string
		User   vivid.ActorRef
	}

	ChatMessage struct {
		UserId string
		Chat   string
	}
)

func New(roomId string) *Room {
	return &Room{
		roomId:  roomId,
		users:   map[string]vivid.ActorRef{},
		userMap: make(map[prc.LogicalAddress]string),
	}
}

type Room struct {
	roomId  string
	users   map[string]vivid.ActorRef
	userMap map[prc.LogicalAddress]string
}

func (r *Room) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case *JoinRoomMessage:
		r.onJoinRoom(ctx, m)
	case *ChatMessage:
		r.onChat(ctx, m)
	case *vivid.OnTerminated:
		r.onTerminated(ctx, m)
	}
}

func (r *Room) onJoinRoom(ctx vivid.ActorContext, m *JoinRoomMessage) {
	if _, exist := r.users[m.UserId]; exist {
		return
	}

	r.userMap[m.User.LogicalAddress()] = m.UserId
	r.users[m.UserId] = m.User
	message := &ChatMessage{
		UserId: m.UserId,
		Chat:   "Joined the room",
	}

	ctx.Watch(m.User)

	for _, ref := range r.users {
		ctx.Tell(ref, message)
	}
}

func (r *Room) onChat(ctx vivid.ActorContext, m *ChatMessage) {
	if _, exist := r.users[m.UserId]; !exist {
		return
	}
	for _, ref := range r.users {
		ctx.Tell(ref, m)
	}
}

func (r *Room) onTerminated(ctx vivid.ActorContext, m *vivid.OnTerminated) {
	userId, exist := r.userMap[m.TerminatedActor.LogicalAddress()]
	if !exist {
		return
	}

	delete(r.users, userId)
	delete(r.userMap, m.TerminatedActor.LogicalAddress())

	message := &ChatMessage{
		UserId: userId,
		Chat:   "Leaved the room",
	}
	for _, ref := range r.users {
		ctx.Tell(ref, message)
	}

	if len(r.userMap) == 0 {
		ctx.Terminate(ctx.Ref(), true)
	}
}
```

可以看到，我们在 `onJoinRoom` 函数中将用户记录下来，并且在用户加入房间时向所有人广播一条消息，告知其到来！值得注意的是，我们在加入房间时候还对用户进行了 `Watch` 操作，这个函数来自于 vivid 的 ActorContext，它能够让被监听的 Actor 在销毁时告知自己。

在 `onChat` 函数中则处理了消息发送的逻辑，它会将用户发送的消息对所有人进行广播。

而在 `onTerminated` 函数中，由于我们在用户加入房间的时候对其进行了监听，所以我们能够收到用户断开连接销毁的消息，这样便能够完成用户的清理，而当用户数量为 0 时，我们关闭这个房间，之后便由房间管理器进行收尾工作。

# 用户
用户是一个比较特殊的 Actor，至于特殊在哪里，我们稍后会进行介绍。首先我们用户需要能够获取到房间管理器，并且能够加入或创建房间、发送消息等。在这里，我们把校验工作放在连接打开的时候，并且在收到客户端消息时告知房间并由房间广播。

```go
package user

import (
	"fmt"
	"github.com/gofiber/contrib/websocket"
	"github.com/kercylan98/minotaur/engine/stream"
	"github.com/kercylan98/minotaur/engine/vivid"
	"github.com/kercylan98/minotaur/examples/internal/chat-room/internal/manager"
	"github.com/kercylan98/minotaur/examples/internal/chat-room/internal/manager/room"
	"github.com/kercylan98/minotaur/toolkit/charproc"
)

func New(mgr vivid.ActorRef) *User {
	return &User{
		manager: mgr,
	}
}

type User struct {
	manager vivid.ActorRef
	writer  stream.Writer
	conn    *websocket.Conn
	userId  string
	room    vivid.ActorRef
}

func (u *User) OnReceive(ctx vivid.ActorContext) {
	switch m := ctx.Message().(type) {
	case stream.Writer:
		u.writer = m
	case *websocket.Conn:
		u.onConnOpened(ctx, m)
	case *stream.Packet:
		u.onPacket(ctx, m)
	case *room.ChatMessage:
		ctx.Tell(u.writer, stream.NewPacketSC(fmt.Sprintf("%s: %s", m.UserId, m.Chat), websocket.TextMessage))
	}
}

func (u *User) onConnOpened(ctx vivid.ActorContext, m *websocket.Conn) {
	userId := m.Query("userId")
	roomId := m.Query("roomId")
	if userId == charproc.None {
		ctx.Tell(u.writer, stream.NewPacketSC("please input userId query param, example for: ws://127.0.0.1:8080/ws?userId=kercylan&roomId=123", websocket.TextMessage))
		ctx.Terminate(ctx.Ref(), true)
		return
	}
	if roomId == charproc.None {
		ctx.Tell(u.writer, stream.NewPacketSC("please input roomId query param, example for: ws://127.0.0.1:8080/ws?userId=kercylan&roomId=123", websocket.TextMessage))
		ctx.Terminate(ctx.Ref(), true)
		return
	}

	// find or create
	reply, err := ctx.FutureAsk(u.manager, &manager.FindOrCreateRoomMessage{
		RoomId: roomId,
	}).Result()
	if err != nil {
		ctx.Tell(u.writer, stream.NewPacketSC("get room info failed, please retry!", websocket.TextMessage))
		ctx.Terminate(ctx.Ref(), true)
		return
	}

	r := reply.(*manager.FindOrCreateRoomMessageReply)
	ctx.Tell(r.Room, &room.JoinRoomMessage{
		UserId: userId,
		User:   ctx.Ref(),
	})

	u.userId = userId
	u.room = r.Room
}

func (u *User) onPacket(ctx vivid.ActorContext, m *stream.Packet) {
	if u.room == nil {
		return
	}

	ctx.Tell(u.room, &room.ChatMessage{
		UserId: u.userId,
		Chat:   string(m.Data()),
	})
}
```

可以看到，用户在创建时候便要求了房间管理器的引用作为参数，之后在处理的消息中当收到连接时，表示连接已经打开，这时候我们从连接获取必要参数，对其校验后向房间管理器查找或创建一个房间并加入。

特殊的点在于，用户虽然实现了 Actor，但它并不是一个 Actor，而是作为 Stream Actor 的行为表现去使用的。如果你已经了解了网络服务器那一章节，你会理解这是什么。

关于 `stream.Writer`，它是来自流传入的写入器，通过它我们可以向客户端发送消息，而 `websocket.Conn` 则是 Minotaur 对于 WebSocket 封装中传入的连接，通过它，我们可以获取到原生 WebSocket 中必要的信息。

# 开始运行
接下来便是我们的 main.go 文件，它很简单。它会建一个 `fiber.App` 和 `vivid.ActorSystem`，并且在 ActorSystem 中创建我们的房间管理器，之后通过 `stream` 包中的 `NewFiberWebSocketHandler` 函数创建一个 WebSocket 处理函数并交由 fiber 进行路由。

之前说比较特殊的点便是，我们没有创建过用户的 Actor，用户实际上是对于 `stream.Actor` 的扩展。

注意这行代码：`c.WithPerformance(vivid.FunctionalActorPerformance(user.New(roomManager).OnReceive))`，我们创建了一个用户，并且将用户的 `OnReceive` 函数交给了 Stream 的行为表现可选项，之后 Stream 在处理完消息后均会向这个函数进行一次转发，以此，我们便继承了 Stream 的功能，并对齐完成了扩展。

```go
package main

import (
	"github.com/gofiber/fiber/v2"
	"github.com/kercylan98/minotaur/engine/stream"
	"github.com/kercylan98/minotaur/engine/vivid"
	"github.com/kercylan98/minotaur/examples/internal/chat-room/internal/manager"
	"github.com/kercylan98/minotaur/examples/internal/chat-room/internal/user"
)

func main() {
	fiberApp := fiber.New()
	system := vivid.NewActorSystem()
	defer system.Shutdown(true)

	roomManager := system.ActorOfF(func() vivid.Actor {
		return manager.New()
	})

	fiberApp.Get("/ws", stream.NewFiberWebSocketHandler(fiberApp, system, stream.FunctionalConfigurator(func(c *stream.Configuration) {
		c.WithPerformance(vivid.FunctionalActorPerformance(user.New(roomManager).OnReceive))
	})))

	if err := fiberApp.Listen(":8080"); err != nil {
		panic(err)
	}
}
```

> 完整示例代码包含在项目仓库中，访问地址为：[https://github.com/kercylan98/minotaur/tree/develop/examples/internal/chat-room](https://github.com/kercylan98/minotaur/tree/develop/examples/internal/chat-room)
{.is-info}