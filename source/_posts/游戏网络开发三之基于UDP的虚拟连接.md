---
title: 游戏网络开发三之基于UDP的虚拟连接
date: 2019-05-19 11:36:58
tags: [GafferOnGames, UDP]
categories: Multiplayer
---
# 原文

[原文出处](https://gafferongames.com/post/virtual_connection_over_udp/)

## Introduction

Hi, I’m [Glenn Fiedler](https://gafferongames.com/about) and welcome to [**Networking for Game Programmers**](https://gafferongames.com/categories/game-networking/).

In the [previous article](https://gafferongames.com/post/sending_and_receiving_packets) we sent and received packets over UDP. Since UDP is connectionless, one UDP socket can be used to exchange packets with any number of different computers. In multiplayer games however, we usually only want to exchange packets between a small set of connected computers.

As the first step towards a general connection system, we’ll start with the simplest case possible: creating a virtual connection between two computers on top of UDP.

But first, we’re going to dig in a bit deeper about how the Internet really works!

## The Internet NOT a series of tubes

In 2006, Senator Ted Stevens made internet history with his [famous speech](https://en.wikipedia.org/wiki/Series_of_tubes) on the net neutrality act:

  
“The internet is not something that you just dump something on. It’s not a big truck. It’s a series of tubes”

When I first started using the Internet, I was just like Ted. Sitting in the computer lab in University of Sydney in 1995, I was “surfing the web” with this new thing called Netscape Navigator, and I had absolutely no idea what was going on.

You see, I thought each time you connected to a website there was some actual connection going on, like a telephone line. I wondered, how much does it cost each time I connect to a new website? 30 cents? A dollar? Was somebody from the university going to tap me on the shoulder and ask me to pay the long distance charges? :)

Of course, this all seems silly now.

There is no switchboard somewhere that directly connects you via a physical phone line to the other computer you want to talk to, let alone a series of pneumatic tubes like Sen. Stevens would have you believe.

## No Direct Connections

Instead your data is sent over Internet Protocol (IP) via packets that hop from computer to computer.

A packet may pass through several computers before it reaches its destination. You cannot know the exact set of computers in advance, as it changes dynamically depending on how the network decides to route packets. You could even send two packets A and B to the same address, and they may take different routes.

On unix-like systems can inspect the route that packets take by calling “traceroute” and passing in a destination hostname or IP address.

On windows, replace “traceroute” with “tracert” to get it to work.

Try it with a few websites like this:

```
traceroute slashdot.org  
traceroute amazon.com  
traceroute google.com  
traceroute bbc.co.uk  
traceroute news.com.au  
```

Take a look and you should be able to convince yourself pretty quickly that there is no direct connection.

<!-- more -->

## How Packets Get Delivered

In the [first article](https://gafferongames.com/post/udp_vs_tcp/), I presented a simple analogy for packet delivery, describing it as somewhat like a note being passed from person to person across a crowded room.

While this analogy gets the basic idea across, it is much too simple. The Internet is not a flat network of computers, it is a network of networks. And of course, we don’t just need to pass letters around a small room, we need to be able to send them anywhere in the world.

It should be pretty clear then that the best analogy is the postal service!

When you want to send a letter to somebody you put your letter in the mailbox and you trust that it will be delivered correctly. It’s not really relevant to you _how_ it gets there, as long as it does. Somebody has to physically deliver your letter to its destination of course, so how is this done?

Well first off, the postman sure as hell doesn’t take your letter and deliver it personally! It seems that the postal service is not a series of tubes either. Instead, the postman takes your letter to the local post office for processing.

If the letter is addressed locally then the post office just sends it back out, and another postman delivers it directly. But, if the address is is non-local then it gets interesting! The local post office is not able to deliver the letter directly, so it passes it “up” to the next level of hierarchy, perhaps to a regional post office which services cities nearby, or maybe to a mail center at an airport, if the address is far away. Ideally, the actual transport of the letter would be done using a big truck.

Lets be complicated and assume the letter is sent from Los Angeles to Sydney, Australia. The local post office receives the letter and given that it is addressed internationally, sends it directly to a mail center at LAX. The letter is processed again according to address, and gets routed on the next flight to Sydney.

The plane lands at Sydney airport where an _entirely different postal system_ takes over. Now the whole process starts operating in reverse. The letter travels “down” the hierarchy, from the general, to the specific. From the mail hub at Sydney Airport it gets sent out to a regional center, the regional center delivers it to the local post office, and eventually the letter is hand delivered by a mailman with a funny accent. Crikey! :)

Just like post offices determine how to deliver letters via their address, networks deliver packets according to their IP address. The low-level details of this delivery and the actual routing of packets from network to network is actually quite complex, but the basic idea is that each router is just another computer, with a routing table describing where packets matching sets of addresses should go, as well as a default gateway address describing where to pass packets for which there is no matching entry in the table. It is routing tables, and the physical connections they represent that define the network of networks that is the Internet.

The job of configuring these routing tables is up to network administrators, not programmers like us. But if you want to read more about it, then this article from [ars technica](https://arstechnica.com/guides/other/peering-and-transit.ars) provides some fascinating insight into how networks exchange packets between each other via peering and transit relationships. You can also read more details about [routing tables](http://www.faqs.org/docs/linux_network/x-087-2-issues.routing.html) in this linux faq, and about the [border gateway protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) on wikipedia, which automatically discovers how to route packets between networks, making the internet a truly distributed system capable of dynamically routing around broken connectivity.

## Virtual Connections

Now back to connections.

If you have used TCP sockets then you know that they sure _look_ like a connection, but since TCP is implemented on top of IP, and IP is just packets hopping from computer to computer, it follows that TCP’s concept of connection must be a _virtual connection._

If TCP can create a virtual connection over IP, it follows that we can do the same over UDP.

Lets define our virtual connection as two computers exchanging UDP packets at some fixed rate like 10 packets per-second. As long as the packets are flowing, we consider the two computers to be virtually connected.

Our connection has two sides:

  
  

*   One computer sits there and _listens_ for another computer to connect to it. We’ll call this computer the server.

*   Another computer _connects_ to a server by specifying an IP address and port. We’ll call this computer the client.

  
  

In our case, we only allow one client to connect to the server at any time. We’ll generalize our connection system to support multiple simultaneous connections in a later article. Also, we assume that the IP address of the server is on a fixed IP address that the client may directly connect to.

## Protocol ID

Since UDP is connectionless our UDP socket can receive packets sent from any computer.

We’d like to narrow this down so that the server only receives packets sent from the client, and the client only receives packets sent from the server. We can’t just filter out packets by address, because the server doesn’t know the address of the client in advance. So instead, we prefix each UDP packet with small header containing a 32 bit protocol id as follows:

```
[uint protocol id]  
(packet data…)
```

The protocol id is just some unique number representing our game protocol. Any packet that arrives from our UDP socket first has its first four bytes inspected. If they don’t match our protocol id, then the packet is ignored. If the protocol id does match, we strip out the first four bytes of the packet and deliver the rest as payload.

You just choose some number that is reasonably unique, perhaps a hash of the name of your game and the protocol version number. But really you can use anything. The whole point is that from the point of view of our connection based protocol, packets with different protocol ids are ignored.

## Detecting Connection

Now we need a way to detect connection.

Sure we could do some complex handshaking involving multiple UDP packets sent back and forth. Perhaps a client “request connection” packet is sent to the server, to which the server responds with a “connection accepted” sent back to the client, or maybe an “i’m busy” packet if a client tries to connect to server which already has a connected client.

Or… we could just setup our server to take the first packet it receives with the correct protocol id, and consider a connection to be established.

The client just starts sending packets to the server assuming connection, when the server receives the first packet from the client, it takes note of the IP address and port of the client, and starts sending packets back.

The client already knows the address and port of the server, since it was specified on connect. So when the client receives packets, it filters out any that don’t come from the server address. Similarly, once the server receives the first packet from the client, it gets the address and port of the client from “recvfrom”, so it is able to ignore any packets that don’t come from the client address.

We can get away with this shortcut because we only have two computers involved in the connection. In later articles, we’ll extend our connection system to support more than two computers in a client/server or peer-to-peer topology, and at this point we’ll upgrade our connection negotiation to something more robust.

But for now, why make things more complicated than they need to be?

## Detecting Disconnection

How do we detect disconnection?

Well if a connection is defined as receiving packets, we can define disconnection as _not_ receiving packets.

To detect when we are not receiving packets, we keep track of the number of seconds since we last received a packet from the other side of the connection. We do this on both sides.

Each time we receive a packet from the other side, we reset our accumulator to 0.0, each update we increase the accumulator by the amount of time that has passed.

If this accumulator exceeds some value like 10 seconds, the connection “times out” and we disconnect.

This also gracefully handles the case of a second client trying to connect to a server that has already made a connection with another client. Since the server is already connected it ignores packets coming from any address other than the connected client, so the second client receives no packets in response to the packets it sends, so the second client times out and disconnects.

## Conclusion

And that’s all it takes to setup a virtual connection: some way to establish connection, filtering for packets not involved in the connection, and timeouts to detect disconnection.

Our connection is as real as any TCP connection, and the steady stream of UDP packets it provides is a suitable starting point for a multiplayer action game.

Now that you have your virtual connection over UDP, you can easily setup a client/server relationship for a two player multiplayer game without TCP.

# 译文

[译文出处](http://gad.qq.com/program/translateview/7161829)

译者：张华栋(wcby) 审校：崔国军（飞扬971）

## 序言

大家好，我是Glenn Fiedler，欢迎阅读《[针对游戏程序员的网络知识](http://gafferongames.com/networking-for-game-programmers/)》系列教程的第三篇文章。

在之前的文章中，我向你展示了如何使用UDP协议来发送和接收数据包。

由于UDP协议是无连接的传输层协议，一个UDP套接字可以用来与任意数目的不同电脑进行数据包交换。但是在多人在线网络游戏中，我们通常只需要在一小部分互相连接的计算机之间交换数据包。

  
作为实现通用连接系统的第一步，我们将从最简单的可能情况开始：创建两台电脑之间构建于UDP协议之上的虚拟连接。

但是首先，我们将对互联网到底是如何工作的进行一点深度挖掘！

## 互联网不是一连串的管子

在2006年，参议院特德·史蒂文斯(Ted Stevens) 用他关于互联网中立（netneutrality）法案的著名演讲创造了互联网的历史：

”互联网不是那种你随便丢点什么东西进去就能运行的东西。它不是一个大卡车。它是一连串的管子“

当我第一次开始使用互联网的时候，我也像Ted一样无知。那是1995年，我坐在悉尼大学的计算机实验室里，在用一种叫做Netscape的网络浏览器（最早最热门的网页浏览工具）“在网上冲浪（surfing the web）“，那个时候我对发生了什么根本一无所知。

你看那个时候，我觉得每次连到一个网站上就一定有某个真实存在的连接在帮我们传递信息，就像电话线一样。那时候我在想，当我每次连到一个新的网站上需要花费多少钱? 30美分吗?一美元吗? 会有大学里的某个人过来拍拍我的肩膀让我付长途通信的费用么？

当然，现在回头看那时候一切的想法都非常的愚蠢。

并没有在某个地方存在一个物理交换机用物理电话线将你和你希望通话的某个电脑直接连起来。更不用说像参议院史蒂文斯想让你相信的那样存在一串气压输送管。

## 没有直接的连接

相反你的数据是基于IP协议(InternetProtocol)通过在电脑到电脑之间发送数据包来传递信息的**。**

一个数据包可能在到达它的目的地之前要经过几个电脑。你没有办法提前知道数据包会经过具体哪些电脑，因为它会依赖当前网络的情况对数据包进行路由来动态的改变路径。甚至有可能给同一个地址发送A和B两个数据包，这两个数据包都采用不同的路由。这就是为什么UDP协议不能保证数据包的到达顺序。（其实这么说稍微容易有点引起误解，TCP协议是能保证数据包的到达顺序的，但是他也是基于IP协议进行数据包的发送，并且往同一个地址发送的两个数据包也有可能采用完全不同的路由，这主要是因为TCP在自己这一层做了一些控制而UDP没有，所以导致TCP协议可以保证数据包的有序性，而UDP协议不能，当然这种保证需要付出性能方面的代价）。  
在类unix的系统中可以通过调用“traceroute”函数并传递一个目的地主机名或IP地址来检查数据包的路由。

在Windows系统中，可以用“tracert”代替“traceroute”，其他不变，就能检查数据包的路由了。

像下面这样用一些网址来尝试下这种方法：

```
traceroute slashdot.org

traceroute amazon.com

traceroute google.com

traceroute bbc.co.uk

traceroute news.com.au
```

运行下看下输出结果，你应该很快就能说服你自己确实连接到了网站上，但是并没有一个直接的连接。

## 数据包是如何传递到目的地的？

  
  
在[第一篇文章](http://gafferongames.com/networking-for-game-programmers/udp-vs-tcp/)中，我对数据包传递到目的地这个事情做了一个简单的类比，把这个过程描述的有点像在一个拥挤的房间内一个人接着一个人的把便条传递下去。

虽然这个类比的基本思想还是表达出来了，但是它有点过于简单了。互联网并不是电脑组成的一个平面的网络，实际上它是网络的网络。当然，我们不只是要在一个小房间里面传递信件，我们要做的事能够把信息传递到全世界。

  
这就应该很清楚了，数据包传递到目的地的最好的类比是邮政服务!

当你想给某人写信的时候，你会把你的信件放到邮箱里并且你相信它将正确的传递到目的地。这封信件具体是怎么到达目的地的和你并不是十分相关，尽管它是否正确到达会对你有影响。当然会有某个人在物理上帮你把信件传递到目的地，所以这是怎么做的呢?

首先，邮递员肯定不需要自己去把你的信件送到目的地！看起来邮政服务也不是一串管子。相反，邮递员是把你的信件带到当地的邮政部门进行处理。

如果这封信件是发送给本地的，那么邮政部门就会把这封信件发送回来，另外一个邮递员会直接投递这封信件。但是，如果这封信件不是发送给本地的，那么这个处理过程就有意思了！当地的邮政部门不能直接投递这封信件，所以这封信件会被向上传递到层次结构的上一层，这个上一层也许是地区级的邮政部门它会负责服务附近的几个城市，如果要投递的地址非常远的话，这个上一层也许是位于机场的一个邮件中心。理想情况下，信件的实际运输将通过一个大卡车来完成。

让我们通过一个例子来把上面说的过程具体的走一遍，假设有一封信件要从洛杉矶发送到澳大利亚的悉尼。当地的邮政部门收到信件以后考虑到这封信件是一封跨国投递的信件，所以会直接把它发送到位于洛杉矶机场的邮件中心。在那里，这封信件会再次根据它的地址进行处理，并被安排通过下一个到悉尼的航班投递到悉尼去。

当飞机降落到悉尼机场以后，一个完全不同的邮政系统会负责接管这封信件。现在整个过程开始逆向操作。这封信件会沿着层次结构向下传递，从大的管理部门到具体的投递区域。这封信件会从悉尼机场的邮件中心被送往一个地区级的中心，然后地区级的中心会把这封信件投递到当地的邮政部门，最终这封信件会是由一个操着有趣的本地口音的邮政人员用手投递到真正的目的地的。哎呀! !

就像邮局是通过信件的地址来决定这些信件是该如何投递的一样，网络也是根据这些数据包的IP地址来决定它们是该如何传递的。投递机制的底层细节以及数据包从网络到网络的实际路由其实都是相当复杂的，但是基本的想法都是一样的，就是每个路由器都只是另外一台计算机，它会携带一张路由表用来描述如果数据包的IP地址匹配了这张表上的某个地址集，那么这个数据包该如何传递，这张表还会记载着默认的网关地址，如果数据包的IP地址和这张路由表上的一个地址都匹配不上，那么这个数据包该传递到默认的网关地址那里。其实是路由表以及它们代表的物理连接定义了网络的网络，也就是互联网（互联网也被称为万维网）。

[因特网](http://baike.baidu.com/view/1706.htm)于1969年诞生于[美国](http://baike.baidu.com/view/2398.htm)。最初名为“[阿帕网](http://baike.baidu.com/view/108095.htm)”（ARPAnet）是一个军用研究系统，后来又成为连接大学及高等院校计算机的学术系统，则已[发展](http://baike.baidu.com/view/141536.htm)成为一个覆盖五大洲150多个国家的开放型全球[计算机网络系统](http://baike.baidu.com/view/541460.htm)，拥有许多服务商。普通电脑用户只需要一台个人计算机用电话线通过[调制解调器](http://baike.baidu.com/view/1074.htm)和[因特网](http://baike.baidu.com/view/1706.htm)服务商连接，便可进入因特网。但[因特网](http://baike.baidu.com/view/1706.htm)并不是全球唯一的[互联网络](http://baike.baidu.com/view/380232.htm)。例如在[欧洲](http://baike.baidu.com/view/3622.htm)，跨国的[互联网络](http://baike.baidu.com/view/380232.htm)就有“欧盟网”（Euronet），“欧洲学术与研究网”（EARN），“欧洲信息网”（EIN），在美国还有“国际学术网”（[BITNET](http://baike.baidu.com/view/370280.htm)），世界范围的还有“飞多网”（全球性的[BBS](http://baike.baidu.com/view/66.htm)系统）等。但这些网络其实根本就不需要知道，感谢IP协议的帮助，只要知道他们是可以互联互通的就可以。

这些路由表的配置工作是由网络管理员完成的，而不是由像我们这样的程序员来做。但是如果你想要了解这方面的更多内容， 那么来自[ars technica](http://arstechnica.com/guides/other/peering-and-transit.ars)的这篇文章将提供网络是如何在端与端之间互联来交换数据包以及传输关系方面一些非常有趣的见解。你还可以通过linux常见问题中路由表（[routing tables](http://www.faqs.org/docs/linux_network/x-087-2-issues.routing.html)）方面的文章以及维基百科上面的边界网关协议（[border gateway protocol](http://en.wikipedia.org/wiki/Border_Gateway_Protocol) ）的解释来获得更多的细节。边界网关协议是用来自动发现如何在网络之间路由数据包的协议，有了它才真正的让互联网成为一个分布式系统，能够在不稳定的连接里面进行动态的路由。

边界网关协议（BGP）是运行于 TCP 上的一种[自治系统](http://baike.baidu.com/view/2663.htm)的[路由协议](http://baike.baidu.com/view/7031.htm)。 BGP 是唯一一个用来处理像因特网大小的网络的协议，也是唯一能够妥善处理好不相关[路由域](http://baike.baidu.com/view/4303246.htm)间的多路连接的协议。 BGP 构建在 EGP 的经验之上。 BGP 系统的主要功能是和其他的 BGP 系统交换网络可达信息。网络可达信息包括列出的[自治系统](http://baike.baidu.com/view/2663.htm)（AS）的信息。这些信息有效地构造了 AS 互联的拓朴图并由此清除了[路由环路](http://baike.baidu.com/view/2098835.htm)，同时在 AS 级别上可实施策略决策。

## 虚拟的连接

现在让我们回到连接本身。

如果你已经使用过TCP套接字，那么你会知道它们看起来真的像是一个连接，但是由于TCP协议是在IP协议之上实现的，而IP协议是通过在计算机之间进行跳转来传递数据包的，所以TCP的连接仍然是一个虚拟连接。

如果TCP协议可以基于IP协议建立虚拟连接，那么我们在UDP协议上所做的一切都可以应用于TCP协议上。

让我们给虚拟连接下个定义：两个计算机之间以某个固定频率比如说每秒10个数据包来交换UDP的数据包。只要数据包仍然在传输，我们就认为这两台计算机之间存在一个虚拟连接。

我们的连接有两侧：

*   一个计算机坐在那儿侦听是否有另一台计算机连接到它。我们称负责监听的这台计算机为服务器（server）。
*   另一台计算机会通过一个指定的IP地址和端口连接到一个服务器。我们称主动连接的这台电脑为客户端（client）。

在我们的场景里，我们只允许一个客户端在任意的时候连接到服务器。我们将在下一篇文章里面拓展我们的连接系统以支持多个客户端的同时连接。此外，我们假定服务器的IP地址是一个固定的IP地址，客户端可以随时直接连接上来。我们将在后面的文章里面介绍匹配（matchmaking）和NAT打穿（NATpunch-through）。

## 协议ID

由于UDP协议是无连接的传输层协议，所以我们的UDP套接字可以接受来自任何电脑的数据包。

我们想要缩小接收数据包的范围，以便我们的服务器只接收那些从我们的客户端发送出来的数据包，并且我们的客户端只接收那些从我们的服务端发送出来的数据包。我们不能只通过地址来过滤我们的数据包，因为服务器没有办法提前知道客户端的地址。所以，我们会在每一个UDP数据包前面加上一个包含32位协议id的头,如下所示:

```
[uint protocol id]

(packet data…)
```

协议ID只是一些独特的代表我们的游戏协议的数字。我们的UDP套接字收到的任意数据包首先都要检查数据包的首四位。如果它们和我们的协议ID不匹配的话，这个数据包就会被忽略。如果它们和我们的协议ID匹配的话，我们会剔除数据包的第一个四个字节并把剩下的部分发给我们的系统进行处理。

你只要选择一些非常独特的数字就可以了，这些数字可以是你的游戏名字和协议版本号的散列值。不过说真的，你可以使用任何东西。这种做法的重点是把我们的连接视为基于协议进行通信的连接，如果协议ID不同，那么这样的数据包将被丢弃掉。

## 检测连接

现在我们需要一个方法来检测连接。

当然我们可以实现一些复杂的握手协议，牵扯到多个UDP数据包来回传递。比如说客户端发送一个”请求连接（request connection）“的数据包给服务器，当服务器收到这个数据包的时候会回应一个”连接接受（connection accepted）“的数据包给客户端，或者如果这个服务器已经有超过一个连接的客户端以后，会回复一个“我很忙（i’m busy）”的数据包给客户端。

或者。。我们可以设置我们的服务器，让它以它收到的第一个数据包的协议ID作为正确的协议ID，并在收到第一个数据包的时候就认为连接已经建立起来了。

客户端只是开始给服务器发送数据包，当服务器收到客户端发过来的第一个数据包的时候，它会记录下客户端的IP地址和端口号，然后开始给客户端回包。

客户端已经知道了服务器的地址和端口，因为这些信息是在连接的时候指定的。所以当客户端收到数据包的时候，它会过滤掉任何不是来自于服务器地址的数据包。同样的，一旦服务器收到客户端的第一个数据包，它就会从“recvfrom”函数里面得到客户端的地址和端口号，所以它也可以忽略任何不是发自客户端地址的数据包。

我们可以通过一个捷径来避开这个问题，因为我们的系统只有两台计算机会建立连接。在后面的文章里，我们将拓展我们的连接系统来支持超过两台计算机参与客户端/服务器或者端对端（peer-to-peer，p2p）网络模型，并且在那个时候我们会升级我们的连接协议方式来让它变得更加健壮。

但是现在，为什么我们要让事情变得超出需求的复杂度呢？（作者的意思是因为我们现在不需要解决这个问题，因为我们的场景是面对只有两台计算机的情况，所以我们可以先放过这个问题。）

## 检测断线的情况

我们该如何检测断线（disconnection）的情况？

那么，如果一个连接被定义为接收数据包，我们可以定义断线为收不到数据包。

为了检测什么时候开始我们收不到数据包，我们要记录上一次我们从连接的另外一侧收到数据包到现在过去了多少秒，我们在连接的两侧都做了这个事情。

每次我们从连接的另外一端收到数据包的时候，我们都会重置我们的计数器为0.0，每一次更新的时候我们都会把这次更新到上一次更新逝去的时间量加到计数器上。

如果计数器的值超过某一个值，比如说10秒，那么我们就认定这个连接“超时”了并且我们会断开连接。

这也可以很优雅的处理当服务器已经与一个客户端建立连接以后，有第二个客户端试图与服务器建立连接的情况。因为服务器已经建立了连接，它会忽略掉不是来自连接的客户端地址发出来的数据包，所以第二个客户端在发出了数据包以后得不到任何回应，这样它就会判断连接超时并断开连接。

## 总结

而这一切都需要设置一个虚拟连接：用某种方法建立一个连接，过滤掉那些不是来自这个连接的数据包，并且如果发现连接超时就断开连接。

我们的连接就跟任何TCP连接一样真实，并且UDP数据包构成的稳定数据流为多人在线动作网络游戏提供了一个很好的起点。

我们还获得了一些互联网是如何路由数据包的见解。举个例子来说，我们现在知道UDP数据包有时候会在到达的时候是乱序的原因是因为它们在IP层传输的时候采用不同的路由！看下互联网的地图，你会不会对你的数据包能够到达正确的目的点感到非常的神奇？如果你想对这个问题进行更加深入的了解，维基百科上的这篇文章([Internet backbone](https://en.wikipedia.org/wiki/Internet_backbone))是一个很好的起点。

现在，既然你已经有了一个基于UDP协议的虚拟连接，你可以轻松的在两个玩家的多人在线游戏里面设置一个客户端/服务器关系而不需要使用TCP协议。

你可以在这篇文章的示例源代码（[examplesource code](http://netgame.googlecode.com/files/VirtualConnectionOverUDP.zip) ）找到一个具体实现。

这是一个简单的客户端/服务器程序，每秒交换30个数据包。你可以在任意你喜欢的机器上运行这个服务器，只要给它提供一个公共的IP地址就可以了，需要公共IP地址的原因是我们目前还不支持NAT打穿（[NAT punch-through](http://www.jenkinssoftware.com/raknet/manual/natpunchthrough.html) ）。

NAT穿越（NATtraversal）涉及TCP/IP网络中的一个常见问题，即在处于使用了NAT设备的私有TCP/IP网络中的[主机](http://baike.baidu.com/view/23880.htm)之间建立连接的问题。

像这样来运行客户端：

```
./Client 205.10.40.50
```

它会尝试连接到你在命令行输入的地址。如果你不输入地址的话，默认情况下它会连接到127.0.0.1。

当一个客户端已经与服务器建立连接的时候，你可以尝试用另外一个客户端来连接这个服务器，你会注意到这次连接的尝试失败了。这么设计是故意的。因为到目前为止，一次只允许一个客户端连接上服务器。

你也可以在客户端和服务器连接的状态下尝试停止客户端或者服务器，你会注意到10秒以后连接的另外一侧会判断连接超时并断开连接。当客户端超时的时候它会退到shell窗口，但是服务器会退到监听状态为下一次的连接做好准备。

预告下接下来的一篇文章的题目:《基于UDP的可靠、有序和拥塞避免的传输》，欢迎继续阅读。

如果你喜欢这篇文章的话，请考虑对我做一个小小的捐赠。捐款会鼓励我写更多的文章!（原文作者在原文的地址上提供了一个捐赠网址，有兴趣的读者可以在文章开始的地方找到原文地址进行捐赠）

【版权声明】

原文作者未做权利声明，视为共享知识产权进入公共领域，自动获得授权。
