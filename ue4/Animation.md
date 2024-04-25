* 蓝图节点资源读取类型：
    1. Sequence Player 直接播放一个动作序列
    2. Blend Space Player 混合空间播放，依靠两个值来决定多个动作之间（但一般情况下最多只有四个）的混合系数。会将一些混合数据离线存入SkolenMesh。
    3. Montage & Slot 不支持混合空间，支持网络同步。

* 动画蓝图更新流程：
    1. 编译期： AnimGraphNode -> AnimNode 将编译期的节点转换成运行时的节点。
    1. 运行期：
        1. UpdateAnimation : 计算权限，更新蒙太奇，更新虚函数（项目逻辑），复制属性给蓝图。
            SkeletalMeshComponent : Tick -> TickPose -> TickAnimation
        1. ParallelUpdateAnimation : 根据AnimNode的节点，各节点调用Update_AnyThread;调用TickAssetPlayerInstances,跟新动画资源播放进度。（节点更新）
            TickAnimation
            Mesh组件 : Tick -> RefreshBoneTransforms -> DispatchParallelEvaluationTasks -> PerformAnimationProcessin
        1. ParallelEvaluateAnimation : 再次编译AnimNode图，各节点调用Evaluate_AnyThread;资源节点读取谷歌位置，混合节点进行融合。(完成骨骼运算)
            单线程： TickAnimation
            多线程： Mesh组件 : Tick -> RefreshBoneTransforms -> DispatchParallelEvaluationTasks -> PerformAnimationProcessin
            Intertialization 节点优化的主要节点：
                按惯性的方式插值替换融合（与普通的Blend会有差别，只支持某些特殊的动作，支持小时间）
            优化：混合Alpha值为0时可直接跳过。
            客户端情况下这部占用性能较高。
            与 UpdateAnimation 频率可能不一样。
            用不到的分支可剪掉。
        1. DispatchQueuedAnimEvents : 触发动画序列和蒙太奇的事件通知。

* Paragon 蓝图示例上的更新流程：
    1. Ground Locomotion 一般根据运动学（站立，跳跃，下蹲，跑步）等计算出当前的Pose。
    1. AimOffset 调整头部动画，控制头部目标朝向。
    1. UpperBodyLayer 上半身动画分层，持枪持刀等。
    1. 通过IK控制脚步。

    参考链接 [Nucl.ai 2016] Bringing a Hero From Paragon to Life with Ue4

    适合Demo期间，适合角色比较少的时候使用。

* Ue4 动画蓝图的复用方式：

    1. Child AnimBP 继承动画蓝图的方式。
        动画逻辑相同，资源不同，具体逻辑不同。
        子类只能重写EventGraph，不能修改AnimGraph。
        消耗较大

    1. Linked AnimGraph (Sub Anim Instance) 额外的动画蓝图
        可以实现单独的逻辑，被任意动画蓝图引用
        要求骨架
        可以通过修改底层支持任意骨架，最新版本已经去除限制。

    1. Linked AnimLayer
        单个蓝图内部实现多个Layer，运行时可动态切换。
        不能被其他浏览图引用

    1. PostProcess AnimBP 最后处理的动画蓝图。
        一般做一些IK或者物理等后处理工作。
        逻辑顺序固定在最后的流程才生效。
        用来处理换装等功能

* 运行时管线，不能强引用动画资产，通过配置来配置资产。

* 早期可参考 Advanced Locomotion System （ALS）

* Locmotion 混合问题；

    一般容易碰上上半身和下半身的混合问题，比如下半身执行向右跑动的动作，上半身执行向左运动的动作，强行混合容易让人感觉腰部要断裂。

    结局方案：
        Paragon 的 Melee Twist 可以做到近战带动下半身。
            Paragon 的向前移动是个混合空间，可以在向某个方向移动时，让身体保持不同的朝向。

        ALS4 本身支持姿态转换。
            将向前向后两个动作分成六个状态，既向左朝前，向前朝前，向右朝前，向左朝后，向后朝后，向右朝后，中间通过状态机进行关联。

    Locomotion 穿腿的问题
        IK造成的。去掉可能会导致滑步。
        状态内部的混合

* 动画资源管线

    1. 动画运行时管线设计：

        1. 按流程：移动，过场，玩法，程序化，动态效果。
        1. 按状态，地面移动，控制移动
        1. 按部位，全身，部分，手脚等

        大量使用混合。

    1. 面部动画（研究LookAt，口型和表情）基于 Pose 和 Curve，一般面部动画细节比较多，很难用逻辑去控制。

    1. 服装动态效果。
        
        动画打底，叠加Kawaii和布料。
        不能通过全程序的方式进行，很难对每个动作都进行调整。
        Kawaii 伪物理。物理太拟真。

    1. 约束。
        将裙摆的骨骼约束到腿部骨骼，叠加部分的物理。百分百避免穿插问题。
        RBF，基于Pose的插值方法，多层级骨骼会导致Pose数量爆炸。

* 物理动画

    计算好最终Pose之后，为其添加约束，然后进行物理计算，得到最终结果。

    1. 死亡表现
        基于单帧的Pose的混合空间资产，根据冲量计算混合Pose，根据冲量大小计算位置上的偏移。
        遍历刚体，对CurrBody执行以下操作：
            根据骨骼位置，计算出TargetPose上对应刚体的TargetTransfrom
            使用TargetTransform创建KinematicActor，并在其和CurrBody之间创建物理约束
            当满足一定条件时断开约束，让布娃娃飞出去
    1. 受击表现
        参考守望先锋的受击表现。
        惯性插值

        1. OverBlend（过度混合）的处理，当混合系数超过0.0和1.0的时候，直接混合会出很奇怪的问题。
        1. 弹性节点处理（纯弹簧处理）
        1. IK处理保持受的姿势以及脚步的吹。

* 换装（模块化外观）

    1. 参考堡垒之夜。
        通过CopyPose将计算好的动作复制到外观Mesh上。

    1. 最大化骨架设计。
        将所有有可能出现的部件的骨架集合到一个主骨架中，然后直接计算主骨架的动画，最后将计算好的骨架通过CopyPose复制到模块骨架中。
        需要经常更新骨架。

* 捏脸

    基于Pose和Curve。

    1. DDC绑定并到处控制器极值Pose。
    1. 引擎内将捏脸控制前值转成曲线并驱动Pose。

* 体型

    修改 BindPose
    重定向 (Scale也要)，开销比较低

* UE4 重定向 （Translation Retargeting Mode） 原理

    分成下面几个规则：

    1. Animation 使用动画数据，没有发生重定向。
    1. Skeleton 使用Target的骨骼位移（丢弃动画位移），使用动画的旋转缩放，不能做动画位移。
    1. Animation Scaled 将动画的骨骼位移缩放，比例为TargetLength/SourceLength，使用动画的旋转缩放。
    1. Animation Relative 计算两个骨架的差（Additive），并应用到动画上。Translation 是直接加上去而不是 Scale，Location 是直接加上去。很少用。
    1. Orient And Scale 如果没有位移，则为 Skeleton，如果有则在Animatio Scale的基础上，将Translation绕着ParentBoneSpace原点旋转。插值为两个RefPose下的旋转差（并不改变Rotation）。

* 性能优化

    1. URO （Update Rate Optimization）
        通过动态更改更新频率，UE内置。
        会影响变形和镜头动画。

    1. AnimationBudget
        通过设定固定的更新时间，动态调整各个角色的更新频率。
        根据优先级，计算资源有限分配给重要Mesh。
        需要表现降级时处理。
    1. ACL动画压缩库

    1. 动画蓝图属性拷贝优化（4.26之后才有）
        通过将节点链接的属性换成绑定到蓝图右边的属性。
    1. Mesh的CalcBounds优化。
        对于一些长链型的刚体，中间的一些刚体的计算可以删掉。
    1. 角色复活时动画热点的优化
        对于一些子图比较多的动画蓝图需要添加一个初始化进行复用。
    1. ControlRog节点优化
        避免虚拟机创建开销。
    1. GetAnimationPose函数的优化
        默认情况下将骨骼和曲线全部计算，在部分情况下只需要骨骼不需要曲线。
    1. 角色动画图和子图的数据共享
    1. VisibilityBasedAnimTickOption 的设置。
    1. TickAnimationOnSkeletalMeshInit  
        创建在主线程同步更新动画来彼岸TPose
        方便获取骨骼和Socket位置。
        打开后会使得更新跑到主线程，关闭后能够提升性能。
    1. BoundingBox更新
        UseAttachParentBound 打开，避免组件的Bound的更新。一般离本体比较近的组件可以用。
        Use Bounds from Master Pose Component 打开
    1. 多用Master Pose（LeaderPose），复用BoneTransformBuffer
    1. 注意动画蓝图的FastPath.
    1. 动画蓝图节点LOD.

* 服务器动画优化
    一般用作命中校验，但实际上能不用动画解决的，务必不要服务器动画。

    1. 单帧总耗时有上限
    1. 动画和客户端发来的移动包相关，一个移动包更新一次。
    1. 要降每次更新的小号
    1. 动画有损优化对动画效果的影响难以评估。

    优化方向：

    1. 子图跳跃更新
    1. 减骨骼
    1. 只更新，不评估动画。只有在需要用的哦动画（校验）的时候才去评估（刷动画）。有动作同步的问题，需要一套机制来保证客户端和服务端动作一致。
    1. 合并移动包，减少动画更新次数。

* 比较新的动画技术

    1. ControlRig
    1. IKRig
    1. MotionMatching

    1. 基于深度学习的Motion Matching/Motion Synthesis
    1. 基于深度学习的 Character Control

* 参考资料

    https://www.gdcvault.com/play/1012300/Animation-and-Player_Control-in