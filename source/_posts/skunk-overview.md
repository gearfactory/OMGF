---
title: Skunk 网络框架概览
categories:
  - skunk
tags:
  - skunk
abbrlink: 7b1c
date: 2020-05-03 03:40:58
---

Gear Factory 基础组件之 Skunk。

<!--more-->

# Skunk 

Skunk 是一个网路通信框架，这个轮子主要的工作就是为我们后续的各种分布式轮子提供网络通信能力，虽然我们后续的分布式轮子可能直接应用都是Crab系列RPC框架-_-。但是Crab需要Skunk提供底层的网络通信能力，各位小伙伴一直觉得用一个现成的网络库就好了，比如libevent，但是我们后来觉得既然造轮子学知识，那就从头开始梭哈吧。

## 背景

Skunk库的作者（[@leemaster](https://www.github.com/leemaster)）目前在美团点评任职系统工程师。突然有一天，这位同学觉得写Web系统的CURD没有意思了，我们先听听他的造轮子宣言`‘Java太限制我发挥了，瘸腿泛型这是人用的东西吗？’` -_- ，所以这位同学自称花了十几天学会了C++就开始写Skunk了，虽然Skunk断断续续开发了将近两个月，但是我们就不要拆他台了，毕竟我们还是要吃外卖的。

那么Skunk是一个什么东西呢？是的Skunk是一个基于Reactor模型实现的网络通信框架，因为我们后续项目规划基本都是分布式的系统，而分布式的系统中的一个设计要点就是协调和同步，如果系统需要协调和同步，在我们自己的电脑上，我们可以用`mutex`、`condition`这些同步原语来进行进程间通信，当然还有Go语言中的`channel`来实现`CSP`模型进行进程间通信。分布式系统的协调和同步在实现上，其实还是网络通信（RPC），只要有了网络中的通信能力，我们就能直接在分布式的系统中模拟系统总线等等等这些单机中的硬件基础设施。

就这样，我们决定自己写一个网络框架，从分布式系统的基础设施做起，一步一步的完成全部的Gear Factory分布式相关的项目。

## 目标

Skunk的定位是一个用在分布式系统中提供通信能力的网络通信框架，那么我们只需要关注TCP协议就好了，完全不用考虑UDP，虽然我们不想承认我们确实没有这个水平在UDP上实现可靠通信，你们就认为我们没有时间写UDP吧-_-/。那么Skunk的设计目标就是这些咯：

1. 完整的TCP服务端客户端抽象
2. 基于Reactor实现网络事件处理
3. 提供人用起来比较舒服的编程接口


## Skunk 的基本使用

TODO: 

## Skunk 需要大家的帮助

Skunk目前是一个基于线程模型实现的Reactor，而且内部的内存申请还是使用C++自带的`new`、`delete`组合，本着造轮子也是学习的想法，我们觉得Skunk应该支持下面的这两个特性：

1. 实现自己的内存池
2. 利用协程实现更高一层的吞吐

不过目前Skunk应该能满足Gear Factory的日常造轮子需求，所以大家目前没有投入实现来关心这两个feature，所以各位看官姥爷要不提点PR，我们一起帮Skunk的性能更上一层楼吧！