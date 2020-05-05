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

Skunk的编程接口。

<!--more-->

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
