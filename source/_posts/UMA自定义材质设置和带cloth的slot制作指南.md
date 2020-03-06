---
title: UMA自定义材质设置和带cloth的slot制作指南
date: 2020-03-06 11:44:39
tags: [UMA, Unity]
categories: Unity
---

## UMA的贴图处理

UMA检测到贴图需要处理时，就会调用`TextureProcessPROCoroutine.cs`文件里的`workerMethod()`来合并贴图，并把最终结果输出到一个`RenderTexture`中。真正合成贴图的操作是在`TextureMerge.cs`中完成的。对于UMA提供的5中贴图类型，只有其中三种是需要调用shader来进行合并处理的，剩下的两种不处理贴图，只是把原图设置到目标材质，或者把颜色设置到目标材质。具体的操作如下表

|  贴图类型  |  操作  |  对应shader  |
|  ---  |  ---  |  ---  |
| Texture | 利用shader合图 | AtlasShader |
| DiffuseTexture | 利用shader合图 | AtlasDiffuseShader |
| NormalMap | 利用shader合图 | AtlasNormalShader |
| MaterialColor | 不处理贴图，只设置颜色 | 无 |
| TintedTexture | 不处理贴图，设置颜色和原始贴图 | 无 |

上面提到的3个shader，除了需要一张`_MainTex`外，还需要一张`_ExtraTex`，这张`_ExtraTex`是作为alpha mash来使用的，不同的shader有不同的用法。还有两个颜色参数`_Color`和`_AdditiveColor`，对应的是recipe中设置的乘法颜色和加法颜色。

这个`_ExtraTex`默认是来自于overlay中设置的Alpha Mask的，如果改overlay中没有设置Alpha Mask，那么就用改overlay的第一张贴图的alpha通过。

三个shader对于这张`_ExtraTex`的使用是不同的，具体如下：

+ AtlasShader

    ```shader
    half4 frag (v2f i) : COLOR
    {
        half4 texcol = tex2D (_MainTex, i.uv) * _Color + _AdditiveColor;
        half4 maskcol = tex2D (_ExtraTex, i.uv);
        return texcol * maskcol.a;
    }
    ```

`_ExtraTex`的alpha通道乘以`_MainTex`的所有通道，得到最终值。

<!-- more -->

+ AtlasDiffuseShader

```glsl
half4 frag (v2f i) : COLOR
{
    half4 texcol = tex2D(_MainTex, i.uv);
    half4 mask = tex2D(_ExtraTex, i.uv);
    return half4(texcol.xyz, mask.w)*_Color+_AdditiveColor;
}
```

`_ExtraTex`的alpha通道替换`_MainTex`的alpha通道，得到最终值。

+ AtlasNormalShader

```glsl
half4 frag(v2f i) : COLOR
{
    half4 n = tex2D(_MainTex, i.uv);
    half4 extra = tex2D(_ExtraTex, i.uv);

#if !defined(UNITY_NO_DXT5nm)
    //swizzle the alpha and red channel, we will swizzle back in the post process SwizzleShader
    n.r = n.a;
#endif
    n.a = min(extra.a, _Color.a);
    return n;
}
```

`_ExtraTex`的alpha和`_Color`的alpha取小的那个，作为最后的alpha值。___因为normal map的特殊性，执行完合图shader后，还会执行一次post process的shader:___ `NormalSwizzleShader`

```glsl
// This works in conjunction with the AtlasNormalShader that
// swizzles the alpha to red channel of the packed normal and blends them.
// Then this post effect re-swizzles the normal into the
// packed format for use with standard shaders.
half4 frag(v2f_img i) : COLOR
{
    half4 r = tex2D(_MainTex, i.uv);

    //G and A are the important ones.
#if defined(UNITY_NO_DXT5nm)
    return half4(r.r, r.g, r.b, 0);
#else
    return half4(1, r.y, 1, r.x);
#endif
}
```

## 带cloth物理组件的slot的制作指南

+ 首先准备好一个skin好的fbx，这和普通的slot制作是一样的。
+ 把fbx整个放到一个unity场景中，在有`SkinMeshRender`的GameObject上添加`cloth`组件。
+ 设置好cloth组件的参数，刷好constrain权重，主要是衣服的上沿要刷成完全constrain的，就是max distance为0，这样衣服才不会掉下来。对于其他部分，根据需要刷对应的max distance，如果不刷，就是没有constrain的，可以完全自由移动。
+ 把设置好的整个GameObject变成一个Prefab。
+ 设置好slot builder，除了Slot Mesh不能设置留空（因为我们制作的Prefab不是mesh类型的）。
+ 然后把整个Prefab拖到Slot Builder的`Drag meshes hehe`框里面，这样就会自动用这个Prefab生成一个Slot，并且是带有cloth组件的。
+ 新建一个ClothProperties，设置好里面的参数，这些参数和cloth组件的参数基本是相同的，当cloth组件被创建后，应该是用这些参数去设置cloth组件的。
+ 新建一个RendererAssert，设置好Renderer Name（这个名字是会作为cloth组件所在的GameObject的名字的）。在`Cloth Properties`中设置刚才创建的ClothProperties。
+ 在创建好的Slot中，设置`Renderer Asset`为刚才创建的RendererAssert。
+ 后面的操作就和普通的slot是一样的了。运行时cloth组件会单独用一个Renderer来渲染的。
