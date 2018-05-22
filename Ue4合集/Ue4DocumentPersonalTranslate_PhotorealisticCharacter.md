[Original Page](https://docs.unrealengine.com/en-us/Resources/Showcases/PhotorealisticCharacter)
相片级角色

这个展示（文章？）的目的是演示如何使用高品质的角色渲染技术，这个所展示的技术与Paragon(EPIC出的一个MOBA游戏)中的角色所用的技术十分相似。要观看这个展示，只用打开工程，在编辑器下点开始，然后就能像电影一样环绕地观看了。

想了解更多地了解渲染角色的技术，请到[Unreal Engine Livestream - Tech & Techniques Behind Creating the Characters for Paragon.](https://www.youtube.com/watch?v=toLJh5nnJA8) (大概视频比较详细？)

#皮肤着色

下面这个角色地皮肤使用UE4的[Subsurface Profile Shading Mode](https://docs.unrealengine.com/en-US/Engine/Rendering/Materials/LightingModels/SubSurfaceProfile)。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Skin.png)
需要注意的是材质函数是皮肤材质的基石。这些函数被设计成可被重复使用的方式，用以构建Paragon的不同的材质，这使得美术能够标准化地去创建一种特定的表面，但是要注意的是，一个基本的材质函数的改变会影响所有引用这个材质函数的材质的改变。

###皮肤着色纹理
（在样例中）主角的所使用的纹理均是4k纹理，并且全都是从演员上扫描出来的，然后被EPIC的美术清理和拆分。整个皮肤由五个贴图组成：

|Texture| Name | Description|
|-------| ---- | -----------|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SkinDiffuse.png)| 漫反射(Diffuse) | 漫反射纹理定义皮肤的基本颜色。在4K下，你甚至可以看到皮肤下面的毛细血管。由褶皱产生的暗化将会使法线贴图更加地突出。（应该是皮肤表面上由褶皱产生地黑斑将由法相贴图来提供）(UE4_Demo_Head_D)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SkinRough.png) | 粗糙(Roughness) | 粗糙纹理储存在散射纹理的Alpha通道上，这是一种很常见的减少纹理量的技术。需要注意的是粗糙度将会是增加内部的毛孔和褶皱。这将使那些区域变得粗糙，由散射纹理和法向纹理产生的孔和褶皱看起来更深。并且，头发位置上的纹理被设置成最大值（1.0），这将阻止头发上的游离高光，这将给头发带来更深刻的感觉。(`UE4_Demo_Head_D`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SkinSpec.png)| 镜面反射(Specular)|镜面反射纹理用来表示有多少镜面高光从皮肤表面上反射出去。要注意一点就是默认的镜面反射值为0.5。当皮肤拉伸地有点紧的地方，需要提高镜面反射值，这将使得皮肤看起来会有一片区域的镜面高光，同时，如果在不需要镜面反射的地方，例如说褶皱和毛孔的中心，需要调低镜面反射值。(`CH2_exp04_merged_spec_f_FC_FINAL`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SkinScatter.png)|散射(Scatter)|散射纹理控制有多少光从皮肤内部散射出来。黑色的区域将会展现出很少的散射效果，例如脸颊。而白色的区域将会有更多的散射，例如说鼻子和耳朵。散射光的颜色由[Subsurface Profile asset](https://docs.unrealengine.com/en-US/Engine/Rendering/Materials/LightingModels/SubSurfaceProfile)控制。(`UE4_Demo_Head_BackScatter`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SkinNormal.png)|法向|法向纹理的使用与默认的材质中的使用方式一直，用来控制像素点上的向量偏移。在本例中，没有任何与其默认作用以外的作用(`UE4_Demo_Head_normals`)|

#头发渲染

头发渲染需要使用UE4的Hair Shader Model。这个渲染方式是一种物理的渲染方式，该种渲染方式基于[Eugene d'Eon, Steve Marschner and Johannes Hanika](http://www.eugenedeon.com/project/importance-sampling-for-physically-based-hair-fiber-models/)的研究，目前也被用于[Weta Digital](https://www.wetafx.co.nz/research-and-tech?url=/research)的渲染当中。着色器近似地模拟从各向异性的镜面高光投射到头发上，进行折射和多发丝间的散射。
要使用头发着色器，将你的材质的Shading Model设置成Hair。

###头发和高光

在现实世界中，头发一般趋向于拥有多种镜面高光，一种表示光照的颜色，另一种则表示头发颜色和光照颜色的混合。为了方便起见，本文将之称之为主要镜面高光和次要镜面高光。头发着色器将会用高品质地效果去近似地模拟这种效果。

![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/TwinblastHair_Specs_Diagram.png)
>1, 主要镜面高光。2，次要镜面高光。

在UE4的头发着色器中使用的近似算法，将用与现实相同的方式去创建这些效果。当一根光线打在一根发束上的时候，它不只是简单地弹开而已。头发是透明的，会允许一些光线穿过它，然后再内部反射，然后再离开头发。头发着色器将使用三条光线在发束可能的移动路线去模拟这种效果，如果下面的GIF所展示的一样。

![我不知道GIF能不能播放来着。。](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/HairDiagram.gif)
>单根头发地剖面图展示了在头发着色器中的三种光线的光线路劲模拟。
>|Number|Description|
>|------|-----------|
>|0|发束的生长方向|
>|1|只有反射的路径，光线从头发表面弹开，形成主要反射光照。|
>|2|折射-折射路径，光线从这个路径从头发射入并从头发的另一面射出，这是光线在众多头发内部的散射路径|
>|3|折射-反射-折射路径，光线从头发表面折射进入头发内部，并在头发内部反射最后折射出头发外部，这主要形成了次要高光|

就像上面图表所展示的一样，一束头发并不是一个完美的圆柱或者管子。事实上，头发的性质表现地更像叠起来的锥体。这意味着，光线离开头发表面的时候，相比于离开完美光滑的表面，会更加的发散。并且，头发地朝向不同，所以镜面高光将不会是统一地，而是取决于头发的朝向而单独的分布。这有点像是各向异性高光，当然这也被UE4的头发着色器所支持的。

###头发和透明

头发着色器使用 Mask 的混合模式而不是透明模式。使用 Mask 模式将使得头发只有两种结果，要么完全不透明，要么完全透明。有一个持续的随机抖动不断在整个表面移动，以去掉锯齿（狗牙）。抖动被用来处理Mask的混合，但是只有在 TemporalAA 启用的时候才有效。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Hair_AAOn.png)
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Hair_AAOff.png)
> 在TemporalAA下使用随机抖动需要几帧处理混合。这将使得头发在移动会有一些非自然的表现。这是TemporalAA这项技术预期的表现。

###边缘蒙版
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/EdgeMaskGraph.png)
虽然这并不是着色器的一部分（指并不是shadermodel的一部分，也就是你自己需要添加），但是需要注意的是，Paragon使用了Edge Mask来对侧对摄像机时的渐隐。当头发使用多种面片来模拟的时候，如下文 Hair And Geometry 所展示的那样， 当面片法线与摄像机方向快垂直时，能很明显地看到面片地边缘，影响效果。

为了减缓这个问题，材质将会计算摄像机的法向与顶点法线。当两个法线互相垂直的时候，头发将会逐渐消失直到完全透明。但是，在另一方面，这样会使得头发着色器渲染出更多的头皮（大概是面片之间的缝隙被加大了，因为垂直的面片被干掉了）。这也是为什么许多头发很厚的角色通常在头皮上也带有头发纹理，正如下面的图片所展示的一样（这里说的是在建模的做脸的皮肤纹理的时候通常会把头发的纹理也做上，这样在面片之间的缝隙里显露出来的就不会是皮肤的颜色而是头发的颜色）：
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/HairPaint.png)
>通常这个功能都会使用一个开关，当运行在底性能平台上时，可以被关掉以节省性能（因为需要使用PixelDepthOffset，这个会影响early-z对像素的预先剔除）。

###头发制作
在使用展示中所使用的技术创建的头发时，需要对Epic构建角色头发的方法有一定的了解。

####头发几何(Hari Geometry 好吧我真不知道这个怎么翻译)
UE4的头发渲染器所使用的的头发几何形通常用很多的非平的面片，通常这种方式常被使用与很多实时头发模拟中。这个通常能被你所使用的DCC APP所生成（大概是建模软件？）。
>通常来讲头发着色器对头发的几何图形并没有什么严格规定，但是需要注意的是，这个角色使用了大约800个面片，总共大约18000个三角形。并且还使用了双面材质。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/HairSheets.png)

####头发纹理
在这个使用中的头发着色器中，最终结果由以下5中主要的纹理：漫反射(Diffuse), 透明度(Alpha),根(Root),深度(Depth)，和一个 ID(Unique-per-strand ID)纹理。在Epic中，这些纹理常常使用3Ds Max's的头发系统，将模拟出来的结果投射到一张纹理中。但是，这里也有很多其他的选项去调整模拟的结果。

|Texture|Name|Description|
|-------|----|-----------|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Diffuse.png)|漫反射(Diffuse)|漫反射纹理主要提供了头发本身的漫反射颜色或者说基色。它有时候并不会着色而是通过参数去控制其最后的颜色。特别是那种有杀马特造型的角色（就是头发花花绿绿的）。(`UE4_Demo_Hair_D`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Alpha.png)|透明度(Alpha)|透明度贴图控制头发的不透明区域，隐藏不需要的部分.(`UE4_Demo_Hair_A`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Root.png)|根(Root)|根贴图用来控制从发根到发梢的颜色变化。(`UE4_Demo_Hair_Roots`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/Depth.png)|深度(Depth)|深度纹理用来控制像素的深度偏移，以形成将头发推地更远，以至于陷进头发体的错觉。这也可以用来被当成一个基准，在不同深度下改变颜色或者其他着色属性的值，例如在头发下跌进头皮时减少一大片的镜面反射等(想象不出来。。。)。(`UE4_Demo_Hair_Depth`)|
|![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/ID.png)|ID(Unique ID)|唯一ID贴图为了每一个头发集合体内的每一根头发都提供了一个唯一的ID的值。这通常用来提供一些细微的变化(?)(`UE4_Demo_Hair_ID`)|

####头发着色器属性

当你使用头发着色器的时候会在属性的输出接口发现一些新的属性：散射(Scatter)，切线(Tangent)和背光式(Backlit)。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/HairProperties.png)
>在写这篇文章的时候（4.13），背光式(Nacklit)属性的存在只是兼容一些旧式的shader，链接它并没有什么卵用，可以忽略它。

#####散射(Scatter)
我们通常将头发着色器认为只是一个近似模拟，其原因是其并不会模拟光线对每一根单独的头发的交互。在现实世界中，当光线弹出头发时，通常会碰到另一根光线，并且重复数次相同的过程。以现在计算机的性能来说，没办法实时模拟。
然而，无论是在游戏还是在现实世界中，光线在头发中的散射行为任然是对头发的颜色至关重要的。为了控制这这哦那个效果，头发着色器提供了散射属性，替换了金属度成为了第一个属性，并且被限定在了0~1之间。散射控制多少光线穿过了整个头发，如果它是一个单面片一样。
一个需要注意的重点是，在亮色的头发上，散射更强，而在深色的头发上，散射更弱。这是依照了现实世界的的规律——深色的颜色趋向于吸附更多的颜色。实际上，如果你正尝试创建一个金发的角色，你会发现，单改变漫反射贴图和颜色还不够，同时你也需要改变散射值。
在这个例子中，发根颜色和发梢颜色被设置成灰金色，并且随机变形设置成0.0。散射被使用于控制多少光线穿过头发。这举例说明了单单改变散射系数就能产生很多种的头发。

#####切线(Tangent)
切线属性替换了头发着色器的法向向量属性。切向属性是每一根头发的走向的向量，并且指向根部。切向属性的目的在于正确地实现各向异性反射。各项异性反射是指光在带有细小沟槽地物体上弹开时所照成地效果，例如说拉丝金属。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/AnisoVsNonAniso.png)
> 左边的球就是各项异性渲染的，而右边的就不是。注意看各项异性镜面高光是怎么从表面上延申的。

切线属性主要是用一个法向向量控制头发的切线，控制各项异性镜面高光的延申。
![](In this image the yellow line represents the tangent along a strand of hair, pointing back toward the root.)
> 这上面的图片种，黄色的线表示表示一根头发的切线，从发梢指向发根的方向。

> 在这个角色的头发渲染器种，这个向量在Z轴上用UniqueID给定了一个从0.3到-0.3的随机值。这产生了一个拥有随机弓形方向的向量，并且能够给各向异性镜面高光提供一些变化，就像是真实的一撮头发一样。

切线贴图通常由两个来源：自动生成或者通过流动贴图(Flow Map)生成。自动生成只需要保证你的头发根在上并且头发尖在下。如果你的头发很短，并且没有面片被环绕或者扭转，那么最后的结果都是可以接受的。在示例中的角色就使用了这种效果。
另一种方式是使用流动贴图。这种方法需要生成一张流动贴图。如果你的头发比较长，并且是弯曲而且卷地比实际上的几何图形要更弯曲，或者如果每一个贴图的不同部分的头发的朝向不一致（不像上面说的从上到下）。流动贴图代表了头发在切线空间或者在平面下的移动方向。在Photoreal Character Bust这个工程下，你会发现一张没有使用过的叫`T_Hair_Flow`的流动贴图。在下面有一个对比图展示了流动贴图对镜面高光的影响。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SparowFlowMap.png)![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/SparowFlowMap2.png)
> 在这里你能看到流动贴图是怎么沿着表面头发去绘制的。要注意的是，流动贴图只需要用到一些头发面片，而不是所有的头发。注意流动贴图上不同的值是如何巧妙地沿着头发移动高光。

#####在头发着色器中使用像素深度偏移
像素深度偏移(PDO)并不是头发着色器中特有的属性。用外行人的话来说，PDO使得像素看起来远离摄像机，造成一种表面凹陷的人工效果。当头发是由多个简单面片组成的时候，就像下面图片所展示的那样，PDO的使用能提供一种头发整个陷入到头发整体上的真实效果。同时这也会打碎面片和模型的接触点，如下图所示。
![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/PDO_off.png)![](https://docs.unrealengine.com/portals/0/images/Resources/Showcases/PhotorealisticCharacter/PDO_On.png)

###眼睛渲染
鸽了
