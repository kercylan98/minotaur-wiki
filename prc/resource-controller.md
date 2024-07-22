---
title: 资源控制器
description: 以进程为核心的资源控制器
published: true
date: 2024-07-22T10:54:59.644Z
tags: 
editor: markdown
dateCreated: 2024-07-22T10:54:59.644Z
---

# 介绍
资源控制器(ResourceController)是 Minotaur 提供的低级功能，它以进程为基本单元，提供了针对进程的管理及通讯功能。同时，资源控制器还支持分布式网络通讯及集群模式。它通过 GRPC 和 ProtoBuffer 进行网络通讯，并使用基于流言算法(Gossip)实现的 memberlist 对集群的一致性进行管理，

虽然大多数情况下，应该无需使用到资源控制器，但是通过它，可以方便快速的构建分布式应用。

目前它被用广泛在 vivid 的 ActorSystem 以及 future 的实现中。

# 进程
进程是资源管理器中的最小单元，每一个进程包含了初始化、关闭，以及对用户及系统消息的处理。当我们实现 `prc.Process` 接口时，便可以轻松的交由资源管理器进行管理。