* 生成材质的中间shader文件：
    1. 在游戏命令行中，依次调用下列命令：
        ```
        r.ShaderDevelopmentMode 1
        r.DumpShaderDebugInfo 1
        r.DumpShaderDebugShortNames 1
        ```
        或者在ConsoleVariables.ini中输入下列命令：
        ```
        r.ShaderDevelopmentMode = 1
        r.DumpShaderDebugInfo = 1
        r.DumpShaderDebugShortNames = 1
        ```

    1. 改动你所需要查看shader文件的材质，`con + shift + .` 重新编译和保存。

    1. `Saved\ShaderDebugInfo\PCD3D_SM5\`下即使生成的所有Shader文件与生成命令。

---

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