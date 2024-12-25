---
title: ActorContext 深入
description: 理解并深入 ActorContext，感受高效并发编程
published: true
date: 2024-12-25T09:40:56.233Z
tags: actor, vivid, actor system, actor context, minotaur
editor: markdown
dateCreated: 2024-12-25T09:00:44.293Z
---

# 前言

在阅读完前面的章节后，相信你已经对 `Vivid` 的使用方法和设计理念有了初步了解。接下来，我们将深入探讨其核心部分：`ActorContext`，它是实现高效、灵活的消息处理与任务调度的关键所在。通过对 `ActorContext` 的进一步理解，你将能解锁更多强大功能，掌握如何在实际开发中运用它，提升系统的可扩展性和性能，并避免潜在的设计陷阱。让我们一起揭开这些强大特性的神秘面纱！

**💗 值得了解的一些信息**  
- **消息处理是线程安全的**：在 `Actor.OnReceive` 中处理消息时，Actor 的模型本身已经保证了线程安全，因此无需使用锁等传统的并发控制机制。  
- **单一职责**：每个 Actor 应该只关注一个单一的职责，避免让它承担过多的功能，这样更有利于扩展和维护。  
- **失败隔离**：Actor 模式的设计天然支持失败隔离（Failure Isolation）。当某个 Actor 出现异常时，其影响可以被限制在自身，利用 Supervisor 策略可以控制失败的处理方式。  
- **逐步扩展**：Actor 模型适合逐步扩展。如果需要提高系统的并发能力，可以轻松地通过增加 Actor 来分担负载。  
- **跨网络通讯是简单的**：需要跨网络通讯仅需要将消息使用 Protobuf 进行定义便可轻松支持。  

**💥 使用前需注意的一些警告**  
- **避免字段导出**：不要轻易将 Actor 的内部字段或方法暴露给外部，否则外部调用可能导致线程安全问题，除非你非常明确其影响并能够接受后果。  
- **避免暴露上下文**：切勿将 ActorContext 直接暴露或传递给外部逻辑。这样做可能会破坏 Actor 的隔离性和线程安全特性。  
- **消息格式的约束**：确保传递给 Actor 的消息是可预期的格式和内容，否则可能引发不可控的行为。  
- **长时间阻塞操作**：在 Actor 内部避免执行长时间阻塞操作，如网络请求或 IO 操作。这可能导致消息处理队列阻塞，从而影响系统性能。  
- **资源泄露**：当一个 Actor 的生命周期结束时，确保它拥有的资源（如文件句柄、数据库连接等）都已正确释放。  
- **消息是不可变的**：为了确保消息处理的安全性和可预测性，传递给 Actor 的消息对象应设计为不可变（即：仅只读而不写）。  
- **避免共享状态**：Actor 内部的数据应与外部隔离，尽量避免使用共享内存的方式在 Actor 间传递数据，以免引入竞态条件。  
- **避免循环依赖**：设计消息流时，注意避免 Actor 间产生循环依赖，这会导致消息无法处理完成，甚至系统死锁。  

> 无需产生任何压力，`Vivid` 是便于使用的，你只需要知晓拥有这项功能，便学会了这项功能的使用！
{.is-success}

# 常用信息获取

在开发过程中，我们很难说不会需要获取到 Actor 或 ActorSystem 等更多的信息。为此，我们提供了一些函数用于获取这些基本且必要的内容。

## 获取 ActorSystem

在 ActorContext 中，我们常常会需要获取其所属的 ActorSystem，这时可通过函数 `System()` 来安全的获取，例如：

```go
ctx.System()
```

## 获取 ActorRef

通常，我们获取 ActorRef 的方式是创建时候获得它或通过消息传递获得它，在 ActorContext 中，可简单的通过 `Ref()` 函数进行获取，例如：

```go
ctx.Ref()
```

## 获取物理地址

在一些场景中，我们也许需要知道 Actor 所属 ActorSystem 的物理地址，这时候便可以通过 `PhysicalAddress()` 函数进行获取，例如：

```go
ctx.PhysicalAddress()
```

> 物理地址代表了其 ActorSystem 的物理地址，同一 ActorSystem 下所有 Actor 物理地址相同。
{.is-info}

## 获取逻辑地址

逻辑地址是 Actor 在 ActorSystem 中的定位符，它是 ActorSystem 内唯一的，当我们需要获取时，可通过 `LogicalAddress()` 函数进行获取，例如：

```go
ctx.LogicalAddress()
```

## 获取子 ActorRef

有时，我们需要广播或等行为时，需要获取到 ActorContext 下的子 Actor 引用，这时候便可以通过 `Children()` 函数获取，例如：

```go
ctx.Children()
```

> 需要注意的是，该函数仅能够获取到该 ActorContext 下的子 ActorContext 的 ActorRef，无法获取孙级引用，目前来说，Vivid 也不提供孙级引用的获取。
{.is-info}

# 待定


