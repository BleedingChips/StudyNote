Maya插件：
TressFX/Tool/Maya/TressFX_Exporter.py 复制到一个专用的文件夹（比如说plugs-in），然后搜索一个文件Maya.env（通常再C:\用户\<用户名>\文档\Maya\2016），然后用记事本打开，输入PYTHONPATH = <你存放上诉python文件的地址>，保存关闭。打开maya。

Maya Xgen 
找不到过程XgCreateDescription
>
打开 Maya201x\plug-ins\xgen\scripts\xgenm\xgGlobal.py
再最前面添加三行代码：
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
保存

import TressFX_Exporter
reload(TressFX_Exporter)
TressFX_Exporter.UI()

导出的模型的骨骼需要4个以上。。。
这个是个BUG

编译 https://github.com/lion03/UE4-K4W-Plugin/tree/TressFX-4.17

尝试在vs2017用vs2015的工具链编译，有很多的文件reinclude了。
尝试使在vs2017用vs2015的工具链编译。
遇见错误：combaseapi.h(229): error C2760: syntax error: unexpected token 'identifier', expected 'type specifier'
解决办法：https://github.com/EpicGames/UnrealEngine/commit/4f48ef53ed646a22532e8e981f5515c94f303932
简单来说就是在 Engine/Source/Programs/Windows/BootstrapPackagedGame/Private/BootstrapPackagedGame.h
和
Engine/Source/Runtime/Core/Public/Windows/MinWindows.h
两个文件种的`#include <windows>`前面加上`struct IUnknown;`

UE4.17

FTressFXShadeUniformBuffer': undeclared identifier
TressFXTypes.h 与 TressFXVertexFactory.h 循环依赖。
把 TressFXTypes.h 的 #include "TressFXVertexFactory.h" 注释掉。

fbxproperty.h(1215): error C2903: 'GetPropertyValue': symbol is neither a class template nor a function template

git config --global http.postBuffer 524288000

UE4.16 代码分析：

头文件包含关系：
`PrimitiveComponent.h`
- `TressFXComponent.h`

类型：
`FTressFXSimulationSettings`
`FTressFXHairGenerationSettings`
FTressFXShadeSettings
UTressFXComponent

新建 Config/DefaultEditorPerProjectUserSettings.ini
写上
[/Script/SourceCodeAccess.SourceCodeAccessSettings]
PreferredAccessor=VisualStudio2017
自动生成vs2017文件

UTressFXComponent 继承于 UPrimitiveComponent

UTressFXComponent 创建 FTressFXSceneProxy
FTressFXSceneProxy 继承于 FPrimitiveSceneProxy


Engine\Source\Editor\UnrealEd\Private\Factories\TressFXFactory.cpp
TressFXFactory里边创建资源，创建TressFXAsset

TressFx文件信息猜测：

FTressFXTFXFileHeader
{
    	float version;                      // Specifies TressFX version number
	uint32 numHairStrands;        // Number of hair strands in this file. All strands in this file are guide strands.
										// Follow hair strands are generated procedurally.
	uint32 numVerticesPerStrand;  // From 4 to 64 inclusive (POW2 only). This should be a fixed value within tfx value. 
										// The total vertices from the tfx file is numHairStrands * numVerticesPerStrand.

										// Offsets to array data starts here. Offset values are in bytes, aligned on 8 bytes boundaries,
										// and relative to beginning of the .tfx file
	uint32 offsetVertexPosition;  // Array size: FLOAT4[numHairStrands]
	uint32 offsetStrandUV;         // Array size: FLOAT2[numHairStrands], if 0 no texture coordinates
	uint32 offsetVertexUV;         // Array size: FLOAT2[numHairStrands * numVerticesPerStrand], if 0, no per vertex texture coordinates
	uint32 offsetStrandThickness;  // Array size: float[numHairStrands]
	uint32 offsetVertexColor;      // Array size: FLOAT4[numHairStrands * numVerticesPerStrand], if 0, no vertex colors

	uint32 reserved[32];           // Reserved for future versions
}

struct FTressFXTFXBoneFileHeader
{
	float version;
	unsigned int numHairStrands;
	unsigned int numInfluenceBones;
	unsigned int offsetBoneNames;
	unsigned int offsetSkinningData;
	unsigned int reserved[32];
};

目前的主要问题是 UTressFXComponent::ShouldCreateRenderState() 这里过不去。
这里需要：
Asset != nullptr && Asset->ImportData.IsValid() && HairTexure != nullptr && Asset->IsValid() && Asset->ImportData->SkinningData.Num();

被 UActorComponent::ExecuteRegisterEvents() 调用，估计是Default Object？

Asset->ImportData->SkinningData 为 0

不是很了解这里面的玩意。

Asset本身需要被设置属性。。。
然后跑是能跑了。。。

坑了。