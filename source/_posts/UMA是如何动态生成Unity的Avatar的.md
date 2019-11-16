---
title: UMA是如何动态生成Unity的Avatar的
date: 2019-11-16 14:05:51
tags: [UMA, Unity]
categories: Unity
---

1. ## 名词解释

   + `UnityEngine.SkeletonBone`:
  
        + 是一个Struct
        + Details of the Transform name mapped to the skeleton bone of a model and its default position and rotation in the T-pose.
        + 表示模型中的骨骼，其中带有骨骼的名字，还有在TPose时的平移缩放旋转信息。

   + `UnityEngine.HumanBone`:

        + 是一个Struct
        + The mapping between a bone in the model and the conceptual bone in the Mecanim human anatomy.
        + 表示模型中的骨骼与Mecanim人体解剖学中的概念性骨骼之间的映射，其中模型中骨骼的名字，和Mecanim骨骼的名字，还有这个定义了这个骨骼的肌肉的旋转极限的`HumanLimit`。

   + `UnityEngine.HumanLimit`:

        + 是一个Struct
        + This class stores the rotation limits that define the muscle for a single human bone.
        + 表示一根骨骼的肌肉的旋转极限。

   + `UMA.UmaTPose`:

        + UMA->Extract T-Pose生成的文件，就是由这个类的实例序列化来的。
        + 是一个Classbaok
        + UMA用来表示和存储Unity中的Avatar的一个类，其中包含了：

          + `SkeletonBone`的数组，用来记录在TPose时所有模型骨骼的信息（包括在config unity avatar时所有的节点，比如Model节点等，不光光是代表骨骼的节点）。
          + `HumanBone`的数组，用来记录模型骨骼和Mecanim骨骼的一一映射关系。
          + 还包含了Muscles设置中的一些配置信息。

   + `UnityEngine.HumanDescription`:

        + 是一个Struct
        + 其实就是Unity版的UmaTPose，其中包含了`SkeletonBone`的数组和`HumanBone`的数组和其他的一些avatar配置，可以用这个来创建一个新的avatar。

     <!-- more -->

2. ## 生成UMA用的TPose文件

    ExtractTPose()函数中包含了两种提取TPose的方法，这个函数是在Unity的Config Avatar界面执行的。

    + 把选中的asset变成一个ModelImporter，然后从modelImporter.humanDescription中读取HumanDescription信息。用这种方法只能提取出avatar的其他配置信息，而关键的`SkeletonBone`的数组和`HumanBone`的数组是得不到的。<font color=#F08080>_而且实际运行中，这部分代码并没有执行到_</font>。
    + 全局找带有Animator的GameObject，并且从这个GameObject和它的下属层次结构中得到`SkeletonBone`的数组和`HumanBone`的数组。这个方法不能得到avatar的其他配置信息，但是没关系，这些信息但用到之前会赋予和unity中一样的默认值。<font color=#F08080>_实际运行中，UMA是用了这个方法来提取TPose的_</font>。因为在Config Avatar界面，模型就是在TPose的。

3. ## 运行时动态生成Avatar

    + 当UMA检测到人物的骨架发生变化时（通过DNA调整），就会重建Avatar。
    + UMA会将骨架重置到没有运用DNA修改的状态，就是去掉DNA的功能。
    + UMA会利用提取出来的TPose文件，让骨架回到TPose的状态。
    + 然后再运用DNA的修改。
    + 最后，在这个状态下，读取当前的模型骨骼的信息，填入到`SkeletonBone`的数组，`HumanBone`的数组保持TPose文件中的样子。然后利用这些信息生成新的avatar，最后设置给Animator。
  