---
title: KBEngine源码解读三
date: 2019-06-28 14:12:53
tags: [ServerFramework, KBEngine]
categories: GameServer
---
## BaseApp::createEntityAnywhere
```cpp
// 把Entity的类型和参数全部序列化，然后发送给Baseappmgr，由Baseappmgr::reqCreateEntityAnywhere负责创建。
BaseApp::createEntityAnywhere
// 首先会找一个没有一个实体的baseapp或者是所有baseapp中负载最小的baseapp。
// 然后再调用这个baseapp的onCreateEntityAnywhere函数
--> Baseappmgr::reqCreateEntityAnywhere
// 把数据反序列化，然后调用createEntity函数创建。创建完成后，如果发起方和创建方是相同的，
// 则直接调用_onCreateEntityAnywhereCallback返回创建结果。如果不相同，则发送消息给发起方，
// 调用发起方的onCreateEntityAnywhereCallback函数返回结果。
--> BaseApp::onCreateEntityAnywhere
--> BaseApp::onCreateEntityAnywhereCallback //调用_onCreateEntityAnywhereCallback
// 如果有设置python的callback，就调用python的callback。如果是其他baseapp创建的，则创建并保存成EntityCall。
-> BaseApp::_onCreateEntityAnywhereCallback
```

## Baseapp::createEntityAnywhereFromDBID
```cpp
Baseapp::createEntityAnywhereFromDBID
// 首先会找一个没有一个实体的baseapp或者是所有baseapp中负载最小的baseapp。
// 然后再调用这个baseapp的onGetCreateEntityAnywhereFromDBIDBestBaseappID函数
--> Baseappmgr::reqCreateEntityAnywhereFromDBIDQueryBestBaseappID
--> Baseapp::onGetCreateEntityAnywhereFromDBIDBestBaseappID
// 常见DBTaskQueryEntity，添加到bufferedDBTasksMaps_中，然后有task完成任务
--> Dbmgr::queryEntity
// 从数据库中查询相关Entity的信息
-> DBTaskQueryEntity::db_thread_process
// 回到主线程，因为query mode是1，所以调用onCreateEntityAnywhereFromDBIDCallback
-> DBTaskQueryEntity::presentMainThread
--> Baseapp::onCreateEntityAnywhereFromDBIDCallback
// 消息中包含了要在那个Baseapp上创建，就是前面决定的那个Baseapp。
--> Baseappmgr::reqCreateEntityAnywhereFromDBID
// 创建Entity，并调用回调函数onCreateEntityAnywhereFromDBIDOtherBaseappCallback，
// 如果是同一个Baseapp就直接回调，否则就是RPC回调
--> Baseapp::createEntityAnywhereFromDBIDOtherBaseapp
// 如果有python的callback id，就调用python的callback函数。
--> Baseapp::onCreateEntityAnywhereFromDBIDOtherBaseappCallback
```

## Baseapp::createCellEntityInNewSpace
```cpp
// 在cellapp上创建一个空间(space)并且将该实体的cell创建到这个新的空间中，它请求通过cellappmgr来完成。
Baseapp::createCellEntityInNewSpace
// 如果CellappIndex > 0，那么用这个index模总的cellapp数量，得到目标cellapp。
// 如果CellappIndex == 0，那么找到负载最小的cellapp作为目标cellapp。
--> Cellappmgr::reqCreateCellEntityInNewSpace
// 通过SpaceMemorys::createNewSpace(spaceID, entityType)创建一个新的SpaceMemory（就是一个space）。
// 并且创建Cellapp上的Entity，设置Baseapp的EntityCall，如果有Client，则还设置Client的EntityCall，并创建Witness。
// 把新创建的Entity加入到SpaceMemory中。
// 在SpaceMemory的构造函数/析构函数中，还会调用Cellappmgr::updateSpaceData。Cellappmgr好像也维护了
// 每个Cellapp中的Space信息，所有需要更新一下。
--> Cellapp::onCreateCellEntityInNewSpaceFromBaseapp
--> Baseapp::onEntityGetCell
```

## Baseapp::createCellEntity
```cpp
// 请求在一个cell里面创建一个关联的实体。
Baseapp::createCellEntity
--> Cellapp::onCreateCellEntityFromBaseapp
// 根据spaceID找到对应的SpaceMemory，然后在该SpaceMemory中创建Entity，
// 设置Baseapp的EntityCall，如果有Client，则还设置Client的EntityCall，并创建Witness。
// 把新创建的Entity加入到SpaceMemory中。
-> Cellapp::_onCreateCellEntityFromBaseapp
--> Baseapp::onEntityGetCell
```

<!-- more -->

## Cell

目前Cellapp中的Cell和Cells都只有很简单的骨架，没有任何实际功能，也基本没有其他地方用到。所以只是一个占位符，没有相应的功能。

## Space

Baseapp中也有个Sapce（是Entity的子类），它其实只是Cellapp中Space的一个句柄，用来关联操作Cellapp中的Space的。真正的Space是在Cellapp中的。Cellapp中的Space的c++类是SpaceMemory（也有个Space，但是里面没有东西，也没用到）。
SpaceMemory中包含了所有在这个Space中的Entity（Cellapp中的Entity）。

## CoordinateSystem

有x，y，z三轴的CoordinateNode*的双向链表组成。每插入一个CoordinateNode，先放入三个双向链表的头，然后在update(Node)。每个双向链表中的Node，都是按照x，y或者z值从小到大排序的。如果xyz之发生了变化，那么update(Node)会跟新链表，让其有序。`EntityCoordinateNode`继承自CoordinateNode，关联了一个Entity。

## SpaceViewer

将Entity的position和rotation发送给Console，在GUIConsole的SpaceViewer里面可以查看Entity的信息

## RangeTrigger

需要一个中心点CoordinateNode，和xz范围，和y范围。然后会创建两个RangeTriggerNode来确定范围，如果进入范围了，会调用onEnter(CoordinateNode * pNode)虚函数，如果离开范围了，会调用onLeave(CoordinateNode * pNode)虚函数。

### ViewTrigger

RangeTrigger的子类，通过实现onEnter/onLeave，把onEnter/onLeave的消息传递给关联的Witness（这个是RangeTrigger的origin对应的Entity的Witness）。调用Witness::onEnterView/Witness::onLeaveView。

### TrapTrigger

RangeTrigger的子类，通过实现onEnter/onLeave，把onEnter/onLeave的消息传递给关联的ProximityController。调用ProximityController::onEnter/Witness::onLeave。

## ProximityController

包含一个TrapTrigger，通过onEnter/onLeave，调用关联的Entity的onEnterTrap/onLeaveTrap

## Witness

Witness就是客户端在Cellapp中的一个代理，cellapp将实体的View内的其他实体的信息不断的通过Witness同步给客户端。Entity A进入Entity B的viewRadius_范围内，才算进入了Entity B的View。而Entity A要离开Entity B的View，需要离开Entity B的viewRadius_ + viewHysteresisArea_才行。Witness会将进入本View的Entity记录在viewEntities_和viewEntities_map_中。在update()中，会把所有在view中的Entity的信息发送给本Entity的客户端。如果关掉了coordinate_system，那么View的功能就不起作用了，即本View中没有其他的Entity，也不会同步了。Entity A进入Entity B的View后，在Entity A的witnesses_中会记录Entity B的ID，表示Entity A被Entity B目击到了。

有三种状态，一（将要进入视野范围内）：其他玩家进入拥有者玩家的视野范围，那么将其他玩家在拥有者玩家的Witness里的引用的状态更改成普通状态，并同步其他玩家的客户端信息与位置信息给拥有者玩家，通知拥有者玩家有其他玩家进入视野范围；二（将要离开视1野范围内）：其他玩家离开拥有者玩家的视野范围，那么通知拥有者玩家有其他玩家要离开，并删除其他玩家在拥有者玩家的Witness里的引用；三（普通状态，还在视野范围内），同步还在视野范围的其他玩家的volatile信息给拥有者玩家。一直在update

## Updatable

用来描述一个总是会被更新的对象。Updatables用来管理所有的Updatable。Cellapp每个tick都会调用Updatables::update, 这个函数会调用所有的Updatable来更新状态。需要实现不同的Updatable来完成不同的更新特性。

## Controllers

每个Entity都包含一个Controllers的实例，来管理所有的Controller。Entity有对应的函数来创建对应的Controller，然后添加到Controllers实例中来管理。目前有的Controller有MoveController，ProximityController和TurnController。如果Controller需要每帧更新，那么是通过继承Updatable来实现的。

## globalData

globalData是通过`GlobalDataClient`和`GlobalDataServer`来实现的。Dbmgr在初始化的时候会创建三个`GlobalDataServer`的实例，分别对应globalData，baseAppData和cellAppData。`EntityApp`在初始化的时候，会创建`GlobalDataClient`对应globalData的`GlobalDataServer`。`Baseapp`在初始化时，会创建`GlobalDataClient`对应baseAppData的`GlobalDataServer`。`Cellapp`在初始化时，会创建`GlobalDataClient`对应cellAppData的`GlobalDataServer`。然后只要client的map发生变化，会自动同步到所有的baseApp（对应baseAppData），或者所有的cellApp（对应cellAppData），或者所有的baseApp和cellApp（对应globalData）。

## EntityCall

`EntityCall`和`EntityComponentCall`都继承自`EntityCallAbstract`，两者的行为很类似。只是`EntityCall`是可以远程调用Entity的方法，而`EntityComponentCall`是远程调用EntityComponent的方法。`EntityCallAbstract`中存储了远端机器的COMPONENT_ID和Address，在调用远程方法时（`EntityCallAbstract::sendCall`），会通过这两个属性找到对应的Channel，然后通过这个Channel把调用消息发送给对应的远程主机。当然其中还包括ENTITY_ID和对应方法的信息。在构建远程方法时（`EntityCallAbstract::newCall_`），如果远程主机是Baseapp，则会将目标函数`BaseApp::onEntityCall`编码到消息中，如果是Cellapp，则目标函数是`CellApp::onEntityCall`。如果是客户端远程调用服务器的方法，则如果是BaseApp的方法，那么目标方法是`BaseApp::onRemoteMethodCall`，如果是CellApp的方法，那么目标方法是`BaseApp::onRemoteCallCellMethodFromClient`。

下面以`BaseApp::onEntityCall`为例讲解后面的步骤。在这个方法中，会先根据ENTITY_ID从本进程的所有Entity中找到对应的Entity。然后根据ENTITYCALL_TYPE来做不同的处理。如果是ENTITYCALL_TYPE_BASE，则直接调用这个Entity的`onRemoteMethodCall`方法（这个方法会调用对应的python方法）。如果是ENTITYCALL_TYPE_CELL_VIA_BASE，则会通过`pEntity->cellEntityCall()`得到cell部分的EntityCall，然后在转发这个EntityCall给Cell部分。如果是ENTITYCALL_TYPE_CLIENT_VIA_BASE，则会通过`pEntity->clientEntityCall()`得到对应Client的EntityCall，然后转发这个EntityCall给Client。

## 注册流程

```cpp
Unity3d:CreateAccount
--> Loginapp::reqCreateAccount 
// 看看注册是否开放，传入的注册信息是否符合要求，是否已经有相同的账号还在注册中的。
// 还会调用python层的回调函数onRequestCreateAccount来处理。
// 如果中途失败了，则直接调用ClientInterface::onCreateAccountResult返回错误。
-> Loginapp::_createAccount 
// 通过findBestInterfacesHandler()找到一个Interface来处理，如果没有第三方Interface，
// 那么默认是InterfacesHandler_Dbmgr来处理，如果有第三方Interface。那么是InterfacesHandler_Interfaces
// 来处理。InterfacesHandler_Interfaces::createAccount会把消息转发给第三方Interface来处理。
// 下面的流程针对InterfacesHandler_Dbmgr来说的
--> Dbmgr::reqCreateAccount
// 会根据是不是mail账号，分别创建DBTaskCreateMailAccount或者DBTaskCreateAccount，
// 然后添加到ThreadPool中，有task来完成相应的任务。下面以DBTaskCreateAccount为例说明。
-> InterfacesHandler_Dbmgr::createAccount
-> DBTaskCreateAccount::writeAccount // 创建新账号，存入数据库，这个函数在其他线程中执行
-> DBTaskCreateAccount::presentMainThread // 回到主线程，把创建结果返回给Loginapp
--> Loginapp::onReqCreateAccountResult // 把结果交给python回调函数处理onCreateAccountCallbackFromDB
--> Client::onCreateAccountResult
```

## 登录流程

### 登录 Step1
```cpp
Unity3d:login
// 检查数据合法性，调用python回调函数onRequestLogin，
// 如果中途出错了，直接调用ClientInterface::onLoginFailed返回错误
--> Loginapp::login
// 通过findBestInterfacesHandler()找到一个Interface来处理，如果没有第三方Interface，
// 那么默认是InterfacesHandler_Dbmgr来处理，如果有第三方Interface。那么是InterfacesHandler_Interfaces
// 来处理。InterfacesHandler_Interfaces::loginAccount会把消息转发给第三方Interface来处理（InterfacesInterface::onAccountLogin）。
// 下面的流程针对InterfacesHandler_Dbmgr来说的
--> Dbmgr::onAccountLogin
// 创建DBTaskAccountLogin加入到ThreadPool中，有task来完成任务
-> InterfacesHandler_Dbmgr::loginAccount
-> DBTaskAccountLogin::db_thread_process // 查询数据库，验证账号合法性，这个函数在其他线程中执行。
-> DBTaskAccountLogin::presentMainThread // 回到主线程，把账号信息发送回Loginapp
// 调用python的回调函数onLoginCallbackFromDB，如果componentID>0（说明当前账号仍然存活于某个baseapp上），
// 则调用Baseappmgr::registerPendingAccountToBaseappAddr，否则调用Baseappmgr::registerPendingAccountToBaseapp，
// 下面以Baseappmgr::registerPendingAccountToBaseapp来说明
--> Loginapp::onLoginAccountQueryResultFromDbmgr
// 找到当前负载最低的Baseapp，把账号信息发送给该Baseapp。
--> Baseappmgr::registerPendingAccountToBaseapp
// 得到当前Baseapp的IP和port（包括TCP和UDP），再发送回Baseappmgr
--> Baseapp::registerPendingLogin
--> Baseappmgr::onPendingAccountGetBaseappAddr
-> Baseappmgr::sendAllocatedBaseappAddr // 把IP和port等信息，发送给Loginapp
--> Loginapp::onLoginAccountQueryBaseappAddrFromBaseappmgr // 把IP和port等信息，发送给Client
--> ClientInterface::onLoginSuccessfully 
```

### 登录 Step2
```cpp
Unity3d:loginBaseapp      //尝试在指定Baseapp上登录
// 检查登录 处理重复登录 向数据库查询账号详细信息
--> Baseapp::loginBaseapp 
// 新建DBTaskQueryAccount，加入到Buffered_DBTasks，然后由task来执行操作
--> Dbmgr::queryAccount
-> DBTaskQueryAccount::db_thread_process // 查询数据库，得到账号详细信息
-> DBTaskQueryAccount::presentMainThread // 回到主线程，把信息发送给Baseapp
// 创建Proxy，并且绑定客户端的EntityCall
--> Baseapp::onQueryAccountCBFromDbmgr
// 调用pEntity->initClientBasePropertys()，在这个函数调用中，把这个Proxy的一些属性，
// 通过ClientInterface::onUpdatePropertys远程调用，发回给了客户端。
// 最后调用ClientInterface::onCreatedProxies，把proxy的ID等传给了客户端。
-> Baseapp::createClientProxies
--> ClientInterface::onCreatedProxies
```

## Backuper & Archiver

Backuper是将entity置脏，而Archiver是将数据写入db，如果没有entity没有置脏，那么不会写入db。每帧都会调用Backuper和Archiver的tick函数。他们都会生成一个需要backup/archive的Entity的列表。然后根据设置的备份时间间隔，把列表分散到每个帧中，就是每个帧处理列表中的一部分。知道列表为空了，再重新生成一份列表。至于什么会加入到列表中，是根据shouldAutoBackup_/shouldAutoArchive_这两个flag来的，如果flag>0就是true，默认这两个Flag都是1。如果flag == KBEngine.NEXT_ONLY，那么只有一次是true的，然后就会自动变成false。

如果此Entity没有dbid，那么DBMgr认为是insert类型，反之，update类型；如果是insert类型 且 Entity有个属性是ARRAY类型，那么这个会在Content的optable循环插入需要insert或update的数据。只要Entity的一个需要持久化的属性改变了，那么它就脏了，整个Entity会存档。

```cpp
// 在BaseApp上backup某个Entity
Backuper::backup
-> Entity::writeBackupData
-> Entity::onBackup
// 把需要backup的Cell Entity的ID发给Cellapp
-> Entity::reqBackupCellData
// 找到对应的Entity，调用backupCellData
--> Cellapp::reqBackupEntityCellData
// 通过函数addCellDataToStream序列化自己，然后用sha1做hash，
// 得到的hash值和persistentDigest_的hash值做比较，如果改变了，
// 就说明数据脏了，需要backup，并且把新的hash存到persistentDigest_中。
// 只有数据脏了，才会把序列化的数据发送会Baseapp。
-> Entity::backupCellData
// 找到对应的Entity，调用onBackupCellData
--> Baseapp::onBackupEntityCellData
// 如果Cell数据没脏，那么什么也不做，如果脏了，就把cell数据存下来，并且清空persistentDigest_。
-> Entity::onBackupCellData
```

```cpp
// 在Baseapp上archive某个Entity
Archiver::archive
// 什么也没做，直接调用Cellapp的reqWriteToDBFromBaseapp
-> Entity::writeToDB(NULL, NULL, NULL);
// 找到对应的Entity，调用writeToDB
--> Cellapp::reqWriteToDBFromBaseapp
// 其中callbackid是0，如果Baseapp中的Entity的DBID <= 0, 那么shouldAutoLoad就是0，否则是-1，extra2是null。
// 调用python的回调函数onWriteToDB，还调用Entity::backupCellData()，如果数据脏了，
// 就序列化并发送给Baseapp中的Entity保存。没脏，那就什么也不做。
-> Entity::writeToDB
// 找到对应的Entity，调用onCellWriteToDBCompleted。
--> Baseapp::onCellWriteToDBCompleted
// 调用python回调onWriteToDB。调用addPersistentsDataToStream，把base和cell部分的数据都序列化。
// 计算sha1，并和persistentDigest_比较，如果没脏，就直接退出了，脏了，就记录新的hash到persistentDigest_，
// 并把数据发送给Dbmgr。如果cell部分的数据脏了，那么persistentDigest_已经被清空了，所以肯定是脏的。
-> Entity::onCellWriteToDBCompleted
// 新建DBTaskWriteEntity，让这个task来处理
--> Dbmgr::writeEntity
// 把Entity数据写入数据库
-> DBTaskWriteEntity::db_thread_process 
// 返回写entity的结果， 成功或者失败
-> DBTaskWriteEntity::presentMainThread
// 调用对应Entity的onWriteToDBCallback
--> Baseapp::onWriteToDBCallback
// 如果有python的callback，就调用callback。
-> Entity::onWriteToDBCallback
```

## AllClients

所有的目击到自己的Entity。

## Entity::teleport

1. 任何形式的teleport都被认为是瞬间移动的（可突破空间限制进入到任何空间）， 哪怕是在当前位置只移动了0.1米, 这就造成如果当前entity刚好在某个trap中， teleport向前移动0.1米但是没有出trap， 因为这是瞬间移动的特性我们目前认为entity会先离开trap并且触发相关回调, 然后瞬时出现在了另一个点， 那么因为该点也是在当前trap中所以又会抛出进入trap回调.

2. `Entity::teleportLocal`如果是当前space上跳转则立即进行移动操作

3. `Entity::teleportRefEntity`如果是跳转到其他space上, 但是那个space也在当前cellapp上的情况时， 立即执行跳转操作(因为不需要进行任何其他关系的维护， 直接切换就好了)。 从当前Space移除，加入到目标Space。

4. 如果要跳转的目标space在另一个cellapp上`Entity::teleportRefEntityCall`：
   
    4.1 当前entity没有base部分， 不考虑维护base部分的关系， 但是还是要考虑意外情况导致跳转失败， 那么此时应该返回跳转失败回调并且继续
    正常存在于当前space上。

    4.2 当前entity有base部分， 那么我们需要改变base所映射的cell部分(并且在未正式切换关系时baseapp上所有送达cell的消息都应该不被丢失)， 为了安全我们需要做一些工作

    ```cpp
    // 如果这个entity有base部分， 假如是本进程级别的传送，那么相关操作按照正常的执行
    // 如果是跨cellapp的传送， 那么我们可以先设置entity为ghost并立即序列化entity发往目的cellapp
    // 如果期间有base的消息发送过来， entity的ghost机制能够转到real上去， 因此传送之前不需要对base
    // 做一些设置，传送成功后先设置base的关系base在被改变关系后仍然有0.1秒的时间收到包继续发往ghost，
    // 如果一直有包则一直刷新时间直到没有任何包需要广播并且超时0.1秒之后的包才会直接发往real）, 
    // 这样做的好处是传送并不需要非常谨慎的与base耦合
    // 传送过程中有任何错误也不会影响到base部分，base部分的包也能够按照秩序送往real。
    //
    // 有base会调用Baseapp上的Entity::onMigrationCellappStart
    Entity::teleportRefEntityCall
    // 我们需要将entity打包发往目的cellapp
    // 暂时不销毁这个entity, 把这个entity设置成ghost，等那边成功创建之后再回来销毁
    // 此期间的消息可以通过ghost转发给real
    // 如果未能正确传输过去则可以从当前cell继续恢复entity.
    -> Entity::onTeleportRefEntityCall
    // 创建一个新的Entity，并从数据中反序列化，通知Baseapp上的Entity::onMigrationCellappEnd
    // 通知客户端离开了老的space，ClientInterface::onEntityLeaveSpace。
    // 进入新的Space。把结果发送回原来的Cellapp
    --> Cellapp::reqTeleportToCellApp
    // 如果传送成功了，就销毁原来的Entity。如果失败了，就恢复Entity，并恢复成Real实体。
    --> Cellapp::reqTeleportToCellAppCB
    ```
## Entity和Proxy等python类不能有虚函数

[原文链接](https://bbs.comblockengine.com/forum.php?mod=viewthread&tid=6744&extra=page%3D1)

最近在做一些底层实体的扩展工作，发现了 如果 entity或者proxy中有虚函数的声明，那么程序在创建实体后 调用__init__方法时会抛出异常。于是我打开调试，看看为什么会这样，发现了很奇怪的事情：

![kbengine3_1.png](http://blog.sensedevil.com/image/kbengine3_1.png)

obj和entity应该是一个对象，但是调试信息里面obj的python结构是正常的，而entity的python结构是异常的。这时我才恍然大悟，原来是C++编译器给对象内存添加了虚函数表的指针导致了内存错位了。

![kbengine3_2.png](http://blog.sensedevil.com/image/kbengine3_2.png)

可以看到，python的GC头部后面应该紧接的是PyObject，但是因为有虚函数，所以C++编译器在给的对象指针顶部添加了一个虚函数表指针数据，导致entity的值比(PyObject*)entity的值小个指针大小。所以 entity 以及派送类是不允许有虚函数以及虚继承的。

总结就是能够被python继承的类，C++对象模型顶部一定是PyObject，所以该类不能有虚函数以及虚继承等 破坏这个内存模型的操作。