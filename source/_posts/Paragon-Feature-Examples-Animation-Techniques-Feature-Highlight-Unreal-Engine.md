---
title: >-
  Paragon Feature Examples: Animation Techniques | Feature Highlight | Unreal
  Engine
date: 2019-05-22 09:36:36
tags: [UE4, Animation]
categories: GameEngine
---
<iframe width="560" height="315" src="https://www.youtube.com/embed/1UOY-FMm-xo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

主要内容：该视频由Paragon游戏制作者Laurent Delayen(Senior Programmer, Gameplay)和RayArnett(Senior Artist, Animation)讲述制作过程中使用到的动画技术。

包括：Transitions（动画过渡），Synchronize marker（同步标记），Turn（转向动作），Speed Warping（使用IK根据速度调整步幅），Slope Warping（使用IK根据斜坡斜率调整脚步位置），Jump动作的制作，以及AnimGraph。

声明：以下英文部分为个人听力记录，视频无字幕，不保证准确性，其中带问号的为听不清的单词。中文翻译部分更不保证准确性，建议结合视频看英文部分。

24'50'' **Transitions**.Our idea is to do motion prediction. Predict where the character is going to stop. Gives a few frame to do the anticipation for the stop. When we do the starts, we drop the marker and look backwards. Drop the marker location in the world, and we are using them to synchronizing our animations. We translate the physical movement to a curve, a distance curve.

In the case of start transition, look backing time and where the marker was, that curve describes the distance of the actor to that marker.

我们的想法是做动作预测。预测角色将要停止的位置。用一些动作帧来做停止动作。当做起步动作时，在起点放标记然后往回看。在世界中放下标记位置，用他们来同步我们的动作。我们把物理移动转换成一个距离曲线。

在起步的过渡时，往回看标记所在的位置，曲线描述actor到标记的距离。

26'46'' 使用DistanceCurve保存Actor to Marker的位置。

28'21'' Backward动作的Transition.

28'50'' **Pivoting**。两个动作的结合。Reach the pivot and leave that pivot.

29'39'' when we call the actor in the studio, remap the recorded motion to the map. Using the distance, we get really precise foot placement, no foot sliding at all.

30'15'' For people who are familiar with root motion, like the animation tells the capsule whereto go. If I cross the room in the animation, the capsule will follow the animation along. With this kind of foot slide ahead(?) basically the capsule is gonna cross the room, and looks that curve it say, it walks? across the room,am I in this animation, and plays that for aim. so uses the motion of the capsule to pick which frame to play in the animation. the animation stay is locked. to the capsule is doing movement. If the capsule is sitting still and starts moving forwards , the animation says OK, 2 units from where I started,I'll find that on this curve and play that for aim. 

对熟悉RootMotion的人来说，动画告诉胶囊该移动到哪里。如果人物在动画中穿过房间，胶囊会跟随动画。所以用胶囊的运动来选取哪一帧来在动画中播放。在胶囊运动的过程中，动画所在位置是锁定的。如果胶囊还在原地准备向前移动，动画说距离出发的位置有2个单位距离，会在曲线上找到那帧来播放。

33' synchronize marker? 从start到loop动作。当toe和root交叉时，有标记。

33'47'' The reason for doing that way is, they sort of tells us, spash? you where the foot is and we can synchronize animation that way. We thought about this, other people seem to describe the movement when the foot touch the ground and leave the ground,sort of like the anpitute? of the step. and we decided to go instead with the spash? where it was around the player to minimize foot sliding. So when you transition between animations, we sort of try to get the closest position. to where the capsule is basically to minimize sliding. So we sacrifice bits the where the foot is, we try to keep the position.  

35'10'' 以脚的动作为标准，可以保证不会滑步。

37'50'' **转向**。

38'50'' 角度Curve。

41'02'' 动画蓝图。

42'40''Backward-Forward转换。DistanceCurve标记Turn角度。

43'40'' .速度低会导致动作慢，用Speed Warping可以小步移动。

45'55'' 显示Speed Warping和无Speed Warping的区别。

51' 原理：根据SpeedScale移动IKBone。

53'24'' 比如2X速率移动，IKBone在2X距离位置

54'50'' 后退动作的有无SpeedWarping比较

58' BlendSpace。JogForwardSlopeLean。slope斜率。左右倾斜角度。速度

60' Jogging状态机节点。Blend->AimOffset->RotateRoot

60'50'' 移动效果。

62'40'' Slope移动效果。

64'43'' **SlopeWarping**开关的区别。

66'41'' 斜坡上开启网格模式的效果。能看到地板位置，Normal，IKBone的位置

68'14'' TwoBoneIK for legs蓝图节点

68'47'' 改变角度观察BlendSpace

70'50'' 脚悬空不可避免。

73'30'' **Jump动作**。Jumping分解为3个部分。InAir，Apex，Landing。用DistanceCurve标记到地面的距离。We use it here to get a few extra frames of compression. Because animation runs after physics. So when we note that you are jumping, it's already too late you are already in jumping.  So this allows it to compensate and have a few extra frames that feet on the ground.

Because the capsule is moving.

74'53'' Jump动作演示。

75'43''DistanceCurveForLanding. Put a marker on where you are gonna lands. Use that to synchronize the feet getting close to the ground. The arc is synchronizing with the apex?

76'06'' Apex。DistanceCurve. How far before it and go past it.

76'30'' **Recovery Additive**.Play on top of anything in the game. 为了nice landing compression.如果不用additive。会Blending Jogforward, backward。Maybe the result is not what would be blended exactly, you still get a nice feel without making a time to time content.

77'38'' when we do blend between animations, it have difference. Blend feet quickly, Upperbody lowly. 

78'50'' AnimDynamics. Not show. 可以在另一个视频中看到专门讲这个节点。AnimDynamic features used in Paragon.[https://www.youtube.com/watch?v=5h5CvZEBBWo](https://www.youtube.com/watch?v=5h5CvZEBBWo)

79'36'' **AnimGraph**. 

## 总结：

### 一、曲线的使用。

根据曲线同步动作。如起步过渡动作中，进入过渡状态时在起点放置标记marker，actor移动时根据当前actor和marker的距离distance，在DistanceCurve上查找该Distance应该对应的帧进行播放。

如原地转身动作（Rotate）中，根据当前actor的旋转角度angle，在Curve上查找该Angle对应的帧进行播放。

### 二、Synchronize Marker 左右脚同步标记

在动画中，当toe和root重合时，添加Notify记录当前是左脚还是右脚。动画过渡时，以脚为基础，避免了滑步的出现。

### 三、IK的使用

SpeedWarping和SlopeWarping，使用IK使脚在动画中处于正确的位置。

### 四、Jump动作

由于胶囊运动，动画制作原地起跳时需要考虑胶囊的移动。Curve标记当前和地面的距离。通过Curve同步动画。
