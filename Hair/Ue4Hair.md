在UE4 4.24下的头发资源生成和模拟（以Maya XGen为例）
1. 头发资源准备
    Ue4中需要的头发曲线分为两个种，一个是用来产生物理效果的【导线】，另一些则是根据【导线】的位置进行插值的【发线】。对于传统的直发，也可以直接提供【发线】，在导入的时候引擎会自动生成【导线】，但对于一些特殊的头发造型，如卷发，则需要同时提供【导线】与【发线】。
    关于xGen生成头发的过程不再累述，具体可以参考网上的教程，或者参考[官方文档](https://docs.unrealengine.com/zh-CN/Engine/HairRendering/XgenGuidelines/index.html)这里默认已经生成好头发丝的两种曲线，即【导线】与【发线】，如下图：

    ![图片](image/UE4Hair/img_1.png)

    这这里讲起分成了两种不同的组别，方便对不同的头发进行。

1. 属性赋予

    在UE4中，所需要的[属性](https://docs.unrealengine.com/zh-CN/Engine/HairRendering/AlembicForGrooms/index.html)中，比较重要的有如下几个：

    |名字|意义|类型|范围|
    |---|---|---|---|
    |groom_ guide|该头发是否是导线|int8/16/32|常量/统一 0=非导线/1=导线|
    |groom_group_id|头发分组ID|int32|常量/统一 0~INT_MAX|

    除此之外，在Maya中导出Alembic格式文件时，需要添加几个[属性](https://graphics.pixar.com/usd/docs/Maya-USD-Plugins.html)以控制导出，比较重要的如下几个：

    |名字|意义|类型|范围|
    |---|---|---|---|
    |riCurves|强制将曲线导出成一组|bool|1|
    |{attr}_AbcGeomScope|控制该熟悉在曲线顶点上的分布|string|见下表|

    |分布|意义|
    |---|---|
    |'con'|常量 所有的曲线都有同一个值|
    |'vtx'|顶点 每个曲线的每个顶点都有一个值|
    |'uni'|统一 每条曲线，都有一个值|

    在Maya中，可以选中对应的曲线组别，然后通过如下Python代码设置属性：

    ```python
    GuideAttr = 'groom_guide'
    GroupId = 'groom_group_id'
    Selected = cmds.ls(sl = True)
    for Ite in Selected:
        cmds.addAttr(Ite, longName=GuideAttr, attributeType='short', defaultValue=0, keyable=True)
        cmds.addAttr(Ite, longName='{}_AbcGeomScope'.format(GuideAttr), dataType='string', keyable=True)
        cmds.addAttr(Ite, longName=GroupId, attributeType='short', defaultValue=0, keyable=True)
        cmds.addAttr(Ite, longName='{}_AbcGeomScope'.format(GroupId), dataType='string', keyable=True)
        cmds.addAttr(Ite, longName='riCurves', attributeType='bool', defaultValue=1, keyable=True)
    ```

    添加完毕后，可以手动在附加属性中自由编辑。
1. 资源导出
    选中所有要导出的曲线：
    ![图片](image/UE4Hair/img_2.png)
    导出配置如下：
    ![图片](image/UE4Hair/img_3.png)
    ![图片](image/UE4Hair/img_4.png)
    即可。
1. UE4设置

    转至 编辑（Edit） > 项目设置（Project Setting），以打开项目设置。在以下各个部分，启用这些设置：

    * 渲染（Rendering）> 优化（Optimizations）> 启用 支持计算皮肤缓存（Support Compute Skincache）
    * 动画（Animation）> 性能（Performance）> 禁用 Tick Animation on Skeletal Mesh Init（@@@） 
    
    转至 编辑（Edit） > 插件（Plugins），打开插件浏览器窗口。在 几何体（Geometry） 类别下，启用这些插件：
    * Alembic Groom Importer
    * Groom 

    重启编辑器。

1. 导入头发资源

    直接在编辑器资源窗口导入对应的.abc文件。

    ![图片](image/UE4Hair/img_5.png)

    从Maya导出的曲线的翻转如下，BuildSettinig选项控制自动生成的导线，如果自带了导线可以不用管。

    导入之后，有两种安装方式。
    * 将头发资源拖动到场景中，调整位置使得头发与目标Actor位置重合。然后将世界大纲中的头发移动到目标Actor上去使其成为目标Actor的子Actor，这时候会跳出来一个选择框，选择合适的节点。
    * 在目标Actor下添加一个Groom的Component，填充该Component的Groom资源，并且把BindGroomToSkeletalMesh点上。

    在安装完毕后，在Groom下，添加NiagaraParticleComponent，在NiagaraSystemAsset中选择GroomAssetSystem。

1. 调整头发参数。

    见[头发参数](https://docs.unrealengine.com/zh-CN/Engine/HairRendering/Reference/index.html)

1. 头发材质。

    头发材质与旧流程中的头发材质一致，唯一多了的一点就是材质里的表达式HairAttributes，该表达式可以返回在XGen中生成的头发数据，以帮助渲染。