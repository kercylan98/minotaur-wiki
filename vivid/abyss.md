---
title: 消息深渊
description: 
published: true
date: 2024-08-08T09:45:03.357Z
tags: vivid, actor system, pub-sub, dead letter
editor: markdown
dateCreated: 2024-08-08T09:45:03.357Z
---

# 消息深渊

在 Vivid 中，无法被传递的消息将会被投递到消息深渊中，这些消息通常在其他地方也被称为死信。它可能在本地（例如 Actor 正在终止）中或者在网络中发生。

消息深渊应该主要被用于调试用途，并且整个系统中应尽量减少这种情况的发生。

# 订阅

在消息进入消息深渊中，将会向所有 `vivid.AbyssTopic` 主题的订阅者发送 `*vivid.OnAbyssMessageEvent` 消息，该消息可在网络中被传输。例如在集群中，那么所有节点的订阅者均能收到。

```go
ctx.Subscribe(vivid.AbyssTopic)
```

通过订阅消息深渊，我们可以对系统的异常进行分析，并且可以针对性的尝试对于消息进行重试或其他方案。