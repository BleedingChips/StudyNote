HairWork需要核弹厂(NVIDIA)的特化版UE4，要获取该版本，有下面两种方式：

* 直接从管事的人哪里拉一个拷贝。
* 直接获取源码编译。
    1. 在[Github](https://github.com/)上申请一个账号。
    1. 在[虚幻官网](https://www.unrealengine.com/)中创建账户，并在账户设置中链接的你Github账号。链接后会自动加入EPICGame的组织，可以获取其源码以编译。
    1. 在[NVIDIA的GameWork](https://developer.nvidia.com/gameworks)中创建账号，并加入GameWorksAccessTeam，并链接你的Github账号。（貌似需要审核啥的。。。）
    1. 从[NVIDIA的Github](https://github.com/NvPhysX/UnrealEngine)上Fork到你的本地仓库。这个仓库上有很多分支，但我们需要的只有HairWork。然后需要将这个仓库clone到你的本地（download只能下载mater分支，里面只有个说明啥的，没用。然后Github Destop上下载经常挂，所以还是推荐使用命令行，代码：`github clone https://github.com/aaabbb/UnrealEngine.git J:\github -b HairWorks --depth 1`其中 `aaabbb`表示你的账户，`J:\github`表示你的地址，然后 `-b HairWorks`表示只要这个分支 `--depth 1` 表示只要最近的提交记录）。
    1. 编译即可
        1. 点Setup跑进度。
        1. 点`GenerateProjectFile.bat`生成vs2015的工程文件，或者调用`GenerateProjectFile -2017`生成vs2017的工程文件。
        1. 然后打开工程，编译即可。

准备头发资源。

1. 从[GameWork下载中心](https://developer.nvidia.com/gameworksdownload)中在左侧的搜索栏中搜索MAYA，选择 HairWorks Plug-in for Maya and 3DS Max，下载，选择对应版本的插件安装。（在Maya中，需要选 窗口/选项/插件管理器 （`windows->Settings/Preference->Plug-inManager`）开启HairWork这个插件，在max中则会自动打开）。
1. 通过Maya或从Max中创建头发曲线数据并导出（见附录）。
1. （可选）将模型和头发资源导入HairWorkViewer，调整外观和样式。
    1. 在[下载中心](https://developer.nvidia.com/gameworksdownload)搜索Hair，然后下载`NVIDIA HairWorks`。
    1. 导入数据，调整参数（见附录）。
1. 打开UE4，导入模型数据与头发资源，并调整其外观。

渲染。

1. 导入.FBX和.APX文件，调整头发样式。
1. 新建Actor，挂载.FBX为Component。
1. 挂载.APX为.FBX的子Component。 

附录：
---
* Max中创建头发曲线数据并导出：
    1. 创建或导入一个模型，加入并蒙皮至少一个骨骼。
    1. 点选要生成头发的模型，然后通过`Modifiers -> Hair and Fur`为该模型创建初始的头发。
    ![图片](image/HairWork/HairWork1.png)
    ![图片](image/HairWork/HairWork2.png)
    ![图片](image/HairWork/HairWork3.png)
    1. 选用Selection模式，选择需要生成头发的区域，点击`Update Selection`重新生成。
    ![图片](image/HairWork/HairWork4.png)
    ![图片](image/HairWork/HairWork5.png)
    ![图片](image/HairWork/HairWork6.png)
    1. 点击Styling下的`Style Hair`按钮，造型，造型完毕后点击`Finish Styling`。
    ![图片](image/HairWork/HairWork7.png)
    ![图片](image/HairWork/HairWork8.png)
    1. 由头发创建线条，创建HairWork并指定线条。
    ![图片](image/HairWork/HairWork9.png)
    ![图片](image/HairWork/HairWork10.png)
    ![图片](image/HairWork/HairWork11.png)
    ![图片](image/HairWork/HairWork12.png)
    ![图片](image/HairWork/HairWork13.png)
    ![图片](image/HairWork/HairWork14.png)
    1. 添加碰撞骨骼并绑定碰撞体。
    ![图片](image/HairWork/HairWork15.png)
    ![图片](image/HairWork/HairWork16.png)
        * 关于碰撞，HairWork只支持圆形和园柱碰撞，碰撞体只有圆形一种，只可以静态绑定在骨骼节点上，然后通过链接碰撞体产生园柱。另外，能绑定的碰撞体也是有限制的，HairWork只能检测与生成头发模型所绑定的骨骼。也就是说，如果想要设置的碰撞点不在生成模型的相关骨骼节点下，只能添加一个权重为0的蒙皮节点。
    1. 插入并设置插销（可选）。
    ![图片](image/HairWork/HairWork17.png)
    ![图片](image/HairWork/HairWork16.png)
    1. 导出检查。
    ![图片](image/HairWork/HairWork18.png)
        * 如果会发现有`X out of Y fur stands share mesh vertices with other lines`的错误报告，会自动选中重复的曲线，删除后重新导出即可。
    1. 导出.FBX和.APX格式即可。
* Maya中创建头发曲线数据并导出：
    * 需要外部插件生成头发。

另：
[HairWorks的文档](http://docs.nvidia.com/gameworks/content/artisttools/hairworks/index.html)
[HairWorks View的文档](http://docs.nvidia.com/gameworks/content/artisttools/hairworks/HairWorks_viewerReference.html)

附：[HairWork特定毛发设置参考（非官方）](http://gad.qq.com/article/detail/23698%20target=)

另，各[HairWork属性](http://docs.nvidia.com/gameworks/content/artisttools/hairworks/HairWorks_viewerReference_hairTab.html#hairworks-viewerreference-hairtab)中翻：

（有些选项后面有一些曲线或者黑白格表示可以用曲线或者贴图来控制这个数值）

* ImportSetting 导入设定
    * Groom 控制在导入Hair文件的时候，是否同时导入groomed guide curves。
    * Materials 决定是否同时导入材质属性。
    * Constraints 决定是否同时导入pin constraints（销约束，大概是在做发型的时候用的）。
    * Textures 纹理。
    * Collision 碰撞。

* General Options 全局配置。
    * Enable 是否启用这个Hair。关掉了之后就啥都没有了。
    * Spline Multiplier 控制细分数，越高的值看起来越平滑，但是效率越低。

* Visualization 显示，这个貌似只在Editor下才有显示。
    * Hair 是否渲染这个头发。
    * Guide Curves 导向曲线。
    * Skinned Guide Curves 在蒙版坐标系下的导向曲线。
    * Control Points 控制点。
    * Growth Mesh 用来生成头发的模型。
    * Bones 头发的骨骼。
    * Bone Names 骨骼的名字。
    * Bounding Box 包围盒。
    * Collision Capsules 碰撞觉囊体。
    * Hair Interaction 控制头发相互作用的线。粉红色表示在原始的距离，绿色表示太长了。
    * Pin Constraints 约束点。
    * Shading Normal 法线。
    * Shading Normal Center 发现中心。
    * Control Texture 如果生成模型被显示的话，就在模型上显示控制控制纹理。
    * Colorize Options 用颜色来显示各参数

* Physical 物理系数
    * General 基本设置
        * Simulate 是否开启模拟。
        * Mass Scale 设置头发的模拟质量，单位为一个标准质量，与场景设置的质量单位无关。
        * Damping 阻尼系数。。。不懂的百度一下。
        * Inertia Scale 惯性系数。就是当角色移动很快，或者瞬移的时候，会使得头发突然拉长，可以通过设置这个系数减少拉长的效果。为0时为无惯性。
        * Inertia Limit 惯性阈值，大于这个速度地才开启惯性。

    * Stiffness 刚度设置（下面所有选项都可以用贴图或者曲线表示）
        * Global 控制每个头发偏移蒙皮骨骼的位置。
        * Strength 弹性。1时为无弹性。
        * Damping  弹性的阻尼。
        * Root 发根的僵直度。
        * Tip 发梢的僵直度。
        * Bend 控制头发的物理效果对生成模型的影响程度。

    * Collision 碰撞。
        * Backstop 控制模型的表面用作碰撞。0表示完全不用，1表示用全模型的平均边线来决定一个近似值。这个会使得头发人为的变得更疏松。
        * Friction 摩檫力。。。。这个自己测试下吧。
        * Capsule Collision 允许碰撞形状的碰撞处理。
        * Hair Interaction 控制头发是单独处理还是集体处理碰撞。
    
    * Hair Pins 头发插销，类似于固定点一样的玩意。
        * Hair Pin List 控制所有的插销点。
        * HairPin Display Mesh 在插销点上显示的模型。
        * Dynamic Hair Pin 控制插销点是否能通过头发的物理特性等影响其位移。
        * Tether Pin 当动态插销点开启时，这个将通过一个约束限制头发插销和其绑定的骨骼的距离。
        * Stiffness 控制插销离开范围的难易程度。
        * Influence Fall Off 控制当插销从开始到边界的衰减程度。
    
* Style
    * Volume 体积设置
        * Density 浓度，控制导电之间毛发的生成密度。0表示不生成头发，1.0表示每一个三角形生成64根头发。同理2.0表示128根。如果用了贴图来设置这个值，则最终的值为这个值与贴图采样的值的乘积。默认情况下，其值通过定点采样确定。
        * Use Pixel density 如果用一张贴图来控制密度，那么从原来的按顶点采样将被改成按像素采样。这个将会导致明显的性能下降。
        * Length Scale 控制头发的长度。如果是0则表示0倍的长度，1表示1倍的长度。如果用贴图，则用这个值乘以贴图按顶点采样的值的结果作为最终的值。
        * Length Noise 产生非均匀的头发。
    
    * Strand Width 头发宽度
        * Width 基本宽度，单位是毫米，这个标签页下所有的选项都要乘以这个值，包括用贴图控制的部分。
        * Root Scale 发根的宽度。
        * Tip Scale 发梢的宽度。
        * Noise 噪声，生成不同宽度的头发。

    * Clumping 群聚因子
        * Scale 多大的力度让每一根头发凝结成团。
        * Roundness 控制团的形状，0时为凹的团，2时为凸的团，1时为中性团。
        * Noise 噪声。

    * Waviness 产生波浪的因子
        * Scale 控制波浪的振幅，单位是厘米，不受基本单位的影响。
        * Scale Noise 对每个头发产生的噪声。
        * Scale Strand 控制单个头发的波浪强度。
        * Scale Clump 控制一团头发的波浪强度。
        * Frequency 波浪产生的平吕。
        * Frequency Noise 频率噪声。
        * Root Straighten 控制波浪到根部的衰减。0表示没有影响。1表示线性衰减。

* Graphics
    * Color
        * Root Color 发根的颜色
            * Root Color Map 贴图，相乘。
        * Tip Color 发梢的颜色
            * Tip Color Map 贴图，相乘。
        * Root/Tip Color Weight 控制发根和发梢颜色的平衡。
        * Root/Tip Color Falloff 颜色衰减。
        * Root Alpha Falloff 根部混合系数。
    
    * Strand
        * Per Strand Texture 大概是头发用的纹理。
        * Strand Blend Mode 混合模式。
        * Strand Blend Scale 混合系数。

    * Diffuse
        * Diffuse Blend 控制发根与生成网格模型的散射光混合。
        * Hair Normal Center 为Hair Normal Weight指定一个骨骼。
        * Hair Normal Weight 从上面指定的骨骼中继承其阴影强度，并用来减弱混合。通常用在长发中。
    
    * Specular
        * Color 控制主要和次要镜面反射的颜色。
        * Primary Scale 主要镜面反射的强度。
        * Primary Shininess 主要镜面反射的衰减程度。
        * Primary Breakup 基于噪声的主要镜面反射的宽度。
        * Secondary Scale 次要。
        * Secondary Shininess 次要。
        * Secondary Offset 从主要镜面反射中补偿次要反射，从而生成两块分离的高光区域。

    * Glint
        * Strength 头发生成的闪光的数量。
        * Size 设置噪声对闪光的影响程度。
        * Power Exponent 散光的反差强度。

    * Shadow
        * Shadow Attenuation 阴影对头发的影响强度。
        * Shadow Density Scale 阴影密度。在绘制阴影贴图时头发使用的密度，设置少一点以提高性能。
        * Cast Shadows 控制是否产生阴影。
        * Receive Shadows 控制是否接受阴影。

* Level of Detail
    * Culling 控制裁剪的设置。
        * View Frustrum Culling 当头发原理视口后被裁剪。
        * Backface Culling 当面指向摄像机时被裁剪。
        * Backface Culling Threshold 上面的阈值。

    * Continuous Distance LoD 根据距离控制头发的浓度和密度。并且从一定距离外全部裁剪掉。
        * Enable 是否使用。
        * Start Distance 从这个距离开始使用LOD。
        * End Distance LOD结束距离，这个距离之后，将不会再渲染。
        * Fade Start Distance 这个距离开始之后将使头发变得透明。
        * Base Width Scale 当到达End LOD Distance之后，宽度将会被线性插值到这个宽度。通常这个将会比物理页面上设置的宽度制更大，以保证在更小的密度下看起来不镂空。
        * Density Scale 线性插值的目标密度。

    * Continuous Detail LoD 让发束随着距离改变宽度和浓度的玩意。
        * Enable 是否开启。
        * Start Distance 当距离离这更近时，调用这个LOD。
        * End Distance 当距离接近到这个距离时，改变终止。
        * Base Width Scale 同上。
        * Density Scale 同上。
        
* Control Texture Channels 设置控制贴图使用的通道（能够多个属性共用一张贴图）