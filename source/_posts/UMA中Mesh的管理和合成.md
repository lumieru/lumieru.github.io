---
title: UMA中Mesh的管理和合成
date: 2019-11-19 18:30:32
tags: [UMA, Unity]
categories: Unity
---
1. ## 名词解释

    + `UMA.UMABlendFrame`:

        在UMAMeshData.cs文件中

        ```csharp
        public class UMABlendFrame
        {
            public float frameWeight = 100.0f; //should be 100% for one frame
            public Vector3[] deltaVertices = null;
            public Vector3[] deltaNormals = null;
            public Vector3[] deltaTangents = null;
        };
        ```

        + 一个BlendShape可以包含很多帧，最后一帧（`frameWeight = 100.0f`）是这个BlendShape的最终形态，而中间还可以有很多过度的平滑帧（就像放动画一样）。他们一般都是按frameWeight从小到大排列的。
        + `UMABlendFrame`代表BlendShape的某一帧。在大部分情况下，一个BlendShape只有一帧。
        + `frameWeight`表示这一帧是最终形态的百分之多少。
        + `deltaVertices`数组，大小和整个base mesh的顶点数相同。数组中的每一项，表示这一帧BlendShape的顶点和baseMesh中相应顶点的位置差（offset）。baseMesh中的顶点位置加上这个deltaVertices可以得到BlendShape中顶点的绝对位置。
        + `deltaNormals`数组，意义同上，只是表示的是法线。
        + `deltaTangents`数组，意义同上，只是表示的是切线。

    + `UMA.UMABlendShape`:

        在UMAMeshData.cs文件中

        ```csharp
        public class UMABlendShape
        {
            public string shapeName;
            public UMABlendFrame[] frames;
        }
        ```

        + 表示一个完整的BlendShape，里面可以包含多个帧。

        <!-- more -->
  
    + `UMA.UMAMeshData`:

        在UMAMeshData.cs文件中

        + UMA用来表示unity中mesh data的类。
        + 主要包含了mesh的所有顶点数据（包括顶点，法线，切线，颜色，多组uv），骨骼权重信息，bindPose，三角形面的索引信息（通过SubMesh来管理），所有的BlendShapes等。
        + 这其中还有static版本的mesh数据，主要是在mesh合成时用来优化内存，减少gc用的。（每次合并mesh，都会生成一个临时的UMAMeshData来缓存数据，生成完，把数据赋给unity后就释放了）。

    + `UMA.SlotDataAsset`:

        在SlotDataAsset.cs文件中

        + Contains the immutable data shared between slots of the same type.
        + 这个类中包含了slot所用到的数据，这些数据是不可变的，而且是在相同类型的slot间共享的。
        + 在运行时，`SlotLibrary`中会有场景中用到的所有slot的SlotDataAsset，每个类型有且仅有一份。
        + 这个类的其中一个成员变量就是`UMAMeshData`类型的，保存了这个slot所用的mesh的数据，包括BlendShapes。所以生成完slot后，对应的fbx就没用了，UMA自己会保存所有相关信息。

2. ## Mesh合成过程

    + 在UMAGeneratorBuiltin.cs文件的`public virtual bool HandleDirtyUpdate(UMAData data)`函数中

        ```csharp
        if (umaData.isMeshDirty)
        {
            UpdateUMAMesh(umaData.isAtlasDirty);
            umaData.isAtlasDirty = false;
            umaData.isMeshDirty = false;
            SlotsChanged++;
            forceGarbageCollect++;

            if (!fastGeneration)
                return false;
        }
        ```

        当`umaData.isMeshDirty`为true时，就会调用`UpdateUMAMesh()`来更新Mesh。

    + `UpdateUMAMesh()`函数会调用`meshCombiner.UpdateUMAMesh()`来执行合并生成工作。这是一个虚函数，实际上会调用到`UMADefaultMeshCombiner.UpdateUMAMesh()`函数。
    + `UMADefaultMeshCombiner.UpdateUMAMesh()`函数中，会根据合并得到的材质数量，来确定最终会合成几个mesh。然后把所有用到相同材质的mesh合并成一个mesh，减少draw call。合并的结果会保存在一个临时的UMAMeshData中，最后通过函数`umaMesh.ApplyDataToUnityMesh()`赋值给unity的mesh。
    + 其中的具体合并工作是通过`SkinnedMeshCombiner.CombineMeshes()`函数来完成的。它会把所有mesh的顶点数据都合并到一个mesh中，这个比较简单。我们主要详细看一下BlendShape的合并和Bake过程。

        + 先通过函数`AnalyzeBlendShapeSources()`统计所有需要合并的BlendShape的信息，如果某个BlendShape设置了bake，那么这个统计就会跳过这个BlendShape。
        + 在函数`InitializeBlendShapeData()`中，根据统计结果，生成需要合并的target BlendShape的初始化数据，这里生成的BlendShape的顶点数是和最终要生成的Mesh的顶点数相同的，并且生成的BlendShape其实是没有作用的，所有的delta都为零。设置为bake的BlendShape会跳过这步。
        + 然后把源BlendShape中的数据，根据顶点序号的offset，设置到刚才生成的target BlendShape中。如果某个BlendShape设置了bake，那么不会执行copy这步，取而代之的会执行`BakeBlendShape()`函数。
        + 在`BakeBlendShape()`函数中，会根据权重把BlendShape直接插值合并到最终生成的Mesh中，然后这个BlendShape就不会到Unity的Mesh的BlendShape中去了，可以减小内存和加快每帧的渲染速度。

3. ## BlendShape要点

    + 带有BlendShape的Slot的顶点数越小越好，因为占有的内存少。（这部分内存是整个race共享的，和单独人物的多少没有关系）。
    + 设置BlendShape的bake后，就把BlendShape烘培到Unity的Mesh中去了，Unity的Mesh中就不会带有BlendShape了，可以减少内存消耗，加快每帧的渲染速度。
    + 如果不bake，那么每个人物的Unity的mesh中都会带有相应的BlendShape顶点数据，人物越多浪费的内存就越多。而且每帧Unity都要去把BlendShape插值计算一次，浪费渲染时间。
