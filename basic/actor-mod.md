---
title: Actor 模组化
description: 构建大型复杂 Actor 的利器
published: false
date: 2024-06-21T06:13:28.417Z
tags: actor, mod
editor: markdown
dateCreated: 2024-06-21T06:13:28.417Z
---

# 前言
我们在项目开发过程中，发现在对于某些特定场景下，一个 `Actor` 的生命周期会非常长久，且涉及的功能会很复杂。

假设我们正在开发一个游戏，下意识的我们可能会希望存在这些 `Actor`：
 - 维护玩家长连接的 `Actor`；
 - 维护账号基本信息等信息的 `Actor`；
 - 维护玩家背包数据的 `Actor`；
 - 维护玩家每日任务数据的 `Actor`，该 `Actor` 可能会与背包 `Actor` 交互，例如发放物品等行为；

如果我们真的采用了这样的设计，似乎开发过程将会陷入一种难以言说的折磨之中。

如果每一个玩家分配一个对应功能的 `Actor`，将会造成一个玩家拥有大量的 `goroutinue`，并且我们的大多数的消息都将需要通过 `Ask` 消息来保证同步。举个例子，假设**玩家登录后的流程**如下：
> `Login` => `SyncAccountInfo` => `SyncBagInfo & SyncDailyTask`

如果