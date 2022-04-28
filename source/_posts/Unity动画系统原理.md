---
title: Unity动画系统原理
date: 2021-09-03 15:42:03
tags: Unity
categories: GameEngine
---

## 通用结构

float4是一个float[4]的数组，主要对应simd的数据类型，存储的可以是vector4或者quaternion，可以代表平移，旋转或者缩放。
xform里面有 float4的t，q，s，就是simd版的transform信息。

## AvatarConstant

```c++
std::string AvatarBuilder::BuildAvatar(Avatar& avatar, const Unity::GameObject& go, bool doOptimizeTransformHierarchy, const HumanDescription& humanDescription, Options options)
```
会创建`AvatarConstant`，会用到下面这些结构体

<!-- more -->


```c++
enum Bones
{
    kHips = 0,
    kLeftUpperLeg,
    kRightUpperLeg,
    kLeftLowerLeg,
    kRightLowerLeg,
    kLeftFoot,
    kRightFoot,
    kSpine,
    kChest,
    kNeck,
    kHead,
    kLeftShoulder,
    kRightShoulder,
    kLeftUpperArm,
    kRightUpperArm,
    kLeftLowerArm,
    kRightLowerArm,
    kLeftHand,
    kRightHand,
    kLeftToes,
    kRightToes,
    kLeftEye,
    kRightEye,
    kJaw,
    kLastBone
};

struct HumanBone
{
	UnityStr			m_BoneName; //对应的fbx中的骨骼的名字
	UnityStr			m_HumanName; //对应的是human.cpp中定义的骨骼名。
	SkeletonBoneLimit   m_Limit;
};

struct SkeletonBone
{
	UnityStr	m_Name; //对应的fbx中的骨骼的名字
	Vector3f	m_Position; // 对应骨骼的本地平移
	Quaternionf m_Rotation; // 对应骨骼的本地旋转
	Vector3f	m_Scale; // 对应骨骼的本地缩放
	bool		m_TransformModified;
};

struct HumanDescription
{
	HumanBoneList m_Human; // HumanBone的数组，排列顺序不一定
	SkeletonBoneList m_Skeleton; // SkeletonBone的数组，按照fbx中导入的顺序排列。

	float	m_ArmTwist;
	float	m_ForeArmTwist;
	float	m_UpperLegTwist;
	float	m_LegTwist;

	float	m_ArmStretch;
	float	m_LegStretch;

	float	m_FeetSpacing;

	UnityStr	m_RootMotionBoneName;
};

struct Hand
{
    int32_t		m_HandBoneIndex[s_BoneCount]; // index分别是左右手的15个手指关节的enum，对应的值是Human::m_Skeleton的索引
};

struct Human
{
    // root motion的平移是骨骼按重量世界位置加权平均得来的（躯干处的骨骼权重高，四肢权重地，所以重心一般在hip这里）
    // root motion的方向：up是两个upLeg的平均位置指向两个upArm的平均位置得来的向量，right：Normalize((rightUpLeg - leftUpLeg) + (rightUpArm - leftUpArm)), front = right X up， right = up X front
    math::xform				m_RootX; // m_SkeletonPose所对应的root motion的t的y（人物重心的y）tr，s是1

    OffsetPtr<skeleton::Skeleton>		m_Skeleton; // 去掉了fbx中一些没有用到的叶子节点骨骼后的所有骨骼的skeleton。
    OffsetPtr<skeleton::SkeletonPose>	m_SkeletonPose; // 用户在编辑avatar时所调整的T pose的transform信息。
    OffsetPtr<hand::Hand>				m_LeftHand;
    OffsetPtr<hand::Hand>				m_RightHand;

    uint32_t							m_HandlesCount;		
    OffsetPtr<human::Handle>			m_Handles;

    uint32_t							m_ColliderCount;
    OffsetPtr<math::Collider>			m_ColliderArray;

    int32_t					m_HumanBoneIndex[kLastBone]; // index是mecanim::human::Bones这个枚举，对应的值是m_Skeleton的索引
    float					m_HumanBoneMass[kLastBone];
    int32_t					m_ColliderIndex[kLastBone];
    
    
    float					m_Scale; // m_SkeletonPose所对应的root motion的t的y（人物重心的y）

    float					m_ArmTwist;
    float					m_ForeArmTwist;
    float					m_UpperLegTwist;
    float					m_LegTwist;

    float					m_ArmStretch;
    float					m_LegStretch;

    float					m_FeetSpacing;

    bool					m_HasLeftHand;
    bool					m_HasRightHand;
};

struct AvatarConstant
{
    OffsetPtr<skeleton::Skeleton>			m_AvatarSkeleton; // 包含所有fbx中骨骼的skeleton
    OffsetPtr<skeleton::SkeletonPose>		m_AvatarSkeletonPose; // 用户在编辑avatar时所调整的T pose的transform信息。

    OffsetPtr<skeleton::SkeletonPose>		m_DefaultPose;	// 导入fbx时的transform信息，没有包括用户的编辑操作。

    uint32_t								m_SkeletonNameIDCount;  // m_AvatarSkeleton的数组大小
    OffsetPtr<uint32_t>						m_SkeletonNameIDArray;	// fbx中所有骨骼的名字的crc，只是名字，没有路径，顺序和m_AvatarSkeleton一致

    OffsetPtr<human::Human>					m_Human;

    uint32_t								m_HumanSkeletonIndexCount; // m_Human.m_Skeleton的数组大小
    OffsetPtr<int32_t>						m_HumanSkeletonIndexArray; // index就是m_Human.m_Skeleton的index，对应的值是m_AvatarSkeleton的index

    // needed to update human pose and additonal bones in optimize mode
    // decided to put the info in constant for perf and memory reason vs doing masking at runtime
    uint32_t								m_HumanSkeletonReverseIndexCount; // m_AvatarSkeleton的数组大小
    OffsetPtr<int32_t>						m_HumanSkeletonReverseIndexArray; // index是m_AvatarSkeleton的index，对应的值是m_Human.m_Skeleton的index，-1表示没有对应

    // 下面这些信息对应humanoid的动画是没有作用的，只有对于generic的动画有效
    int32_t									m_RootMotionBoneIndex;
    math::xform								m_RootMotionBoneX;
    OffsetPtr<skeleton::Skeleton>			m_RootMotionSkeleton;
    OffsetPtr<skeleton::SkeletonPose>		m_RootMotionSkeletonPose;
    uint32_t								m_RootMotionSkeletonIndexCount;
    OffsetPtr<int32_t>						m_RootMotionSkeletonIndexArray;
};
```

`mecanim::skeleton::Skeleton`是所有骨骼的父子关系信息，Axes目前不知道是什么，Axes中的Min、Max是绕着某个轴转的角度范围(角度制)

`mecanim::skeleton::SkeletonPose`是对应于某个Skeleton的某一个动作的快照，存的是所有骨骼的transform信息（t,r,s）

AnimationClip是对应fbx中定义的一个动画take。
void ModelImporter::ImportMuscleClip(GameObject& rootGameObject, AnimationClip** clips, size_t size)把fbx中的一个AnimationClip中的所有的动画曲线，转换成Muscle Space中的动画曲线。

std::string GenerateMecanimClipsCurves(AnimationClip** clips, size_t size, ModelImporter::ClipAnimations const& clipsInfo, mecanim::animation::AvatarConstant const& avatarConstant, bool isHuman, HumanDescription const& humanDescription, Unity::GameObject& rootGameObject, AvatarBuilder::NamedTransforms const& namedTransform)这个函数是执行曲线转换的核心

人物的动画数据都是存在`AnimationClip::m_FloatCurves`中，骨骼的动画信息只包含每个骨骼的旋转信息(x,y,z)，而且不是每个骨骼都包含xyz，有的只有两项或者一项。这些旋转信息都被拆分成独立的float曲线存放。除了骨骼动画信息外，还保存了RootMotion的信息（T.xyz, Q.xyzw）和双手双脚的动画信息（T.xyz, Q.xyzw）。最后才是用户在动画中定义的曲线。

这些曲线的顺序是这样的，这个就是attribute的名字
```c++
"RootT.x"
"RootT.y"
"RootT.z"
"RootQ.x"
"RootQ.y"
"RootQ.z"
"RootQ.w"

"LeftFootT.x"
"LeftFootT.y"
"LeftFootT.z"
"LeftFootQ.x"
"LeftFootQ.y"
"LeftFootQ.z"
"LeftFootQ.w"

"RightFootT.x"
"RightFootT.y"
"RightFootT.z"
"RightFootQ.x"
"RightFootQ.y"
"RightFootQ.z"
"RightFootQ.w"

"LeftHandT.x"
"LeftHandT.y"
"LeftHandT.z"
"LeftHandQ.x"
"LeftHandQ.y"
"LeftHandQ.z"
"LeftHandQ.w"

"RightHandT.x"
"RightHandT.y"
"RightHandT.z"
"RightHandQ.x"
"RightHandQ.y"
"RightHandQ.z"
"RightHandQ.w"

"Spine Front-Back",
"Spine Left-Right",
"Spine Twist Left-Right",
"Chest Front-Back",
"Chest Left-Right",
"Chest Twist Left-Right",

"Neck Nod Down-Up",
"Neck Tilt Left-Right",
"Neck Turn Left-Right",
"Head Nod Down-Up",
"Head Tilt Left-Right",
"Head Turn Left-Right",

"Left Eye Down-Up",
"Left Eye In-Out",
"Right Eye Down-Up",
"Right Eye In-Out",

"Jaw Close",
"Jaw Left-Right",

"Left Upper Leg Front-Back",
"Left Upper Leg In-Out",
"Left Upper Leg Twist In-Out",
"Left Lower Leg Stretch",
"Left Lower Leg Twist In-Out",
"Left Foot Up-Down",
"Left Foot Twist In-Out",
"Left Toes Up-Down",

"Right Upper Leg Front-Back",
"Right Upper Leg In-Out",
"Right Upper Leg Twist In-Out",
"Right Lower Leg Stretch",
"Right Lower Leg Twist In-Out",
"Right Foot Up-Down",
"Right Foot Twist In-Out",
"Right Toes Up-Down",

"Left Shoulder Down-Up",
"Left Shoulder Front-Back",
"Left Arm Down-Up",
"Left Arm Front-Back",
"Left Arm Twist In-Out",
"Left Forearm Stretch",
"Left Forearm Twist In-Out",
"Left Hand Down-Up",
"Left Hand In-Out",

"Right Shoulder Down-Up",
"Right Shoulder Front-Back",
"Right Arm Down-Up",
"Right Arm Front-Back",
"Right Arm Twist In-Out",
"Right Forearm Stretch",
"Right Forearm Twist In-Out",
"Right Hand Down-Up",
"Right Hand In-Out"

// 左手的5 * 4 = 20个自由度
// 右手的5 * 4 = 20个自由度

// 最后是用户自己定义的float曲线
```

这些muscle曲线都经过Skeleton中定义的Axes转换到muscle space的（RegartFrom），然后运行的时候再从msucle space转换回来（RetargetTo）。转换的算法目前没有看懂。

优化措施，这个曲线会被分成三类StreamedClip (hermite曲线)，DenseClip （线性的折线），ConstantClip （常数）。

## AnimationClip

```c++
struct GenericBinding
{
	BindingHash    path; // 0
	BindingHash    attribute; // attribute字符串的crc32值，对于kBindMuscle的，会替换成muscleIndex，即上面的attribute字符串表的索引。
	PPtr<Object>   script;  // 空的
	UInt16         classID; // Animator = 95
	UInt8          customType;  // 对于动画中定义的muscle曲线，是kBindMuscle = 8，对于我们定义的曲线是kUnbound=0
	UInt8          isPPtrCurve; // 0
};

struct AnimationClipBindingConstant
{
	dynamic_array<GenericBinding>    genericBindings; //顺序是，所有的m_FloatCurves先被分成三类（Streamed，Dense，Constant），然后按照三类的顺序依次append在后面
	dynamic_array<PPtr<Object> >     pptrCurveMapping; //是空的
};

// StreamedClip用到的结构
struct CurveTimeData
{	
	float  time; // 后面跟着的所有CurveKey的时间
	UInt32 count; // 后面总共有多少个CurveKey，他们的时间都是相同的
	// This is implicitly followed by CurveKey in the stream
};

// StreamedClip用到的结构
struct CurveKey
{
	int    curveIndex; // 这个CurveKey是属于哪个curveIndex的
	float  coeff[4];  // 这个CurveKey的四个系数
};

// hermite曲线
struct StreamedClip
{
	uint32_t            dataSize;
    // 所有曲线的所有key按照时间从小到大排序，如果时间相同就按照curveIndex从小到打排序。
    // 把时间相同的key的时间统一用一个CurveTimeData记录，后面跟着这些CurveKey，所以data的结构如下
    // CurveTimeData|CurveKey|...|CurveKey| ... |CurveTimeData|CurveKey|...|CurveKey| ... |CurveTimeData
    // 最后有一个time是infinity的CurveTimeData
	OffsetPtr<uint32_t> data; 
	UInt32              curveCount; // 曲线数量
};

// 线性的折线
struct DenseClip
{
    int                     m_FrameCount; // 根据动画时间长度和m_SampleRate计算出来
    uint32_t                m_CurveCount; // 曲线数量
    float                   m_SampleRate; // 一般是60
    float                   m_BeginTime;  // 一般是0
    
    uint32_t                m_SampleArraySize; //  = m_FrameCount * m_CurveCount
    OffsetPtr<float>		m_SampleArray;  //每条曲线有m_FrameCount 个 float
};

// 常数
struct ConstantClip
{
	UInt32              curveCount; // 曲线数量
	OffsetPtr<float>    data;   // 每条曲线的值
};

struct Clip
{
    StreamedClip					m_StreamedClip; // curveIndex
    DenseClip                       m_DenseClip; // curveIndex - m_StreamedClip.curveCount
    ConstantClip                    m_ConstantClip;  // curveIndex - m_DenseClip.m_CurveCount - m_StreamedClip.curveCount
    
    OffsetPtr<ValueArrayConstant>	m_DeprecatedBinding; // 空的
};

// 人物四肢的transform信息
struct HumanGoal
{
    math::xform m_X;
    float m_WeightT;
    float m_WeightR;
};

// 人物手和手指的transform信息
struct HandPose
{
    math::xform m_GrabX;
    float m_DoFArray[s_DoFCount];
    float m_Override;
    float m_CloseOpen;
    float m_InOut;
    float m_Grab;	
};

// 从clip中采样得来的人物姿势
struct HumanPose
{
    math::xform		m_RootX;
    math::float4	m_LookAtPosition;
    math::float4	m_LookAtWeight;

    HumanGoal		m_GoalArray[kLastGoal];
    hand::HandPose	m_LeftHandPose;
    hand::HandPose	m_RightHandPose;	

    float			m_DoFArray[kLastDoF];
};

// 动画在开始和结束时候的sample结果
struct ValueDelta
{
    float m_Start; // 对应m_StartTime时的值
    float m_Stop; // 对应m_StopTime时的值
}; 

struct ClipMuscleConstant
{
    human::HumanPose m_DeltaPose; // 动画结束时的人物姿势减去动画开始时的人物姿势得到的人物姿势的差	
    
    math::xform	m_StartX; // 动画开始时人物重心root的transform
    math::xform m_LeftFootStartX; // 动画开始时人物左脚的transform
    math::xform m_RightFootStartX; // 动画开始时人物右脚的transform
    
    math::xform	m_MotionStartX; // 在start时的motion（muscle index：0~6）的transfrom
    math::xform	m_MotionStopX; // 在end时的motion（muscle index：0~6）的transfrom
    
    math::float4 m_AverageSpeed; // 整个动画期间，人物重心root的平均速度，分成20份来计算的。

    OffsetPtr<Clip>	m_Clip; // 对应一个Clip
    
    float	m_StartTime; // 动画的起始时间，都是0
    float	m_StopTime;  // 动画的结束时间，等于动画的长度，单位秒
    float	m_OrientationOffsetY; // 对应Root Transform Rotation的Offset设置
    float	m_Level; // 对应Root Transform Position(Y)的Offset设置
    float	m_CycleOffset; // 对应Cycle Offset设置
    float	m_AverageAngularSpeed;  // 整个动画期间，人物重心root的绕着y轴的平均角速度，分成20份来计算的。

    int32_t				m_IndexArray[s_ClipMuscleCurveCount]; // 用muscleIndex作为索引，然后值是genericBindings数组的索引curveIndex

    uint32_t				m_ValueArrayCount; // 所有曲线的数量和：m_StreamedClip.curveCount + m_DenseClip.m_CurveCount + m_ConstantClip.curveCount
    OffsetPtr<ValueDelta> m_ValueArrayDelta; // 记录了动画在m_StartTime和m_StopTime时的所有曲线的值，数组索引就是curveIndex

    bool	m_Mirror; // 对应Mirror设置
    bool	m_LoopTime; // 对应Loop Time设置
    bool	m_LoopBlend; // 对应Loop Pose设置
    bool	m_LoopBlendOrientation; // 对应Root Transform Rotation的Bake Into Pose设置
    bool	m_LoopBlendPositionY; // 对应Root Transform Position(Y)的Bake Into Pose设置
    bool	m_LoopBlendPositionXZ; // 对应Root Transform Position(XZ)的Bake Into Pose设置

    bool	m_KeepOriginalOrientation; // 对应Root Transform Rotation的Based Upon设置: Original => true, Body Orientation => false
    bool	m_KeepOriginalPositionY; // 对应Root Transform Position(Y)的Based Upon (at Start)设置: Original => true, Center of Mass/Feet => false
    bool	m_KeepOriginalPositionXZ; // 对应Root Transform Position(XZ)的Based Upon (at Start)设置:Original => true, Center of Mass => false
    bool	m_HeightFromFeet; // 对应Root Transform Position(Y)的Based Upon (at Start)设置: Feet => true, 其它的是false
};

class AnimationClip {
    AnimationStateList m_AnimationStates;

	float           m_SampleRate;  // 一般是60
	bool			m_Compressed;  // 一般是false
	bool            m_UseHighQualityCurve; // 一般是false
	int             m_WrapMode; ///< enum { Default = 0, Once = 1, Loop = 2, PingPong = 4, ClampForever = 8 }， 一般是0
	
	QuaternionCurves   m_RotationCurves; // 都是空的
	Vector3Curves      m_PositionCurves; // 都是空的
	Vector3Curves      m_ScaleCurves; // 都是空的
	FloatCurves        m_FloatCurves; // 都是空的
	PPtrCurves         m_PPtrCurves; // 都是空的
	Events             m_Events; // 都是空的
	AnimationType      m_AnimationType; // kHumanoid - 3

    mecanim::animation::ClipMuscleConstant*		m_MuscleClip; // 动画自带的曲线和我们定义的曲线都在m_MuscleClip中
	mecanim::uint32_t							m_MuscleClipSize;
	UnityEngine::Animation::AnimationClipBindingConstant m_ClipBindingConstant;

    AABB m_Bounds; // 一般都是0
};
```

## AnimatorController

