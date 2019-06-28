---
title: KBEngine源码解读五--转载二
date: 2019-06-28 15:05:55
tags: [ServerFramework, KBEngine]
categories: GameServer
---
[原文地址](https://blog.csdn.net/qq98756524aa/article/details/78782156)

此笔记是本人在开发及研读的过程中记录下来的，由于没整理，会看得有些吃力，请读者视能力而读，个人理解，如有问题，悉心接受。

部分引用KBEngine官网的一些句段，感谢kbe。

 

每个cell在被创建时当被观察者观察到时（也就是进入到别人的AOI里）那么会被调用kbe底层回调python层onWitnessed方法，

当然只在增加观察者时第一次被观察到或者删除观察者时观察者个数为0时才回调，通过参数1或者0进行区别，看以下代码

![KBE2_1.png](http://blog.sensedevil.com/image/KBE2_1.png)
![KBE2_2.png](http://blog.sensedevil.com/image/KBE2_2.png)

controlledBy:kbe底层提供了可以把一个非客户端持有的cell上的entity给客户端控制，由客户端来进行可以控制的改变权限是改变方向和位置，

但是这个entity必须是在控制者的entity（demo里的avatar的entity）的Aoi范围内，而且这个控制者是必须被客户端控制的一个entity，

具有aoi也就是还必须是具有cell的，操作如下：

在cell上self.controlledBy = base（操作者的base）

<!-- more -->

Kbe具体实现代码如下：

![KBE2_3.png](http://blog.sensedevil.com/image/KBE2_3.png)
![KBE2_3_2.png](http://blog.sensedevil.com/image/KBE2_3_2.png)

当被控制了，那么客户端可以直接发送onUpdateDataFromClient消息对被控制者进行update，在cellApp上这条消息回调方法里会进行判断的

![KBE2_4.png](http://blog.sensedevil.com/image/KBE2_4.png)

BaseAppMgr与CellAppMgr如何做到负载均衡？

CellApp或者BaseApp，每次在handleGameTick，也就是每一帧都会调用updateLoad，updateLoad会计算出一个可以表现负载的一个值，

然后再调用onUpdateLoad()，onUpdateLoad里会将此值和APP的标记同步和当前app的entities的数量同步到相关的AppMgr上，例如下：

![KBE2_5.png](http://blog.sensedevil.com/image/KBE2_5.png)

Appmgr接收方：

![KBE2_6.png](http://blog.sensedevil.com/image/KBE2_6.png)
![KBE2_7.png](http://blog.sensedevil.com/image/KBE2_7.png)

KBE的H5插件是客户端的实体ID读取方式是以智能优化的方式，而不是根据服务端的配置进行相应方式读取，

所以服务端为了配合客户端的智能优化，需要将aliasEntityID和entitydefAliasID开启

所有Cell上的entity都拥有EntityCoordinateNode，coordinateNode的flag是COORDINATE_NODE_FLAG_ENTITY

只有当entity被调用installCoordinateNodes时，才会给entity的EntityCoordinateNode设置坐标系统

![KBE2_8.png](http://blog.sensedevil.com/image/KBE2_8.png)

问题：假如avatar位置设置为0、0、0，space创建了cell的entity，为什么在avatar的onEnterAoI没有space这个entity？

因为space没有手动设置hasClient为True或者space的属性没有跟CELL的client相关的标志。代码中的判断如下

![KBE2_9.png](http://blog.sensedevil.com/image/KBE2_9.png)

服务端关服：

![KBE2_10.png](http://blog.sensedevil.com/image/KBE2_10.png)

## 信号回调函数

这里指的信号是一种软中断，它即可以作为进程间通信的一种机制，更重要的是，信号总是中断一个进程的正常运行，

它更多的用于处理一些非正常情况。例如 ctrl+c 就是一个向进程发一个信号，信号是异步的，进程并不知道信号什么时候到达，

进程既可以处理信号，也可以发送信号给特定的进程，每个信号都有一个名字，这些名字以SIG开头，例如SIGABRT是进程异常终止信号。

硬件异常产生的信号有例如除数为0、无效的存储访问等，这些条件通常由硬件检测到，并将其通知到内核，

然后内核为该条件发生时正在运行的进程产生相应的信号

软件产生的异常信号可以用kill、raise、alarm、settimer、sigqueue产生信号

## 信号的种类：

### 不可靠信号：

Linux信号机制基本上是从Unix系统中继承过来的。早期Unix系统中的信号机制比较简单和原始，后来在实践中暴露出一些问题，

因此，把那些建立在早期机制上的信号叫做"不可靠信号"，信号值小于SIGRTMIN的叫不可靠信号(1~31)。

每次信号处理后，该信号对应的处理函数会恢复到默认值。但现在的Linux已经对其进行了改进，信号处理函数一直是用户指定的或者是系统默认的。

信号可能丢失。

不可靠信号不支持信号排队，同一个信号产生多次，只要程序还未处理该信号，那么实际只处理此信号一次。

 
### 可靠信号：

信号值位于SIGRTMIN和SIGRTMAX之间的信号都是可靠信号，可靠信号克服了信号可能丢失的问题 。

实时信号与非实时信号：Linux目前定义了64种信号（将来可能会扩展），前面32种为非实时信号，后32种为实时信号。非实时信号都不支持排队，

都是不可靠信号，实时信号都支持排队，都是可靠信号。

信号排队意味着无论产生多少次信号，信号处理函数就会被调用同样的次数。

 

Kbe这里监听的SIGINT和SIGHUP是系统信号，SIGINT信号当终端输入了中断字符ctrl+c，默认系统处理是进程终止掉，SIGHUP信号也是差不多，

终端关闭时会产生这个信号，默认处理一样，还有其他系统信号，剩下想了解，自己百度，

Kbe这里就是用的信号注册，当接收到这个信号，可以做自己的关服处理，linux终端下 输入kill–l 就有列出所有的信号了

 

Machine:是机器服，每一台物理机都要开一个machine进程，machine是负责本物理服的所有进程的管理，如果需要配局域网内一组多物理服的话，

那么*mgr、interface、logger的进程在这一组服内只需要各进程开一个即可，

 
第三方登录接口：如果在kbengine.xml或kbengine_def.xml里配置了interfaces的host port那么，dbmgr在initInterfacesHandler()时

会有两个类来对应登录时选择哪个类


![KBE2_11.png](http://blog.sensedevil.com/image/KBE2_11.png)

如果接了第三方，那么需要在kbengine.xml或者kbengine_def.xml配置host和post，那么dbmgr在loginAccount时会先请求interface，代码如下：

![KBE2_12.png](http://blog.sensedevil.com/image/KBE2_12.png)

Interfaces收到onAccountLogin()后调用了脚本层的onRequestAccountLogin,也将需要的参数传给脚本层的方法中，代码如下

![KBE2_13.png](http://blog.sensedevil.com/image/KBE2_13.png)

在脚本层中的处理完相关数据后要调用引擎层的accountLoginResponse，

脚本层调用方式：

![KBE2_14.png](http://blog.sensedevil.com/image/KBE2_14.png)

引擎层代码：

![KBE2_15.png](http://blog.sensedevil.com/image/KBE2_15.png)

在interfaces返回给dmgr的onLoginAccountCBBFromInterfaces接口调用的onAccountLoginCB函数里，

如果success错误码不是由SERVER_ERR_LOCAL_PROCESSING，那么DBTaskAccountLogin不会检查密码，看needCheckPassword变量，代码如下

![KBE2_16.png](http://blog.sensedevil.com/image/KBE2_16.png)

interfaces那边返回的错误码（suceess）是成功的话，那么在DBTaskAccountLogin的db_thread_process中

如果没有此帐号的话，DBTaskAccountLogin的db_thread_process中调用的queryAccount返回false时，那么会判断kbengine.xml或者kbengine_def.xml中

有没有配置loginAutoCreate为true 或者 （有没有配置第三方地址并且不需要检查密码），两个条件一个为true，都会自动创建帐号，代码如下

![KBE2_17.png](http://blog.sensedevil.com/image/KBE2_17.png)
![KBE2_18.png](http://blog.sensedevil.com/image/KBE2_18.png)

Dbmgr处理login服发送过来的帐号登陆查询时，会查询entitylog表，是否有在线记录，有的话，会将componentID和entityID记录下

代码如下：

![KBE2_19.png](http://blog.sensedevil.com/image/KBE2_19.png)

Login服收到onLoginAccountQueryResultFromDbmgr，假如帐号没问题的话，会根据发送过来的componentID和entityID进行判断，

这里的componentID不是客户端的，而是BaseApp的componentID，意味着这个账户是登录在哪个BaseApp上，已经在线与不在线有两个处理方式，代码如下：

![KBE2_20.png](http://blog.sensedevil.com/image/KBE2_20.png)

onLogOnAttempt：当玩家在别处登录时，会通知脚本异常登录请求有脚本决定是否允许这个通道强制登录，调用与Account entity的onLogOnAttempt

 
与客户端的心跳超时：

因为Channel初始化的时候，不管是客户端还是服进程的channel，都会启动不活跃检测，注册检测定时器，超时参数的话，读kbengine配置的，

定时器回调会判断是否是超时，判断条件是这个channel的lastReceivedTime_成员变量是否超出超时参数，如果超时，那么将做超时处理，

将此条channel销毁掉，代码如下：

![KBE2_21.png](http://blog.sensedevil.com/image/KBE2_21.png)
![KBE2_22.png](http://blog.sensedevil.com/image/KBE2_22.png)
![KBE2_23.png](http://blog.sensedevil.com/image/KBE2_23.png)
![KBE2_24.png](http://blog.sensedevil.com/image/KBE2_24.png)

当然kbe底层，包括客户端插件也做了心跳处理，会每隔n时间向目标发送onAppActiveTick消息，ServerApp接收到这条消息后，

更新了接收数据时间，代码如下：

![KBE2_25.png](http://blog.sensedevil.com/image/KBE2_25.png)
