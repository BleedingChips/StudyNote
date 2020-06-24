* 编译的时候，如果出现：

    ```
    2>  Creating makefile for ShaderCompileWorker (no existing makefile)
    2>  Performing full C++ include scan (no include cache file)
    2>  Distributing 77 actions to XGE
    2>EXEC : Fatal error : Failed to open/create session data
    2>  Failed to open/create session data: STATUS_ACCESS_DENIED
    ```

    或者：
    ```
    1>EXEC : Fatal error : Failed to start build.
    1>Failed to connect to Build Service (on local machine): Server is not reachable
    ```

    类似的错误的话，可以是因为联机编译的问题。

    解决方式是将`C:\Users\%USERNAME%\AppData\Roaming\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml`: 中设置联机服务器  bAllowXGE 为false 以关闭联机服务器
    例：
    ```xml
    <?xml version="1.0" encoding="utf-8" ?>
        <Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
            <BuildConfiguration>
                <bAllowXGE>false</bAllowXGE>
            </BuildConfiguration>
        </Configuration>
    ```

---

* 一旦虚幻4项目的帧率低于30，UE4将会禁止粒子碰撞以保证帧数保持在30以上。ISO项目默认锁30，需要在Project Settings > GeneralSettings > Framerate中修改 

---

* UE4打包问题

    ```
    ERROR : System Exception：Couldn't update resource
    ```

    解决办法：把 Save 和 Intermediate 文件夹下的文件删除，然后重新打包即可。

---

* HZB Steup Mips在GPU Profile里边占用了大量时间。
    DX11的问题，将在DX12中修复。

---

* 文档找不到，改语言。
    ZHO， CHN（中文），INT（英文）

---

* stat physics查看模拟数据。

---

* UProperty(Meta = (ExposeOnSpawn = true)) 带参数构造函数

---

* 用于启动vs2017
    在DefaultEditorPerProjectUserSettings.ini 文件中加入一行
    ```
    [/Script/SourceCodeAccess.SourceCodeAccessSettings]
    PreferredAccessor=VisualStudio2017
    ```

---

* 当使用FXAA的时候，因为屏幕空间镜面反射需要获取前一帧的信息，而FXAA没有存，所有由存在变黑的情况。

---

* 在植被中，模型的CULL DISTANCE与模型所用的LOD是有关系的。裁剪距离会根据最低LOD是的screensize所决定的距离和设置的culldiatnce的距离求最大值，用来当成裁剪距离。也就是说如果一个模型的最低LOD级的screensize为0.01或者0，则表示在植被中该模型可能永远不会被裁剪。

---
* FreezeRendering
    冻结渲染项

---

* TAA
[Oringial PPT](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)

---
* 相片级角色渲染

    [Original Page](https://docs.unrealengine.com/en-us/Resources/Showcases/PhotorealisticCharacter)
    [中文地址泪崩](https://docs-origin.unrealengine.com/latest/CHN/Resources/Showcases/PhotorealisticCharacter/index.html)

---
* stat
    * `DECLARE_STATS_GROUP(TEXT("A"), B, STATCAT_Advanced);`
        声明一个`B`类型的STAT，通过`stat A`命令打开统计窗口。
    * `DECLARE_CYCLE_STAT(TEXT("D"), C, B);`
        定义一个`B`类型的STAT的实体，其实体名字为C，且在stat窗口的描述中为`D`。
    * `SCOPE_CYCLE_COUNTER(C);`
        在代码中统计。

---
* 支持的dds文件（测试时间是4.18）。
    * 只支持 `A8R8G8B8` 和 `RGBA16F` 两种像素格式。
    * 只支持`2D`和`CubeMap`两种纹理。

---
* r.TemporalAASamples
    控制TAA的锐度。

---

* 在命令行里边输入help可以打开一个网页以查找所有的命令。

---

[自定义shadingmodel](http://blog.felixkate.net/2016/05/22/adding-a-custom-shading-model-1/)

* 目前来说据了解，UE4只支持单通道8位的图片格式。所以如果用一些16位的数据转成的贴图存在丢失的问题。

* 对于想用纹理来储存数据的时候，主要纹理会进行mipmap，然后会导致纹理进行压缩了，可能会导致数据错误。需要在纹理的设置 Level Of Detail 中将 Mip Gen Steeings 设置成 NoMipmaps 以取消掉mipmap。

* 载入纹理的时候，默认进行伽马修正，会对结果进行处理，使得RGB的值变暗。如果需要RGBA同时储存数据，需要关闭伽马矫正，具体方式为在纹理的属性中把sRGB属性点掉。


* custon 自定义函数

    若想要在custon里边定义函数，一种是使用#define来进行模拟，不过模拟会有问题，具体问题未知。
    
    或者使用另一种方法。

    * 现在材质里边定义一个新的定义用的custom节点node1，输入代码：
        ```
            return float4(0.0, 0.0, 0.0 ,0.0); //目标输出类型
        }
            //其他一些定义的变量啥的
        float4 define_function_here(/*parameter*/)
        {
            return float4(/*...*/);
        ```

    * 再定义一个节点node2，输入代码：
        ```
        return define_function_here(/* ... */);
        ```
    
    * 最后将两个节点的输出用add链接起来，最好将node1的输出放置于node2之上。

    
*   Texture Object 节点

    Texture Object 在作为 custon node 的输入的时候会顺带传入一个SamplerSate。

    比如说一个 Custon node 的输入节点是 Tex ,那么能获取到的是纹理 Tex 和 TexSampler ，后者则是 SamplerState。

*   RenderTarget

    创建一个RenderTarget，如果需要绘制一个Rendertarget，需要调出Begin DrawCanvas To Render Target节点，从获取的到Canvas里边设置要绘制的信息，然后调用End DrawCanvasToRenderTarget关闭绘制。

