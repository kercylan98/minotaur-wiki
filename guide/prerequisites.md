---
title: 先决条件
description: 开始使用 Minotaur 前的必要条件
published: true
date: 2024-08-14T11:35:48.561Z
tags: 
editor: markdown
dateCreated: 2024-06-19T03:06:28.535Z
---

# Go
Minotaur 目前需要 [Go](https://go.dev/) 版本 [1.22](https://go.dev/doc/devel/release#go1.22.0) 或以上。

# Generics 泛型
在 Minotaur 的设计中包含了大量泛型的使用，务必先了解相关内容：

[Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)

# Protobuf
在 Minotaur 的核心设计中，使用了 [`Protocol Buffers`](https://protobuf.dev/getting-started/gotutorial/) 作为网络传输序列化。当需要使用网络通讯或集群功能时，您需要了解它，并使用它来定义消息体。

# Actor Model
Minotaur 中采用了 Actor Model 作为架构思想，使用前请务必了解：

[wikipedia: Actor Model](https://zh.wikipedia.org/wiki/%E6%BC%94%E5%91%98%E6%A8%A1%E5%9E%8B)

# 获取 Minotaur

通过 `go get` 获取从 `Github` 到 [Minotaur](https://github.com/kercylan98/minotaur) ：

```shell
go get -u github.com/kercylan98/minotaur
```

如果您位于中国内陆且无法访问 `Github`，可以尝试通过 [`GOPROXY.IO`](https://goproxy.io/docs/getting-started.html) 代理进行获取。
