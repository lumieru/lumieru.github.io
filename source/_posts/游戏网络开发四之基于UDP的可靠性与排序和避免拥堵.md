---
title: 游戏网络开发四之基于UDP的可靠性与排序和避免拥堵
date: 2019-05-20 11:45:49
tags: [GafferOnGames, UDP]
categories: Multiplayer
---
# 原文

[原文出处](https://gafferongames.com/post/reliability_ordering_and_congestion_avoidance_over_udp//)

## Introduction

Hi, I’m [Glenn Fiedler](https://gafferongames.com/about) and welcome to [**Networking for Game Programmers**](https://gafferongames.com/categories/game-networking/).

In the [previous article](https://gafferongames.com/post/virtual_connection_over_udp/), we added our own concept of virtual connection on top of UDP. In this article we’re going to add reliability, ordering and congestion avoidance to our virtual UDP connection.

## The Problem with TCP

Those of you familiar with TCP know that it already has its own concept of connection, reliability-ordering and congestion avoidance, so why are we rewriting our own mini version of TCP on top of UDP?

The issue is that multiplayer action games rely on a steady stream of packets sent at rates of 10 to 30 packets per second, and for the most part, the data contained is these packets is so time sensitive that only the most recent data is useful. This includes data such as player inputs, the position, orientation and velocity of each player character, and the state of physics objects in the world.

The problem with TCP is that it abstracts data delivery as a reliable ordered stream. Because of this, if a packet is lost, TCP has to stop and wait for that packet to be resent. This interrupts the steady stream of packets because more recent packets must wait in a queue until the resent packet arrives, so packets are received in the same order they were sent.

What we need is a different type of reliability. Instead of having all data treated as a reliable ordered stream, we want to send packets at a steady rate and get notified when packets are received by the other computer. This allows time sensitive data to get through without waiting for resent packets, while letting us make our own decision about how to handle packet loss at the application level.

It is not possible to implement a reliability system with these properties using TCP, so we have no choice but to roll our own reliability on top of UDP.

## Sequence Numbers

The goal of our reliability system is simple: we want to know which packets arrive at the other side of the connection.

First we need a way to identify packets.

What if we had added the concept of a “packet id”? Let’s make it an integer value. We could start this at zero then with each packet we send, increase the number by one. The first packet we send would be packet 0, and the 100th packet sent is packet 99.

This is actually quite a common technique. It’s even used in TCP! These packet ids are called sequence numbers. While we’re not going to implement reliability exactly as TCP does, it makes sense to use the same terminology, so we’ll call them sequence numbers from now on.

Since UDP does not guarantee the order of packets, the 100th packet received is not necessarily the 100th packet sent. It follows that we need to insert the sequence number somewhere in the packet, so that the computer at the other side of the connection knows which packet it is.

We already have a simple packet header for the virtual connection from the previous article, so we’ll just add the sequence number in the header like this:


```
[uint protocol id]  
[uint sequence]  
(packet data…)
```

Now when the other computer receives a packet it knows its sequence number according to the computer that sent it.

<!-- more -->

## Acks

Now that we can identify packets using sequence numbers, the next step is to let the other side of the connection know which packets we receive.

Logically this is quite simple, we just need to take note of the sequence number of each packet we receive, and send those sequence numbers back to the computer that sent them.

Because we are sending packets continuously between both machines, we can just add the ack to the packet header, just like we did with the sequence number:

```
[uint protocol id]  
[uint sequence]  
[uint ack]  
(packet data…)
```

Our general approach is as follows:

  
  

*   Each time we send a packet we increase the _local sequence number_

*   When we receieve a packet, we check the sequence number of the packet against the sequence number of the most recently received packet, called the _remote sequence number_. If the packet is more recent, we update the remote sequence to be equal to the sequence number of the packet.

*   When we compose packet headers, the local sequence becomes the sequence number of the packet, and the remote sequence becomes the ack.

  
  

This simple ack system works provided that one packet comes in for each packet we send out.

But what if packets clump up such that two packets arrive before we send a packet? We only have space for one ack per-packet, so what do we do?

Now consider the case where one side of the connection is sending packets at a faster rate. If the client sends 30 packets per-second, and the server only sends 10 packets per-second, we need _at least_ 3 acks included in each packet sent from the server.

Let’s make it even more complex! What if the packet containing the ack is lost? The computer that sent the packet would think the packet got lost but it was actually received!

It seems like we need to make our reliability system… _more reliable!_

## Reliable Acks

Here is where we diverge significantly from TCP.

What TCP does is maintain a sliding window where the ack sent is the sequence number of the next packet it expects to receive, in order. If TCP does not receive an ack for a given packet, it stops and resends a packet with that sequence number again. This is exactly the behavior we want to avoid!

In our reliability system, we never resend a packet with a given sequence number. We sequence n exactly once, then we send n+1, n+2 and so on. We never stop and resend packet n if it was lost, we leave it up to the application to compose a new packet containing the data that was lost, if necessary, and this packet gets sent with a new sequence number.

Because we’re doing things differently to TCP, its now possible to have _holes_ in the set of packets we ack, so it is no longer sufficient to just state the sequence number of the most recent packet we have received.

We need to include multiple acks per-packet.

How many acks do we need?

As mentioned previously we have the case where one side of the connection sends packets faster than the other. Let’s assume that the worst case is one side sending no less than 10 packets per-second, while the other sends no more than 30. In this case, the average number of acks we’ll need per-packet is 3, but if packets clump up a bit, we would need more. Let’s say 6-10 worst case.

What about acks that don’t get through because the packet containing the ack is lost?

To solve this, we’re going to use a classic networking strategy of using redundancy to defeat packet loss!

Let’s include 33 acks per-packet, and this isn’t just going to be up to 33, but _always_ 33. So for any given ack we redundantly send it up to 32 additional times, just in case one packet with the ack doesn’t get through!

But how can we possibly fit 33 acks in a packet? At 4 bytes per-ack thats 132 bytes!

The trick is to represent the 32 previous acks before “ack” using a bitfield:

```
[uint protocol id]  
[uint sequence]  
[uint ack]  
[uint ack bitfield]  
(packet data…)
```

We define “ack bitfield” such that each bit corresponds to acks of the 32 sequence numbers before “ack”. So let’s say “ack” is 100. If the first bit of “ack bitfield” is set, then the packet also includes an ack for packet 99. If the second bit is set, then packet 98 is acked. This goes all the way down to the 32nd bit for packet 68.

Our adjusted algorithm looks like this:

  
  

*   Each time we send a packet we increase the _local sequence number_

*   When we receive a packet, we check the sequence number of the packet against the _remote sequence number_. If the packet sequence is more recent, we update the remote sequence number.

*   When we compose packet headers, the local sequence becomes the sequence number of the packet, and the remote sequence becomes the ack. The ack bitfield is calculated by looking into a queue of up to 33 packets, containing sequence numbers in the range \[remote sequence - 32, remote sequence\]. We set bit n (in \[1,32\]) in ack bits to 1 if the sequence number remote sequence - n is in the received queue.

*   Additionally, when a packet is received, ack bitfield is scanned and if bit n is set, then we acknowledge sequence number packet sequence - n, if it has not been acked already.

  
  

With this improved algorithm, you would have to lose 100% of packets for more than a second to stop an ack getting through. And of course, it easily handles different send rates and clumped up packet receives.

## Detecting Lost Packets

Now that we know what packets are received by the other side of the connection, how do we detect packet loss?

The trick here is to flip it around and say that if you don’t get an ack for a packet within a certain amount of time, then we consider that packet lost.

Given that we are sending at no more than 30 packets per second, and we are redundantly sending acks roughly 30 times, if you don’t get an ack for a packet within one second, it is _very_ likely that packet was lost.

So we are playing a bit of a trick here, while we can know 100% for sure which packets get through, but we can only be _reasonably_ certain of the set of packets that didn’t arrive.

The implication of this is that any data which you resend using this reliability technique needs to have its own message id so that if you receive it multiple times, you can discard it. This can be done at the application level.

## Handling Sequence Number Wrap-Around

No discussion of sequence numbers and acks would be complete without coverage of sequence number wrap around!

Sequence numbers and acks are 32 bit unsigned integers, so they can represent numbers in the range \[0,4294967295\]. Thats a very high number! So high that if you sent 30 packets per-second, it would take over four and a half years for the sequence number to wrap back around to zero.

But perhaps you want to save some bandwidth so you shorten your sequence numbers and acks to 16 bit integers. You save 4 bytes per-packet, but now they wrap around in only half an hour.

So how do we handle this wrap around case?

The trick is to realize that if the current sequence number is already very high, and the next sequence number that comes in is very low, then you must have wrapped around. So even though the new sequence number is _numerically_lower than the current sequence value, it actually represents a more recent packet.

For example, let’s say we encoded sequence numbers in one byte (not recommended btw. :)), then they would wrap around after 255 like this:

```
… 252, 253, 254, 255, 0, 1, 2, 3, …
```

To handle this case we need a new function that is aware of the fact that sequence numbers wrap around to zero after 255, so that 0, 1, 2, 3 are considered more recent than 255. Otherwise, our reliability system stops working after you receive packet 255.

Here’s a function for 16 bit sequence numbers:

```cpp
inline bool sequence_greater_than( uint16_t s1, uint16_t s2 )  
{  
    return ( ( s1 > s2 ) && ( s1 - s2 <= 32768 ) ) ||  
           ( ( s1 < s2 ) && ( s2 - s1  > 32768 ) );  
}  
```

This function works by comparing the two numbers _and_ their difference. If their difference is less than 1⁄2 the maximum sequence number value, then they must be close together - so we just check if one is greater than the other, as usual. However, if they are far apart, their difference will be greater than 1⁄2 the max sequence, then we paradoxically consider the sequence number more recent if it is _less_ than the current sequence number.

This last bit is what handles the wrap around of sequence numbers transparently, so 0,1,2 are considered more recent than 255.

Make sure you include this in any sequence number processing you do.

## Congestion Avoidance

While we have solved reliability, there is still the question of congestion avoidance. TCP provides congestion avoidance as part of the packet when you get TCP reliability, but UDP has no congestion avoidance whatsoever!

If we just send packets without some sort of flow control, we risk flooding the connection and inducing severe latency (2 seconds plus!) as routers between us and the other computer become congested and buffer up packets. This happens because routers try _very hard_ to deliver all the packets we send, and therefore tend to buffer up packets in a queue before they consider dropping them.

While it would be nice if we could tell the routers that our packets are time sensitive and should be dropped instead of buffered if the router is overloaded, we can’t really do this without rewriting the software for all routers in the world.

Instead, we need to focus on what we can actually do which is to avoid flooding the connection in the first place. We try to avoid sending too much bandwidth in the first place, and then if we detect congestion, we attempt to back off and send even less.

The way to do this is to implement our own basic congestion avoidance algorithm. And I stress basic! Just like reliability, we have no hope of coming up with something as general and robust as TCP’s implementation on the first try, so let’s keep it as simple as possible.

## Measuring Round Trip Time

Since the whole point of congestion avoidance is to avoid flooding the connection and increasing round trip time (RTT), it makes sense that the most important metric as to whether or not we are flooding our connection is the RTT itself.

We need a way to measure the RTT of our connection.

Here is the basic technique:

  
  

*   For each packet we send, we add an entry to a queue containing the sequence number of the packet and the time it was sent.

*   Each time we receive an ack, we look up this entry and note the difference in local time between the time we receive the ack, and the time we sent the packet. This is the RTT time for that packet.

*   Because the arrival of packets varies with network jitter, we need to smooth this value to provide something meaningful, so each time we obtain a new RTT we move a percentage of the distance between our current RTT and the packet RTT. 10% seems to work well for me in practice. This is called an exponentially smoothed moving average, and it has the effect of smoothing out noise in the RTT with a low pass filter.

*   To ensure that the sent queue doesn’t grow forever, we discard packets once they have exceeded some maximum expected RTT. As discussed in the previous section on reliability, it is exceptionally likely that any packet not acked within a second was lost, so one second is a good value for this maximum RTT.

  
  

Now that we have RTT, we can use it as a metric to drive our congestion avoidance. If RTT gets too large, we send data less frequently, if its within acceptable ranges, we can try sending data more frequently.

## Simple Binary Congestion Avoidance

As discussed before, let’s not get greedy, we’ll implement a very basic congestion avoidance. This congestion avoidance has two modes. Good and bad. I call it simple binary congestion avoidance.

Let’s assume you send packets of a certain size, say 256 bytes. You would like to send these packets 30 times a second, but if conditions are bad, you can drop down to 10 times a second.

So 256 byte packets 30 times a second is around 64kbits/sec, and 10 times a second is roughly 20kbit/sec. There isn’t a broadband network connection in the world that can’t handle at least 20kbit/sec, so we’ll move forward with this assumption. Unlike TCP which is entirely general for any device with any amount of send/recv bandwidth, we’re going to assume a minimum supported bandwidth for devices involved in our connections.

So the basic idea is this. When network conditions are “good” we send 30 packets per-second, and when network conditions are “bad” we drop to 10 packets per-second.

Of course, you can define “good” and “bad” however you like, but I’ve gotten good results considering only RTT. For example if RTT exceeds some threshold (say 250ms) then you know you are probably flooding the connection. Of course, this assumes that nobody would normally exceed 250ms under non-flooding conditions, which is reasonable given our broadband requirement.

How do you switch between good and bad? The algorithm I like to use operates as follows:

  
  

*   If you are currently in good mode, and conditions become bad, immediately drop to bad mode

*   If you are in bad mode, and conditions have been good for a specific length of time ’t’, then return to good mode

*   To avoid rapid toggling between good and bad mode, if you drop from good mode to bad in under 10 seconds, double the amount of time ’t’ before bad mode goes back to good. Clamp this at some maximum, say 60 seconds.

*   To avoid punishing good connections when they have short periods of bad behavior, for each 10 seconds the connection is in good mode, halve the time ’t’ before bad mode goes back to good. Clamp this at some minimum like 1 second.

  
  

With this algorithm you will rapidly respond to bad conditions and drop your send rate to 10 packets per-second, avoiding flooding of the connection. You’ll also _conservatively_ try out good mode, and persist sending packets at a higher rate of 30 packets per-second, while network conditions are good.

Of course, you can implement much more sophisticated algorithms. Packet loss % can be taken into account as a metric, even the amount of network jitter (time variance in packet acks), not just RTT.

You can also get much more _greedy_ with congestion avoidance, and attempt to discover when you can send data at a much higher bandwidth (eg. LAN), but you have to be very careful! With increased greediness comes more risk that you’ll flood the connection.

## Conclusion

Our new reliability system let’s us send a steady stream of packets and notifies us which packets are received. From this we can infer lost packets, and resend data that didn’t get through if necessary. We also have a simple congestion avoidance system that drops from 30 packets per-second to 10 times a second so we don’t flood the connection.

# 译文

[译文出处](http://gad.qq.com/program/translateview/7161834)

翻译：艾涛（轻描一个世界） 审校：黄威（横写丶意气风发）

## 简介

嗨，我是格伦-菲德勒，欢迎来到我的[游戏程序员网络设计](http://gafferongames.com/networking-for-game-programmers/)文章系列的第四篇。

在[之前的文章](http://gafferongames.com/networking-for-game-programmers/virtual-connection-over-udp/)里，我们将我们的虚拟连接的概念加入到UDP之上。

现在我们将要给我们的虚拟UDP连接增加可靠性，排序和避免拥堵。

这是迄今为止底层游戏网络设计中最复杂的一面，因此这将是一篇极其热情的文章，跟上我启程出发！

## TCP的问题

熟悉TCP的你们知道它已经有了自己关于连接、可靠性、排序和避免拥堵的概念，那么为什么我们还要重写我们自己的迷你版本的基于UDP的TCP呢？

问题是多人动作游戏依靠于一个稳定的每秒发送10到30包的数据包流，而且在大多数情况下，这些数据包中包含的数据对时间是如此敏感以至于只有最新的数据才是有用的。这包括玩家的输入，位置方向和每个玩家角色的速度以及游戏世界中物理对象的状态等数据。

TCP的问题是它提取的是以可靠有序的数据流发送的数据。正因为如此，如果一个数据包丢失了，TCP不得不停止以等待那个数据包重新发送，这打断了这个稳定的数据包流因为更多的最新的数据包在重新发送的数据包到达之前必须在队列中等待，所以数据包必须有序地提供。

我们需要的是一种不同类型的可靠性。我们想要以一个稳定的速度发送数据包而且当数据被其他电脑接收到时我们会得到通知，而不是让所有的数据用一个可靠有效的数据流处理。这样的方法使得那些对时间敏感的数据能够不用等待重新发送的数据包就通过，而让我们自己拿主意怎么在应用层级去处理丢包。

具有TCP这些特性的系统是不可能实现可靠性的，因此我们别无选择只能在UDP的基础上自行努力。

不幸的是，可靠性并不是唯一一个我们必须重写的东西，这是因为TCP也提供避免拥堵功能，这样它就能够动态地衡量数据发送速率以来适应网络连接的性能。例如TCP在28.8k的调制调解器上会比在T1线路上发送更少的数据，而且它在不用事先知道这是什么类型的网络连接的情况下就能这么做！

## 序列号

现在回到可靠性！

我们可靠性系统的目标很简单：我们想要知道哪些数据包到了网络连接的另一端。

首先我们得鉴别数据包。

如果我们添加一个“数据包id”的概念会怎么样？让我们先给id赋一个整数值。我们能够从零开始，然后随着我们每发送一个数据包，增加一个数值。我们发送的第一个数据包就是“包0”，发送的第100个数据包就是“包99”。

这实际上是一个相当普遍的技术。甚至于在TCP中也得到了应用！这些数据包id叫做序列号，然而我们并不打算像TCP那样去做来实现可靠性，使用相同的术语是有意义的，因此从现在起我们还将称之为序列号。

因为UDP并不能保证数据包的顺序，所以第100个收到的数据包并不一定是第100个发出的数据包。接下来我们需要在数据包中插入序列号这样网络连接另一端电脑便能够知道是哪个数据包。

我们在[前一篇文章](http://gafferongames.com/networking-for-game-programmers/virtual-connection-over-udp/)中已经有了一个简单的关于虚拟网络连接的数据头，因此我们将只需要像这样在数据头中插入序列号：

```
[uint protocol id]

[uint sequence]

(packet data…)
```

现在当其他电脑收到一个数据包时通过发送数据包的电脑它就能知道数据包的序列号啦。

## 应答系统

既然我们已经能够使用序列号来鉴别数据包，下一步就该是让网络连接的另一端知道我们收到了哪个包了。

逻辑上来说这是非常简单的，我们只需要记录我们收到的每个包的序列号，然后把那些序列号发回发送他们的电脑即可。

因为我们是在两个机器间相互发送数据包，我们只能在数据包头添加上确认字符，就像我们加上序列号一样：

```
[uint protocol id]

[uint sequence]

[uint ack]

(packet data…)
```

我们的一般方法如下：

*   每次我们发送一个数据包我们就增加本地序列号。
*   当我们接收一个数据包时，我们将这个数据包的序列号与最近收到的数据包的序列号(称之为远程序列号)进行核对。如果这个包时间更近，我们就更新远程序列号使之等于这个数据包的序列号。
*   当我们编写数据包头时，本地序列号就变成了数据包的序列号，而远程序列号则变成确认字符。

这个简单的应答系统工作条件是每当我们发出一个数据包就会接收到一个数据包。

但如果数据包一起发送这样在我们发送一个数据包之前有两个数据包到达该怎么办呢？我们每个数据包只留了一个确认字符的位置，那我们该怎么处理呢?

现在考虑网络连接中的一端用更快的速率发送数据包这种情况。如果客户端每秒发送30个数据包，而服务器每秒只发送10个数据包，这样从服务器发出的每个数据包我们至少需要3个确认字符。

让我们想得更复杂点！如果数据包留下来了而确认字符丢失了会怎么样？这样发送这个数据包的电脑会认为这个数据包已经丢失了而实际上它已经被收到了！

貌似我们需要让我们的可靠性系统……更加可靠一点！

## 可靠的应答系统

这就是我们偏离TCP的地方。

TCP的做法是在确认字符发送的地方给下一个按顺序预期该收到的数据包序列号的位置维持一个移动窗口。如果TCP对于一个已经发出的数据包没有收到确认字符，它将暂停并重新发送那个对应序列号的数据包。这正是我们想要避免的做法！

因此在我们的可靠性系统里，我们从不为一个已经发出的序列号重新发送数据包，我们精确地只排序一次n，然后我们发送n+1，n+2，依次类推。如果数据包n丢失了我们也从不暂停重新发送它，而是把它留给应用程序来编写一个包含丢失数据的新的数据包，必要的话，这个包还会用一个新的序列号发送。

因为我们工作的方式与TCP不同，它的做法现在可能在我们数据包的确定字符设置中有了个洞，因此现在仅仅陈述最近的数据包的序列号已经远远不够了。

我们需要在每个数据包中包含多个确认字符。

那我们需要多少确认字符呢?

正如之前提到网络连接的一端发包速率比另一端快的情况，让我们假定最糟的情况是一端每秒钟发送不少于10个数据包，而另一端每秒钟发送不多于30个数据包。这种情况下，我们每个数据包需要的平均确认字符数是3个，但是如果数据包发送密集点，我们将需要更多。让我们说6-10个最差的情况。

如果因为包含确认字符的数据包丢失而导致确认字符并没有到达怎么办?

为了解决这个问题，我们将要使用一种经典的使用冗余码的网络设计策略来处理数据包丢失的情况！

让我们在每个数据包中容纳33个确认字符，而且这不仅是他将要达到33个，而是一直是33个。因此对于每一个发出的确认字符我们多余地把它额外多发送了多达32次，仅仅是以防某个包含确认字符的数据包不能通过！

但是我们怎么可能在一个数据包里配置33个确认字符呢？每个确认字符4字节那就是132字节了！

窍门是在“相应确认字符”之前使用一段位域来代表32个之前的确认字符，就像这样：

```
[uint protocol id]

[uint sequence]

[uint ack]

[uint ack bitfield]

(packet data…)
```

我们这样规定“位域”中每一位对应“相应确认字符”之前的32个确认字符。因此让我们说“相应确认字符”是100。如果位域的第一位设置好了，那么这个数据包也包含包99的一个确认字符。如果第二位设置好了，那么它也包含包98的一个确认字符。这样一路下来就到了包68的第32位。

我们调整过的算法看起来就像这样:

*   每次我们发送一个数据包我们就增加本地序列号。
*   当我们接收一个数据包时，我们将这个数据包的序列号与最近收到的数据包的序列号(称之为远程序列号)进行核对。如果这个包是更新的，我们就更新远程序列号使之等于数据包的序列号。
*   当我们编写数据包头时，本地序列号就变成了数据包的序列号，而远程序列号则变成确认字符。 计算确认字符位域是通过寻找一个多达33个数据包的队列，其中包括在\[远程序列号-32，远程序列号\]范围内的序列号。如果序列号“远程序列号-n”正在接收队列中那就把确认字符位域中的位n（在\[1，32\]范围内）设置为位1。
*   此外，当一个数据包被接收了，确认字符位域也被扫描了，如果位n设置好了，那么即使它还没有被应答，我们也认可序列号“远程序列号-n”。

利用这个改善过的算法，你将可能不得不在不止一秒内丢掉100%的数据包而不是让一个数据包停止通过。当然，它能够轻松地处理不同的发包速率和接受一起发送的数据包。

## 检测丢包

既然我们知道网络连接另一端接受的是哪些数据包，那么我们该怎么检测数据包的丢失呢?

这次的窍门是反过来想，如果你在一定时间内还没有收到某个数据包的应答，那么我们可以考虑说那个数据包已经丢失了。

考虑到我们正在以每秒不超过30包的速率发送数据包，而且我们正在多余地发送数据包大概三十次。如果你在一秒内没有收到某个数据包的确认字符，那很有可能就是这个数据包已经丢失了。

因此我们在这儿用了一些小窍门，尽管我们能100%确定哪个数据包通过了，但是我们只能适度地确定那些没有到达的数据包。

这种情况的复杂性在于任何你重新发送的使用了这种可靠性方法的数据需要有它自己的信息id，这样的话在你多次收到它的时候你可以放弃它。这在应用层级是能够做到的。

## 应对环绕式处理的序列号

如果序列号没有环绕式处理覆盖，那么对于序列号和确认字符的讨论是不完整的！

序列号和确认字符都是32比特的无符号整数，因此它们能够代表在范围\[0，4294967295\]内的数字。那是一个非常大的数字！那么大以至于如果你每秒发送三十个数据包也将要花费四年半来把这个序列号环绕式处理回零。

但是可能你想要节省一些带宽这样你将你的序列号和确认字符缩减到到16比特整数。你每个数据包节省了4个字节，但现在他们只需要在仅仅半个小时内即可完成环绕式处理！

所以我们该怎么应对这种环绕式处理的情况呢?

诀窍是要认识到如果当前序列号已经非常高了，而且下一个到达的序列号很低，那么你就必须进行环绕式处理。那么即使新的序列号数值上比当前序列号值更低它也能实际代表一个更新的数据包。

举个例子，让我们假设我们用一个字节编码序列号（顺便说一下并不推荐这样做）。 :))， 之后他们就会在255后面进行环绕式处理，就像这样:

```
… 252, 253, 254, 255, 0, 1, 2, 3,…
```

为了解决这种情况我们需要一个能够意识到在255之后需要环绕式处理回零这样一个事实的新功能，这样0，1，2，3就会被认为比255更新。否则，我们的可靠性系统就会在你收到包255后停止工作。

这就是那个新功能：

```cpp
bool sequence_more_recent( unsigned int s1,
                           unsigned int s2,
                           unsigned int max )
{
    return ( s1 > s2 ) &&
           ( s1 - s2 <= max/2 ) ||
           ( s2 > s1 ) &&
           ( s2 - s1 > max/2 );
}
```

这个功能通过比较两个数字和他们的不同来工作。如果它们之间的差异少于1/2的最大序列号值，那么它们必须靠在一起– 因此我们只需要照常检查某个序列号是否比另一个大。然而，如果它们相差很多，它们之间的差异将会比1/2的最大序列号值大，那么如果它比当前序列号小我们反而认为这个序列号是更新的。

这最后一点是显然需要环绕式处理序列号的地方，那么0，1，2就会被认为比255更新。

多么简洁而巧妙！

一定要确保你在你所做的任何序列号处理当中包含了这一步！

## 避免拥堵

当你已经解决了可靠性的问题的时候，还有避免拥堵的问题。当你获得TCP的可靠性的时候TCP已经提供了避免拥堵的功能作为数据包的一部分，但是UDP无论怎样都不会有避免拥堵！

如果我们仅仅发送数据包而没有某种流量控制，我们正在冒险占满网络连接而且会引起严重的延迟（2秒以上！），正如我们和另外一台电脑之间的路由器会超负荷而缓冲数据包。这个发生是因为路由器很努力地想要尝试传送我们发送的所有数据包，因此在它们考虑丢弃数据包之前会在队列中缓冲数据包。

然而如果我们能告诉路由器我们的数据包是时间敏感的而且如果路由器超载的话这些数据包应该丢弃而不是缓冲这样会很棒的，但只有我们重写世界上所有路由器的软件才能做到这一点！

那么我们反而需要把重点放在我们实际上能做的是避免占满首位网络连接。

做到这个的方法是实施我们自己的基础避免拥堵算法。我强调基础！就像可靠性，我们并不寄希望于像TCP第一次尝试应用那样普通而粗暴地想出某些东西，那么让我们让它尽可能简单吧。

## 衡量往返时间

因为所有避免拥堵的要点就是避免占满网络连接和避免增加往返时间（RTT），关于我们是不是占满网络的最重要的衡量标准是RTT它本身的观点是有道理的。

我们需要一种方法来衡量我们网络连接的RTT。

这是基础的技巧：

*   对我们发送的每个数据包，我们对数据包队列中包含的序列号和他们发送的时间添加一个登记。
*   当我们收到一个应答时，我们找到这个登记, 然后记录我们收到这个应答的时间t1与我们发送数据包的时间的t2的差值(都基于本地时间来计算)。这就是是这个数据包的RTT时间。
*   因为数据包的到达因网络波动而不同，我们需要缓和这个值来提供某些有意义的东西，这样每次我们获得一个新的RTT我们就移动一个我们当前的RTT和数据包的RTT之间距离的百分比。10%在实践中看起来效果很好。这就叫做一个指数级平滑移动平均值，而且它在用一个低通滤波器的情况下能有效地平滑RTT中的杂音。
*   为了确保发送队列永不增长，一旦超过某些最大预期RTT值我们就丢弃数据包。正如上一节关于可靠性讨论过的，任何在一秒内未应答的数据包都极有可能丢失了，那么对于最大RTT来说，一秒是个很棒的值。

既然我们有RTT，我们能把它作为一个衡量标准来推动我们的避免拥堵功能。如果RTT变得太大了，我们更缓慢地发送数据，如果它的值低于可接受范围，我们能努力更频繁地发送数据。

## 简单的好坏机制避免拥堵

正如之前讨论的，我们不要那么贪心，我们将要执行一个非常基础的避免拥堵机制。这个避免拥堵机制有两种模式。好和坏。我把它叫做简单的二进制避免拥堵。

让我们假设你在发送一个确定大小的数据包，就假设256字节吧。你想要每秒发送这些数据包30次，但是如果网络条件差，你可以削减为每秒10次。

那么30次256字节的数据包的速率大概是64kbits/sec，每秒10次的话大概20kbits/sec。世界上没有一个宽带连接不能处理至少20kbits/sec的速率，所以我们在这样的假定下继续前进。不像TCP这样对有任何数量的发送/接受带宽的任何设备都完全通用，我们将假设一个设备的最小支持带宽来参与我们的网络连接。

所以基础想法就是这样了。当网络条件好的时候我们每秒发送30个数据包，当网络条件差的时候我们降至每秒10个数据包。

当然，你能随你喜爱定义好和坏，但是仅考虑RTT的时候我已经得到了好的成效。举个例子，如果RTT超过某些极限值（假设250ms）那你就知道你可能已经正占满了网络连接。当然，这里假设一般没人在非占满网络条件下超过250ms，考虑到我们的宽带要求这是合理的。。

好和坏之间你会怎么转换？我喜欢用下列操作的算法:

*   如果你当前在好模式下，而网络条件突然变坏，立即降至坏模式。
*   如果你正在坏模式下，而且网络条件已经好了一段特定时长”t”，那么回到好模式。
*   为了避免好模式和坏模式之间的快速切换，如果你从好模式降至坏模式持续10秒钟以内，从坏模式回到好模式之前的时间是”t”的两倍。在某些最大值中固定这个时间值，假设60秒。
*   为了避免打击良好的网络连接，当它们有一小段时期的差连接时，每过10秒连接就处于好模式，把坏模式回到好模式之前的时间“t”减半。在某些最小值中固定这个时间值，例如1秒。

利用这个算法，你将对差网络连接迅速反应然后降低你的发送速率至每秒10个数据包，避免占满网络。在网络条件好时，你也将谨慎地尝试好模式，坚持以更高的每秒发送30个数据包的速率发送数据包。

当然，你也能实施复杂得多的算法，丢包率百分比甚至是网络波动（数据包确认字符的时间差异）都可以考虑作为一个衡量标准，而不仅仅是RTT。

对于避免拥堵你还可以更贪心点，并尝试发现什么时候你能以一个更高的带宽（例如LAN）发送数据，但是你必须非常小心！随着贪婪心的增加你占满网络连接的风险也在增大！

## 结语

我们全新的可靠性系统让我们稳定流畅发送数据包，而且能通知我们收到了什么数据包。从这我们能推断出丢失的数据包，必要的话重新发送没有通过的数据。

基于此我们有了能够取决于网络条件在每秒10次和每秒30次发包速率间轮流切换的一个简单的避免拥堵系统，因此我们不会占满网络连接。

还有很多实施细节因为太具体而不能在这篇文章一一提到，所以务必确保你检查[示例源代码](http://netgame.googlecode.com/files/ReliabilityAndFlowControl.zip)来看是否它都被实施了。

这就是关于可靠性，排序和避免拥堵的一切了，或许是低层次网络设计中最复杂的一面了。

【版权声明】

原文作者未做权利声明，视为共享知识产权进入公共领域，自动获得授权；
