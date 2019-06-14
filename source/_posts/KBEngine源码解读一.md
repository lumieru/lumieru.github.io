---
title: KBEngine源码解读一
date: 2019-06-14 16:31:27
tags: [ServerFramework, KBEngine]
categories: GameServer
---
## 一. libs/common部分

**MemoryStream：**

将常用数据类型二进制序列化与反序列化，内部封装了一个`std::vector<uint8>`。使用方法:
```cpp
MemoryStream stream; 
stream << (int64)100000000;
stream << (uint8)1;
stream << (uint8)32;
stream << "kbe";
stream.print_storage();
uint8 n, n1;
int64 x;
std::string a;
stream >> x;
stream >> n;
stream >> n1;
stream >> a;
printf("还原: %lld, %d, %d, %s", x, n, n1, a.c_str());
```

**Tasks:**

任务`Task`的管理类，内部封装了`std::vector<Task *>`，通过调用`process()`来遍历所有`Tash`的`process()`虚函数。

**TimersT&lt;T&gt;:**

定时器管理类，内部用小顶堆管理所有的定时器。通过
```cpp
TimerHandle add(TimeStamp startTime, TimeStamp interval,
                TimerHandler* pHandler, void * pUser);
```
来新增一个定时器，返回的`TimerHandle`是新增定时器的句柄，可以控制相应的定时器。用户需要继承`TimerHandler`类，并重载
```cpp
virtual void handleTimeout(TimerHandle handle, void * pUser)
```
虚函数来实现定时器的回调。

内部预定义了两个`TimersT<T>`类
```cpp
typedef TimersT<uint32> Timers;
typedef TimersT<uint64> Timers64;
```

<!-- more -->

## 二. libs/network部分

**Address:**

封装了ip和port

**Packet:**

发送和接收的包的最小单位，继承自`MemoryStream`。在发送的时候`Packet`都是嵌套在`Bundle`的内部来使用的。

**TcpPacket:**

代表一个收到的TCP包，继承自`Packet`，这个包只是recv收到的字节流，并不是上层协议中的消息(Message)。

**UDPPacket:**

代表一个收到的UDP包，继承自`Packet`，这个包只是recv收到的字节流，并不是上层协议中的消息(Message)。

**Bundle:**

代表要发送的包的集合，内部有`std::vector<Packet*>`数组。内部的`Packet`根据TCP和UDP不同，有不同的最大字节数限制。TCP的一个`Packet`最多包含1460个字节，UDP是1472字节。也就是说序列化到`Bundle`中的数据会自动分包的。

**BundleBroadcast:**

继承自`Bundle`，可以方便的处理如:用UDP向局域网内广播某些信息，并处理收集相关信息。

**EndPoint:**

抽象一个Socket及其相关操作，隔离平台相关性。

**FixedMessages:**

从“server/messages_fixed_defaults.xml”中读入消息的定义，维护了一个消息名字到消息id的映射。

**MessageHandlers:**

每个`MessageHandler`类对应一个消息的处理，通过重载下面的虚函数来处理：
```cpp
virtual void handle(Channel* pChannel, MemoryStream& s)
```
`MessageHandlers`维护`MessageID` -> `MessageHandler`的映射。

在把`MessageHandler`加入到`MessageHandlers`时，如果是固定消息，那么`MessageID`是从`FixedMessages`中的设置中来的，否则就一个递增得到的（会避开固定消息的id）。

以`Baseapp`为例，所有的消息都是用宏定义在baseapp_interface.h/cpp中的，每个消息需要实现一个`MessageHandler`的子类和一个`MessageArgs`的子类（用来处理消息的参数）。并且会创建相应的实例，添加到`messageHandlers`(是Network::MessageHandlers的实例)中去。然后在这些子类的handle中又会去调用`Baseapp`中的相同名字的方法，最后这个消息其实是在`Baseapp`的同名方法中处理的。

**PacketReceiver:**

用来收Packet，收到后会转给PacketReader来处理。

**PacketReader:**

会利用MessageHandlers把包转成对应的Message，即相应的函数调用。

**PacketSender:**

用来发送Packet的。

**Channel:**

抽象一个Socket连接，每个EndPoint都有其对应的Channel，它代表和维护一个Socket连接，如缓冲Packet，统计连接状态等。
提供一个ProcessPackets(MsgHanders* handers)接口处理该Channel上所有待处理数据。

**EventPoller:**

用于注册和回调网络事件，具体的网络事件由其子类实现processPendingEvents产生，目前EventPoller有两个子类: EpollPoller和SelectorPoller，分别针对于Linux和Windows。
通过bool registerForRead(int fd, InputNotificationHandler * handler);注册套接字的可读事件，回调类需实现InputNotificationHandler接口。

**EventDispatcher:**

核心类，管理和分发所有事件，包括网络事件，定时器事件，任务队列，统计信息等等。
它包含 EventPoller Tasks Timers64 三个组件，在每次处理时，依次在这三个组件中取出事件或任务进行处理。Epoll的最小等待时间是到下一个timer的触发时间，最大等待时间目前是0.1秒。这样可以保证Timer能被精确的触发。

**ListenerReceiver/PacketReceiver:**

继承自InputNotificationHandler，分别用于处理监听套接字和客户端套接字的可读事件，通过bool registerReadFileDescriptor(int fd, InputNotificationHandler * handler); 注册可读事件。

**NetworkInterface:**

维护管理监听套接字，创建监听套接字对应的ListenerReceiver，并且通过一个EndPoint -> Channel的Map管理所有已连接套接字，提供一个processChannels(MsgHandlers* handers)接口处理所有Channel上的待处理数据。这一点上，有点像NGServer:ServiceManager。

**网络部分的核心类结构图**
![](http://blog.sensedevil.com/image/kbengine1_1.png)

**各个组件之间的关系图**
![](http://blog.sensedevil.com/image/kbengine1_2.png)