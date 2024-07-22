---
title: 资源控制器
description: 以进程为核心的资源控制器
published: true
date: 2024-07-22T15:23:34.688Z
tags: 
editor: markdown
dateCreated: 2024-07-22T10:54:59.644Z
---

# 介绍
资源控制器(ResourceController)是 Minotaur 提供的低级功能，它以进程为基本单元，提供了针对进程的管理及通讯功能。同时，资源控制器还支持分布式网络通讯及集群模式。它通过 GRPC 和 ProtoBuffer 进行网络通讯，并使用基于流言算法(Gossip)实现的 memberlist 对集群的一致性进行管理，

虽然大多数情况下，应该无需使用到资源控制器，但是通过它，可以方便快速的构建分布式应用。

目前它被用广泛在 vivid 的 ActorSystem 以及 future 的实现中。

> 资源控制器的网络功能默认是通过 ProtoBuffer 进行消息序列化，如果需要发送跨网络消息，需要确保消息为 ProtoBuffer 定义，否则请自定义编解码器！
{.is-warning}

# 资源控制器
资源控制器是管理所有进程及网络解析器的中央管理器，它是并发安全的。如果需要创建一个资源控制器，可以通过 `prc.NewResourceController` 函数进行创建。

资源控制器目前包含了一部分的配置，它由 `prc.ReousrceControllerConfigurator` 进行编辑，具体配置如下：

## 设置物理地址
默认情况下，资源控制器的物理地址为 `localhost`，它将是一个仅支持自身单机的控制器，如果需要对网络开放，我们需要指定其物理地址。

```go
func (c *ResourceControllerConfiguration) WithPhysicalAddress(address PhysicalAddress) *ResourceControllerConfiguration 
```

## 设置日志提供者
在资源控制器中，会记录一些如进程注册、取消注册的日志，如果需要定义日志记录器，可以通过该函数来指定基于 `slog.Logger` 实现的日志记录器。

```go
func (c *ResourceControllerConfiguration) WithPhysicalAddress(address PhysicalAddress) *ResourceControllerConfiguration 
```

## 设置集群名称
集群名称目前仅用作检查进程归属，暂无它用。

```go
func (c *ResourceControllerConfiguration) WithClusterName(name ClusterName) *ResourceControllerConfiguration
```

# 进程
进程是资源管理器中的最小单元，每一个进程包含了初始化、关闭，以及对用户及系统消息的处理。当我们实现 `prc.Process` 接口时，便可以轻松的交由资源管理器进行管理。

