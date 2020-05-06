---
title: Skunk设计 (编程接口)
categories:
  - skunk
tags:
  - skunk
  - design
abbrlink: 972a
date: 2020-05-03 04:17:55
---

Skunk的编程接口 [@leemaster](https://www.github.com/leemaster)

<!--more-->

> 一个基本能用的软件产品的接口应该是清晰且独立的，编程接口应该保持高效的接口语义，让使用者一眼就能看懂并快速学会如何使用。

# Skunk 网络编程

在[Skunk 网络框架概览](https://gearfacoty.github.io/p/7b1c.html) 中已经介绍了Skunk的研发背景和研发目标，是的，我们的很多项目就是通过这只臭鼬来实现我们的网络通信能力的。

## 基本抽象

对于网络编程，一般来说主流的框架都是使用的异步非阻塞模式来组织代码，使用者需要将逻辑分散到各个事件中去，依靠框架的事件通知来触发用户注册的处理函数，实现业务逻辑。那么Skunk也不例外，Skunk完整的实现了`Reactor`模式，关于`Reactor`模式的介绍可以参考这篇Blog[Skunk原理 (Reactor模型)](https://gearfacoty.github.io/p/a21.html)。





```cpp
int main(){
  EventLoop * bossLoop = new EventLoop(4);
  EventLopp * workerLoop = new EventLoop(8);
  InetAddress4 * address = new InetAddress("0.0.0.0",8080);

  workerLoop -> Codec(new SimpleCodec());
  workerLoop -> OnRead(new Handler());
  workerLoop -> OnWrite(new Handler());
  workerLoop -> OnError(new Handler());

  SkunkServer * server = SkunkServer(
    bossLoop,
    workerLoop,
    address,
    2000 // timeout
  );

  auto future = server -> Start();

  future.get();
}
```
