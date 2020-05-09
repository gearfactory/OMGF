---
title: Skunk原理（Reactor模型）
categories:
  - skunk
tags:
  - skunk
  - theory
abbrlink: a21
date: 2020-04-19 06:26:53
---

关于Reactor模式 [@leemaster](https://www.github.com/leemaster)

<!--more-->

> 让callback替代所有的同步，就像让音乐代替所有话语声一样。

# Reactor

这里我们应该感谢`Doug Schmidt`总结出来的这个模式，在[Skunk原理 (I/O)](https://gearfacoty.github.io/p/ae7b.html)一章中我们说到，I/O是一个需要陷入内核的操作，毕竟和硬件设备打交道就不是用户态程序要关心的事情了，用户态和内核态其实就像分布式系统一样，是两部分，那么一个系统通过RPC调用另外一个系统一定是遵守Request-Response模式的，所以在Request发出去的时候，Response没有回来的时候，用户态的程序需要进行等待，这个等待就叫做阻塞。在客户端编程的时候，这个阻塞其实没有什么事情，毕竟我们发出了HTTP请求，一定要等着响应回来，这个是符合程序语义的，性能优化啥也没有什么特别的要求去优化这个东西，因为性能优化的前提就是不能破坏程序本身的顺序语义。

Reactor其实主要解决的还是服务端，也就是Server端的吞吐问题，或者说把非阻塞异步模型给固定成了一个设计模式，我们只要按着这个设计模式搞，写出来的系统性能应该还不错。

## 拥抱异步

在上文说到的这个Request-Response模式，其实就是典型的同步编程模式，我们把逻辑直接组织在了顺序的代码里面，形成了一个指令序列一条一条指令执行就好了。现在我们换个想法，如果我们的请求发出去了，等他真的响应的时候我在处理就好了，不用在响应回来之前做无谓的等待，系统也可以做其他的事情，那么怎么用现有的知识来理解这个事呢？

我们完全可以认为响应是一个事件，就好像CPU的时钟震荡一样，在固定的的时刻触发一个震荡，当这个事件发生的时候我们调用处理这个事件的处理函数就好了。

那么假设我们原来的代码逻辑是这样的

```cpp
Response respone = client -> Request(post);
// do something...
```

那么现在我们想象有一个全局的HashTable，专门用来注册事件和其处理函数，那么这个程序就变成了这个样子

```cpp
HashTable eventMap;
ResponseHandler = [](const char * data){
  // do somethindg...
}

eventMap.put("Response", ResponseHandler);
Status status = client -> Request(post);
```

没错，这个就是非常典型的异步编程模式，在同步编程模式中，我们发送了Request必须无限等待Response的返回，在Response真正返回之前，只能一直阻塞在`client -> Request` 这个调用上，但是现在，我们只需要注册好回调函数，然后剩下的事情交给系统自己去处理，反正他处理好了就会使用你注册好的事件处理器来处理这个这个事件。

## 异步能带来什么

就上文我们看到的异步模型，除了多出来的代码以外好像没有看到什么特别的好处，其实好处之一就是我们将原来需要阻塞等待的事情交给库去关心，在应用层只需要关心什么事件需要处理，以及事件如何处理，在关心的事件从发生到真正处理有一段时间，其实CPU是没有事情可以做的，异步最大的好处就是提升了系统的吞吐量，因为我们都知道I/O的操作是非常耗时的（CPU的速度是远远高于I/O的速度的，网络I/O更加难受）。当然还有一个好处就是我们可以做一些本来就是异步的事情，比如定时和超时、信号处理等等等。

## 事件循环

在上面我们大概看到了一个不太恰当的异步编程模型，但是我们已经找到了异步编程的关注核心`事件`。还记得TCP的三次握手吧，TCP的链接建立只有到了第三次握手成功后才能完成，那么如果我们有个库可以帮助我们在前两次握手的时候做一些其他的事情，不用等待前两次握手完成，是不是可以处理更多的事情了呢？类似的，定时和超时，只需要在定时和超时事件发生的时候我们再处理，在这之前可以做很多事情，比如课间时间，你可以很随意，但是上课那个铃声响起来，你就需要去上课了。

那么我们怎么让事件和处理分离呢？我们用同步的方法模拟一下吧，比如我们关心的事件是`TIMEOUT`事件，在`TIMEOUT`事件发生之前，我们就让我们的程序在这里等着。

```cpp
Timeout timeout(500ms);
EventHandler handler = [](Time time){
  cout << time 
}

while (!timeout.expired()){} // waiting  TIMEOUT
handler(now()); // hanlde timeout event 
```

在最后一行代码中，我们用同步模拟了一个`TIMEOUT`事件，然后事件处理函数就会进行处理。到这里已经基本说明白异步和同步的差别，还有事件为核心的编程思想。那么问题来了，我们怎么让程序真的异步起来呢？这里用到了一个思想就是`事件循环`，我们就想让程序自动帮我们回调我们的事件处理函数，那么这里就涉及到了两个角色，一个角色我们称为`事件注册者`，一个角色我们称为`事件监听者`，事件注册者注册事件和事件处理函数，事件监听者在事件真实发生的时候触发事件注册者注册的事件处理函数，问题就解决了，只要让这两个角色并行的跑起来，事件注册者注册完了事件以后又可以做别的事情，剩下的事情交给事件监听者去处理就好了。为了模拟两个并行的角色，自然而然用线程去处理就好了。

我们先看事件注册者怎么做

```cpp
Thread Register([](){
  Timeout timeout(500ms);
  // register event
  Listener -> OnTimeout(&timeout, [](Time time){

  });
  cout << "register complete";
});
```

然后我们再看事件监听者怎么做

```cpp
Thread Listener([](){
  for(;;){
    if (timeout.expired()){
      TimeoutCallback(now); // Trigger the callback 
    }
  }
});
```

其中事件监听者运行的这个for循环，就被叫做`事件循环`，事件循环主要的工作就是检查事件发生没有，发生了就通知回调，没发生就接着等着发生就行了。

## Reactor 模型

到这里应该已经清楚什么是异步，什么是事件循环了，那么就引出我们的Reactor模型吧。

### 系统中的事件

我在系统开发遇到的事件，大概可以分为以下三类

1. 网络事件
2. 时间事件
3. 系统事件

网络事件其实大家应该都知道，在TCP通信的每个阶段都触发一个事件，比如链接建立的时候，这个事件叫做`OnConnect`事件，当监听的描述符可读`OnReadable`事件，描述符可写（系统缓存有空间）`OnWriteable`事件。

时间事件，其实也就两种，超时事件`OnTimeout`，定时`OnInterval`事件。

系统事件，其实有很多，一般我们关心的无非就是系统调用是不是跪了`OnError`事件，是不是有外部信号`OnSignal`事件。

### 事件模型

异步和事件循环我们清楚了，事件分类我们也大概知道了，现在就可以真正的请出今天的主角——Reacotr了，先来张类图吧。

![](/images/reactor1.png)

在图中，我们看到`Reactor`其实就三个方法，分别是`Register`,`Unregister`,`HandleEvents`，这三个方法其实就是注册事件，取消事件，处理事件。

这里的处理事件，其实就是事件分发，根据不同的事件类型，分发到不同的事件处理器上，也就是`EventHandler`上，那么图中的这个`Handle`是什么呢？按照wikipedia的解释，这个东西叫做句柄。我觉得很多的句柄解释都很麻烦，如果用代码解释一下就很简单了。

```cpp
std::map<string, std::vector<int> > handleMap;
handleMap.insert(std::pair<string, std::vector<int>>("I/O", vector<int>{STDIN,STDOUT,STDERR}))
handleMap.insert(std::pair<string, std::vector<int>>("TIME", vector<int>{TIMERFD1))
```

其实句柄是某一类事件在系统中标示的实例，比如I/O系统中的句柄是文件描述符号，TIME系统中的句柄就是时间描述符号，这里其实就是面向对象的思考方式，这样看句柄就比较简单了。

那我们的异步事件编程有了Reactor模式就比较简单了。

```cpp
class ReadHandler: EventHandler{
  public:
    ReadHandler(int fd): fd_(fd){}
    void HandleEvent(int fd) override{
      // handle the fd 
    }
    int GetHandle() override{
      // return the fd
    }
  private:
    int fd_
};
Reactor.Register(IO_READ, ReadHandler(STDIN));
Reactor.HandleEvents();
```

## Reactor概念

### 反应堆模型

上面我们看到的Reactor是一个单线程程序，一个Reactor就是一条线程，反应堆其实就是一堆Reactor组装到一起，这不就成一堆了嘛。

### 网络编程怎么用

网络编程中，其实分为两部分，一部分是绑定端口监听这种流程化的固定操作，还有一部分是I/O操作，也就是从网卡中拿数据并且进行处理的过程。巧就巧在Linux对于一个socket文件描述符，绑定监听后，新建一个链接也属于一个读事件，那直接弄成两个Reactor跑就行了，也就是一个Reactor专心的accept请求，一个Reactor专心的处理I/O

### 时间和信号呢

Linux的系统调用有`timerfd`和`signalfd`两套系统调用，完全可以把时间事件和信号转换成为文件描述符上的读事件。

### Reactor为什么可以提升吞吐

因为网络编程中，一方面要接受链接建立，一方面要处理这个连接上的数据，也就是I/O，那么如果使用Reactor的话，可以同时的进行accept和read，相当于并行的，而在异步之前，accpet和read是串行的，自然同时能处理的请求就少了，所以异步提升吞吐就在这里，但是如果CPU密集的程序，这个其实没啥特比的大的提升，因为I/O的等待不是瓶颈，反而是CPU的计算是瓶颈了。