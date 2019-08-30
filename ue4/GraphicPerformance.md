*  Disable Smooth FrameRate
    Project Setting -> General Setting -> Frame Rate -> Smooth Frame Rate -> Smooth Frame Rate -> OFF


*  stat FPS
    统计FPS。
*  stat unitgraph/ unit
    统计各线程的时间。
    Frame : 每一帧的总时间。
    Game : CPU用的时间。
    Draw : CPU向GPU提交绘制申请的时间。
    GPU ： GPU实际用的绘制时间。
* stat SceneRendering
    统计Draw call，三角形数量，光照数量，半透明时间。

* stat GPU
    统计各个阶段GPU用的时间。

* GPU Cisualizer
    GPU的各个阶段的统计。
    默认快捷键是(CRT SHIFT ,)

* STAT StartFile
    开始统计
* STAT StopFile
    结束统计。
    文件将保存再 Project / Save / profiling 子文件夹下。

    然后通过 Session Frontend 查看统计信息（CPU，GPU）

* 使用 r.SetRes 480X270 等小一点的分辨率，来计算运行时间，看看是不是由于分辨率太高的问题导致的性能问题。

Pixel - bound Source of trouble

* 跟像素相关的操作时间最慢。
* 大Back Buffer。
* 多光源（动态），重着色器。
* PP

* 半透明材质。
* 粒子用LOD

* Quad overdraw
    GPU通常用4*4或者8 * 8的方式来同时绘制一个或多个像素，所以小的图元也很耗性能。

* 通常三角形的数量并不是瓶颈，防止太小太细的三角形更重要。
    LOD.
    但也会影响到阴影计算。

* Texture的带宽，更少的贴图，压缩的贴图格式。UV的布局更连续（防止GPU上的cache missing）

* TextureStreaming的问题，同一个Level下尽量让材质与Mesh的数量一样。

* Defeered Rendering 和 Forward Renderingh （降低光照的消耗）

* 植物应该多剪裁
LOD

* Optimization ViewModes
    * Light Complexity 灯光的交互。
    * Shader Complexity 像素的重叠度，还有材质的复杂度。（材质的复杂度可能并不太准）
    * Quad Overdraw 表示三角形的大小度。太大的话表示三角形太小了，需要做LOD。
    * Lightmap Density 表示光照贴图的密度。精良密度小一点。（可以调整）Window下Statistics可以看到光照贴图的大小。
    * Texture 上可以调整Texture的大小。

* Pipeline
    * GPUParticles_PreRender 更新GPU粒子的属性。
    * PrePass DDM_AllOpaque PreZ，更新所有的模型深度。
    * ComputeLightGrid 光源的剔除，有点模型的可见性剔除。与光源的数量相关。
    * BeginOcclusionTests 可见性检测。
    * BuildHZB 创建多Mipmap等级的ZBuffer。
    * ShadowDepths 生成全局的深度。
    * CompositionBeforeBasePass 
    * BeginRenderingGBuffer
    * BasePass
    * BeginRenderingCustomDepth
    * CustomDepth
    * RenderVelocities
    * ResolveSceneDepthTexture
    * LightCompositionTasks_PreLighting
        * 延迟贴花和AO AO的消耗可以在Postpost 里边的Fade Out Distance 和 Fade Out Rate 调。
    * ClearTranslucentVolumeLighting
    * DirectLighting
    * FilterTranslucentVolume
    * SkyLightDiffuse
    * ScreenSpaceReflections
    * TAA ScreenSpaceReflectionPS
    * ReflectionEnvironmentAndSky
    * ResolveSceneColor
    * CompositionAfterLighting
        * 处理次表面散射材质。
    * ExponentiaHeightFog
    * BeginRenderingSceneColor
    * GPUParticles_PostRenderOpaque
    * Translucency
    * Translucency
    * Distortion
    * RenderLightShaftBloom
    * ResolveSceneColor
    * PostProcessing

Quad Overdraw

    GPU渲染机制的问题，就是GPU绘制一个像素时是同时将周围几个像素一起绘制，并选取其中有用的。如果有用的太少，就是造成浪费。解决方案是对于某些细长的三角形可以通过添加分割。或者一些半透明的多加顶点进行分割。

Particle Trimming
    
    通过增加顶点数来减少像素填充率

* 动态光与静态光
    动态光多的要保证Actor的数量少。
    静态光不能保证阴影效果，但是能够放更多的Actor。
* 模型重用
    地图加载更快