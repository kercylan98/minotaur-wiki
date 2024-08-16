---
title: 运行时配置表
description: 加载到内存的运行时配置表
published: true
date: 2024-08-16T13:08:22.226Z
tags: minotaur, cli, config, table, xlsx, json, lua
editor: markdown
dateCreated: 2024-08-15T17:22:11.800Z
---

# 介绍

在游戏开发（不仅限）过程中，经常会有大量的配置需要由产品经理或策划人员进行配置，大多数非技术人员可能更期望于使用 Excel 进行数值的配置，这样不仅可以更方便的可视化，同时提供的大量函数也能很方便的进行数值计算，但是在这种情况下，非常不利于程序进行解析。这时候我们便需要一个转换来进行这个行为。

在 Minotaur AX CLI 中，提供了配置表相关的功能，您可以使用其来方便地将配置结构转换为编程代码，也可以将配置数据转换为易于解析的 JSON 和 LUA 代码。

# 开始

在客户端工具中，配置表相关的命令将在 `table` 子命令下，目前仅提供了基于 `xlsx` 的配置转换及代码生成。

可以通过 `ax table help` 命令来获取到帮助信息：

![ax-table-help.png](/ax-table-help.png)

> 如果您还没有安装 `Minotaur AX CLI`，可从[此处](/zh/minotaur-ax/client)跳转。
{.is-info}

# XLSX 配置表

基于 `XLSX` 文件的配置表相关命令则位于 `ax table xlsx` 命令下：

![ax-table-xlsx-help.png](/ax-table-xlsx-help.png)

## 数据表模板

第一步，我们当然是应该先了解数据表的结构应该是什么样子的：

```shell
ax table xlsx template -o template.xlsx
```

通过该命令，我们便可以在运行目录下生成一个模板文件，我们通过它来了解！

## 无索引数据表

无索引数据表通常是用于做全局配置的，它每个字段仅包含一个值。

> ![ax-table-xlsx-template-none-index.png](/ax-table-xlsx-template-none-index.png)

## 有索引数据表

有索引数据表应该是用的最多的，它可以包含至少以个索引，并支持多行数据。

> ![ax-table-xlsx-template-index.png](/ax-table-xlsx-template-index.png)

## 字段定义

在 XLSX 数据表中，字段的定义可以是基本类型、数组、结构体，而这里的基本类型则是 `go` 语言中支持的基本类型。

在定义数组时候，我们仅需要在类型前方加上 `[]`，便可表示该字段为数组字段。

而结构体的定义当然也非常简单，我们仅需要通过 `{ name: type }` 的语法即可完成定义！

例如一个复杂的嵌套结构体：
```text
{ id: int, name: string, info: { lv: int, exp: { mux: int, count: int }}}
```

## 数据定义

目前的数据表示方法是采用的 `JSON`，当然我们对于 `string` 允许它不使用双引号进行包裹。

> 未来将考虑兼容 `Lua` 格式的数据定义方式。
{.is-info}


## 忽略和终止

当我们的Sheet名称、配置名称、字段名称或描述、数据描述、名称等内容前方携带 `#` 前缀时，该数据表、字段或数据行将会被忽略。
