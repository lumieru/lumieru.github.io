---
title: KBEngine源码解读二组件互联
date: 2019-06-22 17:12:04
tags: [ServerFramework, KBEngine]
categories: GameServer
---
## server/Components

组件在互相发现的时候用的是UDP广播，找到对应组件知道其IP和Port后，用TCP来建立并保持连接。

machine组件会监听20086端口，然后其他组件都会往这个端口广播自己的信息（RPC调用`MachineInterface::onBroadcastInterface`函数）。然后machine收到信息后会判断有效性（有没有和其他组件用了相同的设置），如果无效会发回一个无效的通知给该组件，然后该组件就会退出。如果有效的则，machine会记录和自己同一台物理机上的组件到自己的组件列表中（不同物理机的组件会被忽略掉）。

然后组件会有一个需要找的组件的类型的列表，对于每种类型，它都会广播`MachineInterface::onFindInterfaceAddr`这个消息到machine。machine收到这个消息后，会把已经注册过的本地同类型组件发回给该组件。如果没有找到对应的类型，会定时到下一个循环继续找，直到找到自己感兴趣的所有类型的组件为止。

external port和telnet的port都是通过配置文件指定好的，internal port传的零，就是让系统决定一个可用的随机端口，bind成功后在用`getsockname()`得到具体的端口。

下面是每个组件感兴趣的列表:

| 组件类型 | 感兴趣的类型 |
| ----------- | ----------- |
| Cell app | logger, dbmgr, cellAppMgr, baseAppMgr |
| Base app | logger, dbmgr, baseAppMgr, cellAppMgr |
| Base app mgr | logger, dbmgr, cellAppMgr |
| Cell app mgr | logger, dbmgr, baseAppMgr |
| db mgr | logger |

在连接其他组件时，会调用被连接组件的`XXX::onRegisterNewApp`函数。当baseApp或者cellApp连接`Dbmgr`时，同样会调用`Dbmgr::onRegisterNewApp`。在这个函数中，如果连接者是baseApp或者cellApp，那么就会将自己注册到所有其他baseapp和cellapp中。主要是通过遍历已经注册在Dbmgr中的其他baseapp和cellapp，然后RPC调用相应的`onGetEntityAppFromDbmgr`。在被调用的`onGetEntityAppFromDbmgr`中，被调用者会去连接当前组件。这样就能让所有的baseApp和cellApp互相连接了。

<!-- more -->

## 运行时每个组件的互联情况：

在Windows命令行中运行命令
```shell
netstat -ano | findstr <PID>
```
可以得到相应程序的网络活动数据。

db mgr：

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:32000        |  0.0.0.0:0            |  LISTENING    |   684| telnet |
 | TCP  |  0.0.0.0:50277        |  0.0.0.0:0            |  LISTENING    |   684| internal TCP |
 | TCP  |  127.0.0.1:50279      |  127.0.0.1:30099      |  ESTABLISHED  |   684| interfaces |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50297  |  ESTABLISHED  |   684| cell app mgr |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50299  |  ESTABLISHED  |   684| base app mgr |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50301  |  ESTABLISHED  |   684| login app |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50303  |  ESTABLISHED  |   684| cell app 1 |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50305  |  ESTABLISHED  |   684| base app 2 |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50310  |  ESTABLISHED  |   684| base app 3 |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50315  |  ESTABLISHED  |   684| base app 1 |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50321  |  ESTABLISHED  |   684| cell app 2 |
 | TCP  |  169.254.43.79:50277  |  169.254.43.79:50328  |  ESTABLISHED  |   684| cell app 3 |
 | TCP  |  169.254.43.79:50295  |  169.254.43.79:50267  |  ESTABLISHED  |   684| logger |
 | TCP  |  [::1]:50281          |  [::1]:3306           |  ESTABLISHED  |   684| mysqld |
 | TCP  |  [::1]:50282          |  [::1]:3306           |  ESTABLISHED  |   684| mysqld |
 | TCP  |  [::1]:50283          |  [::1]:3306           |  ESTABLISHED  |   684| mysqld |
 | TCP  |  [::1]:50284          |  [::1]:3306           |  ESTABLISHED  |   684| mysqld |
 | TCP  |  [::1]:50285          |  [::1]:3306           |  ESTABLISHED  |   684| mysqld |

base app 1

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:20015        |  0.0.0.0:0            |  LISTENING    |   7624|  external TCP |
 | TCP  |  0.0.0.0:40001        |  0.0.0.0:0            |  LISTENING    |   7624|  telnet |
 | TCP  |  0.0.0.0:50269        |  0.0.0.0:0            |  LISTENING    |   7624|  internal TCP |
 | TCP  |  169.254.43.79:50269  |  169.254.43.79:50316  |  ESTABLISHED  |   7624|  base app 2 |
 | TCP  |  169.254.43.79:50269  |  169.254.43.79:50317  |  ESTABLISHED  |   7624|  base app 3 |
 | TCP  |  169.254.43.79:50269  |  169.254.43.79:50318  |  ESTABLISHED  |   7624|  cell app 1 |
 | TCP  |  169.254.43.79:50293  |  169.254.43.79:50267  |  ESTABLISHED  |   7624|  logger |
 | TCP  |  169.254.43.79:50315  |  169.254.43.79:50277  |  ESTABLISHED  |   7624|  db mgr |
 | TCP  |  169.254.43.79:50319  |  169.254.43.79:50275  |  ESTABLISHED  |   7624|  base app mgr |
 | TCP  |  169.254.43.79:50320  |  169.254.43.79:50272  |  ESTABLISHED  |   7624|  cell app mgr |
 | TCP  |  169.254.43.79:50324  |  169.254.43.79:50274  |  ESTABLISHED  |   7624|  cell app 2 |
 | TCP  |  169.254.43.79:50331  |  169.254.43.79:50276  |  ESTABLISHED  |   7624|  cell app 3 |
 | UDP  |  0.0.0.0:20005        |  \*:\*                |               |   7624|  external UDP |

base app 2

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:20016        |  0.0.0.0:0            | LISTENING     |  13068|  external TCP  |
 | TCP  |  0.0.0.0:40000        |  0.0.0.0:0            | LISTENING     |  13068|  telnet |
 | TCP  |  0.0.0.0:50270        |  0.0.0.0:0            | LISTENING     |  13068|  internal TCP |
 | TCP  |  169.254.43.79:50270  |  169.254.43.79:50306  | ESTABLISHED   |  13068 |  cell app 1 |
 | TCP  |  169.254.43.79:50291  |  169.254.43.79:50267  | ESTABLISHED   |  13068 |  logger |
 | TCP  |  169.254.43.79:50305  |  169.254.43.79:50277  | ESTABLISHED   |  13068 |  db mgr |
 | TCP  |  169.254.43.79:50307  |  169.254.43.79:50275  | ESTABLISHED   |  13068 |  base app mgr |
 | TCP  |  169.254.43.79:50309  |  169.254.43.79:50272  | ESTABLISHED   |  13068 |  cell app mgr |
 | TCP  |  169.254.43.79:50311  |  169.254.43.79:50271  | ESTABLISHED   |  13068 |  base app 3 |
 | TCP  |  169.254.43.79:50316  |  169.254.43.79:50269  | ESTABLISHED   |  13068 |  base app 1 |
 | TCP  |  169.254.43.79:50322  |  169.254.43.79:50274  | ESTABLISHED   |  13068 |  cell app 2 |
 | TCP  |  169.254.43.79:50329  |  169.254.43.79:50276  | ESTABLISHED   |  13068 |  cell app 3 |
 | UDP  |  0.0.0.0:20006        |  \*:\*                |               |  13068 |  external UDP |

base app 3

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:20017        |  0.0.0.0:0            |  LISTENING    |  3412|  external TCP  |
 | TCP  |  0.0.0.0:40002        |  0.0.0.0:0            |  LISTENING    |  3412|  telnet |
 | TCP  |  0.0.0.0:50271        |  0.0.0.0:0            |  LISTENING    |  3412|  internal TCP |
 | TCP  |  169.254.43.79:50271  |  169.254.43.79:50311  |  ESTABLISHED  |  3412|   base app 2 |
 | TCP  |  169.254.43.79:50271  |  169.254.43.79:50312  |  ESTABLISHED  |  3412|   cell app 1 |
 | TCP  |  169.254.43.79:50294  |  169.254.43.79:50267  |  ESTABLISHED  |  3412|   logger |
 | TCP  |  169.254.43.79:50310  |  169.254.43.79:50277  |  ESTABLISHED  |  3412|   db mgr |
 | TCP  |  169.254.43.79:50313  |  169.254.43.79:50275  |  ESTABLISHED  |  3412|   base app mgr |
 | TCP  |  169.254.43.79:50314  |  169.254.43.79:50272  |  ESTABLISHED  |  3412|   cell app mgr |
 | TCP  |  169.254.43.79:50317  |  169.254.43.79:50269  |  ESTABLISHED  |  3412|   base app 1 |
 | TCP  |  169.254.43.79:50323  |  169.254.43.79:50274  |  ESTABLISHED  |  3412|   cell app 2 |
 | TCP  |  169.254.43.79:50330  |  169.254.43.79:50276  |  ESTABLISHED  |  3412|   cell app 3 |
 | UDP  |  0.0.0.0:20007        |  \*:\*                |               |  3412|   external UDP |

cell app 1

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:50001        |  0.0.0.0:0            |  LISTENING     |  2816|  telnet |
 | TCP  |  0.0.0.0:50273        |  0.0.0.0:0            |  LISTENING     |  2816|  internal TCP |
 | TCP  |  169.254.43.79:50289  |  169.254.43.79:50267  |  ESTABLISHED   |  2816|   logger |
 | TCP  |  169.254.43.79:50303  |  169.254.43.79:50277  |  ESTABLISHED   |  2816|   db mgr |
 | TCP  |  169.254.43.79:50304  |  169.254.43.79:50272  |  ESTABLISHED   |  2816|   cell app mgr |
 | TCP  |  169.254.43.79:50306  |  169.254.43.79:50270  |  ESTABLISHED   |  2816|   base app 2 |
 | TCP  |  169.254.43.79:50308  |  169.254.43.79:50275  |  ESTABLISHED   |  2816|   base app mgr |
 | TCP  |  169.254.43.79:50312  |  169.254.43.79:50271  |  ESTABLISHED   |  2816|   base app 3 |
 | TCP  |  169.254.43.79:50318  |  169.254.43.79:50269  |  ESTABLISHED   |  2816|   base app 1 |
 | TCP  |  169.254.43.79:50325  |  169.254.43.79:50274  |  ESTABLISHED   |  2816|   cell app 2 |
 | TCP  |  169.254.43.79:50332  |  169.254.43.79:50276  |  ESTABLISHED   |  2816|   cell app 3 |

cell app 2

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:50000        |  0.0.0.0:0            |  LISTENING    |   8280|  telnet |
 | TCP  |  0.0.0.0:50274        |  0.0.0.0:0            |  LISTENING    |   8280|  internal TCP |
 | TCP  |  169.254.43.79:50274  |  169.254.43.79:50322  |  ESTABLISHED  |   8280|  base app 2 |
 | TCP  |  169.254.43.79:50274  |  169.254.43.79:50323  |  ESTABLISHED  |   8280|  base app 3 |
 | TCP  |  169.254.43.79:50274  |  169.254.43.79:50324  |  ESTABLISHED  |   8280|  base app 1 |
 | TCP  |  169.254.43.79:50274  |  169.254.43.79:50325  |  ESTABLISHED  |   8280|  cell app 1 |
 | TCP  |  169.254.43.79:50288  |  169.254.43.79:50267  |  ESTABLISHED  |   8280|  logger |
 | TCP  |  169.254.43.79:50321  |  169.254.43.79:50277  |  ESTABLISHED  |   8280|  db mgr |
 | TCP  |  169.254.43.79:50326  |  169.254.43.79:50272  |  ESTABLISHED  |   8280|  cell app mgr |
 | TCP  |  169.254.43.79:50327  |  169.254.43.79:50275  |  ESTABLISHED  |   8280|  base app mgr |
 | TCP  |  169.254.43.79:50333  |  169.254.43.79:50276  |  ESTABLISHED  |   8280|  cell app 3 |

cell app 3

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:50002        |  0.0.0.0:0            |  LISTENING     |  8600|  telnet |
 | TCP  |  0.0.0.0:50276        |  0.0.0.0:0            |  LISTENING     |  8600|  internal TCP |
 | TCP  |  169.254.43.79:50276  |  169.254.43.79:50329  |  ESTABLISHED   |  8600|  base app 2 |
 | TCP  |  169.254.43.79:50276  |  169.254.43.79:50330  |  ESTABLISHED   |  8600|  base app 3 |
 | TCP  |  169.254.43.79:50276  |  169.254.43.79:50331  |  ESTABLISHED   |  8600|  base app 1 |
 | TCP  |  169.254.43.79:50276  |  169.254.43.79:50332  |  ESTABLISHED   |  8600|  cell app 1 |
 | TCP  |  169.254.43.79:50276  |  169.254.43.79:50333  |  ESTABLISHED   |  8600|  cell app 2 |
 | TCP  |  169.254.43.79:50290  |  169.254.43.79:50267  |  ESTABLISHED   |  8600|  logger |
 | TCP  |  169.254.43.79:50328  |  169.254.43.79:50277  |  ESTABLISHED   |  8600|  db mgr |
 | TCP  |  169.254.43.79:50334  |  169.254.43.79:50272  |  ESTABLISHED   |  8600|  cell app mgr |
 | TCP  |  169.254.43.79:50335  |  169.254.43.79:50275  |  ESTABLISHED   |  8600|  base app mgr |

login app

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:20013        |  0.0.0.0:0            |  LISTENING     |  10896| external TCP |
 | TCP  |  0.0.0.0:31000        |  0.0.0.0:0            |  LISTENING     |  10896| telnet |
 | TCP  |  0.0.0.0:50278        |  0.0.0.0:0            |  LISTENING     |  10896| internal TCP |
 | TCP  |  169.254.43.79:21103  |  0.0.0.0:0            |  LISTENING     |  10896| http call back |
 | TCP  |  169.254.43.79:50296  |  169.254.43.79:50267  |  ESTABLISHED   |  10896| logger |
 | TCP  |  169.254.43.79:50301  |  169.254.43.79:50277  |  ESTABLISHED   |  10896| db mgr |
 | TCP  |  169.254.43.79:50302  |  169.254.43.79:50275  |  ESTABLISHED   |  10896| base app mgr |

cell app mgr

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:50272        |  0.0.0.0:0            |  LISTENING    |   10860|  internal TCP |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50300  |  ESTABLISHED  |   10860|  base app mgr |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50304  |  ESTABLISHED  |   10860|  cell app 1 |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50309  |  ESTABLISHED  |   10860|  base app 2 |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50314  |  ESTABLISHED  |   10860|  base app 3 |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50320  |  ESTABLISHED  |   10860|  base app 1 |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50326  |  ESTABLISHED  |   10860|  cell app 2 |
 | TCP  |  169.254.43.79:50272  |  169.254.43.79:50334  |  ESTABLISHED  |   10860|  cell app 3 |
 | TCP  |  169.254.43.79:50287  |  169.254.43.79:50267  |  ESTABLISHED  |   10860|  logger |
 | TCP  |  169.254.43.79:50297  |  169.254.43.79:50277  |  ESTABLISHED  |   10860|  db mgr |

base app mgr

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:50275        |  0.0.0.0:0            |  LISTENING     |  11652|   internal TCP |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50302  |  ESTABLISHED   |  11652|   login app |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50307  |  ESTABLISHED   |  11652|   base app 2 |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50308  |  ESTABLISHED   |  11652|   cell app 1 |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50313  |  ESTABLISHED   |  11652|   base app 3 |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50319  |  ESTABLISHED   |  11652|   base app 1 |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50327  |  ESTABLISHED   |  11652|   cell app 2 |
 | TCP  |  169.254.43.79:50275  |  169.254.43.79:50335  |  ESTABLISHED   |  11652|   cell app 3 |
 | TCP  |  169.254.43.79:50292  |  169.254.43.79:50267  |  ESTABLISHED   |  11652|   logger |
 | TCP  |  169.254.43.79:50299  |  169.254.43.79:50277  |  ESTABLISHED   |  11652|   db mgr   |
 | TCP  |  169.254.43.79:50300  |  169.254.43.79:50272  |  ESTABLISHED   |  11652|   cell app mgr |

machine

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:20099        |  0.0.0.0:0       |       LISTENING    |   12256| external TCP |
 | TCP  |  0.0.0.0:50268        |  0.0.0.0:0       |       LISTENING    |   12256| internal TCP |
 | UDP  |  0.0.0.0:20086        |  \*:\*           |                    |   12256| UDP广播接收 |
 | UDP  |  127.0.0.1:20086      |  \*:\*           |                    |   12256| UDP广播接收 |
 | UDP  |  169.254.43.79:20086  |  \*:\*           |                    |   12256| UDP广播接收 |

logger

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:34000        |  0.0.0.0:0            |  LISTENING    |   11204|   telnet |
 | TCP  |  0.0.0.0:50266        |  0.0.0.0:0            |  LISTENING    |   11204|   external TCP |
 | TCP  |  0.0.0.0:50267        |  0.0.0.0:0            |  LISTENING    |   11204|   internal TCP |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50287  |  ESTABLISHED  |   11204|   cell app mgr |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50288  |  ESTABLISHED  |   11204|   cell app 2 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50289  |  ESTABLISHED  |   11204|   cell app 1 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50290  |  ESTABLISHED  |   11204|   cell app 3 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50291  |  ESTABLISHED  |   11204|   base app 2 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50292  |  ESTABLISHED  |   11204|   base app mgr |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50293  |  ESTABLISHED  |   11204|   base app 1 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50294  |  ESTABLISHED  |   11204|   base app 3 |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50295  |  ESTABLISHED  |   11204|   db mgr |
 | TCP  |  169.254.43.79:50267  |  169.254.43.79:50296  |  ESTABLISHED  |   11204|   login app |

interfaces

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:30099      |    0.0.0.0:0           |   LISTENING     |  5324| |
 | TCP  |  0.0.0.0:33000      |    0.0.0.0:0           |   LISTENING     |  5324|   telnet |
 | TCP  |  127.0.0.1:30040    |    0.0.0.0:0           |   LISTENING     |  5324| |
 | TCP  |  127.0.0.1:30099    |    127.0.0.1:50279     |   ESTABLISHED   |  5324|   db mgr |

mysqld

 | Socket | 源地址 | 目标地址 | 状态 | PID | 目标组件 |
 | ----- | ----- | ----- | ----- | ----- | ----- |
 | TCP  |  0.0.0.0:3306       |    0.0.0.0:0         |    LISTENING    |  4808| |
 | TCP  |  0.0.0.0:33060      |    0.0.0.0:0         |    LISTENING    |  4808| |
 | TCP  |  127.0.0.1:49670    |    127.0.0.1:49671   |    ESTABLISHED  |  4808| |
 | TCP  |  127.0.0.1:49671    |    127.0.0.1:49670   |    ESTABLISHED  |  4808| |
 | TCP  |  [::]:3306          |    [::]:0            |    LISTENING    |  4808| |
 | TCP  |  [::]:33060         |    [::]:0            |    LISTENING    |  4808| |
 | TCP  |  [::1]:3306         |    [::1]:50281       |    ESTABLISHED  |  4808|   db mgr |
 | TCP  |  [::1]:3306         |    [::1]:50282       |    ESTABLISHED  |  4808|   db mgr |
 | TCP  |  [::1]:3306         |    [::1]:50283       |    ESTABLISHED  |  4808|   db mgr |
 | TCP  |  [::1]:3306         |    [::1]:50284       |    ESTABLISHED  |  4808|   db mgr |
 | TCP  |  [::1]:3306         |    [::1]:50285       |    ESTABLISHED  |  4808|   db mgr |