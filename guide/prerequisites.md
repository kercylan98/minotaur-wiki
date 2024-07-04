---
title: 先决条件
description: 开始使用 Minotaur 前的必要条件
published: true
date: 2024-07-04T07:57:31.474Z
tags: 
editor: markdown
dateCreated: 2024-06-19T03:06:28.535Z
---

# Go
Minotaur 目前需要 [Go](https://go.dev/) 版本 [1.22](https://go.dev/doc/devel/release#go1.22.0) 或以上

# Generics 泛型
在 Minotaur 的设计中包含了大量泛型的使用，务必先了解相关内容

[Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)

# 获取 Minotaur

通过 `go get` 获取从 `Github` 到 [Minotaur](https://github.com/kercylan98/minotaur) ：

```shell
go get -u github.com/kercylan98/minotaur@develop
```

如果您位于国内且无法访问 `Github`，可以尝试通过 [`GOPROXY.IO`](https://goproxy.io/docs/getting-started.html) 代理进行获取。
