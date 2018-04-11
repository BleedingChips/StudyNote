这个项目主要记载着在使用/学习UE4过程中的一些注意事项和坑。
---

[自定义shadingmodel](http://blog.felixkate.net/2016/05/22/adding-a-custom-shading-model-1/)


<h2 id = "JUMP_POINT_MENU">目录</h2>

* [资源](#JUMP_POINT_RESOURCE)
* [渲染](#JUMP_POINT_RENDER) 
* [流程分析](#JUMP_POINT_PROCEDURE_ANALYZE)

<h3 id = "JUMP_POINT_RESOURCE">资源</h3>

* 目前来说据了解，UE4只支持单通道8位的图片格式。所以如果用一些16位的数据转成的贴图存在丢失的问题。

* 对于想用纹理来储存数据的时候，主要纹理会进行mipmap，然后会导致纹理进行压缩了，可能会导致数据错误。需要在纹理的设置 Level Of Detail 中将 Mip Gen Steeings 设置成 NoMipmaps 以取消掉mipmap。

* 载入纹理的时候，默认进行伽马修正，会对结果进行处理，使得RGB的值变暗。如果需要RGBA同时储存数据，需要关闭伽马矫正，具体方式为在纹理的属性中把sRGB属性点掉。

* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_RENDER">渲染</h3>

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

* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_PROCEDURE_ANALYZE">流程分析</h3>

*   [UStaticMeshComponent](https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/Components/UStaticMeshComponent/index.html) 的绘制

    ```
    <Engine\Source\Runtime\Engine\Classes\Components\StaticMeshComponent.h>
    <Engine\Source\Runtime\Engine\Private\Components\StaticMeshComponent.cpp>

    UActorComponent 
        |-> USceneComponent
	        |-> UPrimitiveComponent INavRelevantInterface
	            |-> 				<-| UPrimitiveComponent
	            |->	UMeshComponent
	                |-> UStaticMeshComponent
    ```

    在 UActorComponent 注册的时候，ActorComponent 会调用：
    ```
    virtual void ActorComponent::CreateRenderState_Concurrent()
    |-> virtual void UPrimitiveComponent::CreateRenderState_Concurrent()
    ```

    *   在这个函数里面将会调用
        ```
        bool UPrimitiveComponent::ShouldComponentAddToScene() const
        ```
        查看是否需要加入到场景里（通常是需要被隐藏啥的）。默认其需要加入到场景（不然怎么绘制啦！
        
        然后调用
        ```
        GetWorld()->Scene->AddPrimitive(this);
        |-> Scene (FSceneInterface*)
        |-> virtual void FSceneInterface::AddPrimitive(UPrimitiveComponent* Primitive)
        |-> void FScene::AddPrimitive(UPrimitiveComponent* Primitive)
        ```
        
        *   该函数内部调用
            ```
            FPrimitiveSceneProxy* UPrimitiveComponent::CreateSceneProxy()
            |-> FPrimitiveSceneProxy* UStaticMeshComponent::CreateSceneProxy()
            ```

            函数内部创建一个  [FStaticMeshSceneProxy](https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/FStaticMeshSceneProxy/index.html) 实例，并返回该实例的指针。

            ```
            <Engine\Source\Runtime\Engine\Public\StaticMeshResources.h>
            <Engine\Source\Runtime\Engine\Private\StaticMeshRender.cpp>

            FPrimitiveSceneProxy
            |-> FStaticMeshSceneProxy
            ```

            *   在 FStaticMeshSceneProxy 的构造函数中，会提取 StaticMesh 的每一个 LOD 信息，创建一个FLODInfo 实例。并且在 FLODInfo 的构造函数中，从 `TArray<FStaticMeshComponentLODInfo> UStaticMeshComponent::LODData` 根据 LOD 等级 获取 FStaticMeshLODResources ，里面包含了顶点信息。从 FStaticMeshLODResources 获取 FStaticMeshSection ，然后根据其材质索引，从
                ```
                UMaterialInterface* UStaticMeshComponent::GetMaterial(int32 MaterialIndex) const
                ```
                中提取材质接口和索引，构建 FLODInfo::FSectionInfo ，并存放到 `TArray<FSectionInfo> FLODInfo::Sections`
        
        该 FStaticMeshSceneProxy 指针将会放到成员变量中 `FPrimitiveSceneProxy* UPrimitiveComponent::SceneProxy` ，根据返回的 FStaticMeshSceneProxy 指针，将会创建一个 [FPrimitiveSceneInfo](https://docs.unrealengine.com/latest/INT/API/Runtime/Renderer/FPrimitiveSceneInfo/index.html) 的实例。
        ```
        [Engine\Source\Runtime\Renderer\Public\PrimitiveSceneInfo.h]
        [Engine\Source\Runtime\Renderer\Private\PrimitiveSceneInfo.cpp]

        FDeferredCleanupInterface
        |->	FPrimitiveSceneInfo
        ```
        其实例的的指针也会被放在成员变量中`FPrimitiveSceneInfo* FPrimitiveSceneProxy::PrimitiveSceneInfo`。
        同时将会用上面创建的 PrimitiveSceneProxy 构建一个 FCreateRenderThreadParameters 的栈对象，通过宏
        ```
        ENQUEUE_RENDER_COMMAND(CreateRenderThreadResourcesCommand)
        ```
        将一个函数发送至渲染线程。
        
        *   在这个命令中， FCreateRenderThreadParameters 对象将会调用其携带的 PrimitiveSceneProxy* 指针调用两个函数
            ```
            void FPrimitiveSceneProxy::SetTransform(const FMatrix& InLocalToWorld, const FBoxSphereBounds& InBounds, const FBoxSphereBounds& InLocalBounds, FVector InActorPosition)
            virtual void CreateRenderThreadResources()
            ```
            来设置一些渲染用的参数，另外 CreateRenderThreadResources 原本的函数为空，并且没有被派生，所以并没有做什么东西。
        
        最后还会通过宏
        ```
        ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)
        ```
        发送一个命令。
        
        * 在这个命令中，将会调用 
            ```
            void FScene::AddPrimitiveSceneInfo_RenderThread(FRHICommandListImmediate& RHICmdList, FPrimitiveSceneInfo* PrimitiveSceneInfo)
            ```
            *   这个函数将生成的 FPrimitiveSceneInfo* 加入到 `TArray<FPrimitiveSceneInfo*> FScene::Primitives` 。而 `int32 FPrimitiveSceneInfo::PackedIndex` 将会被当前所在的数组的位置索引所赋值。接下来将依次调用下列函数：
                ```
                FPrimitiveSceneInfo::LinkAttachmentGroup
                FPrimitiveSceneInfo::LinkLODParentComponent
                FPrimitiveSceneInfo::AddToScene
                ```

                AddToScene 函数内部将会调用函数：
                 ```
                FPrimitiveSceneInfo::AddStaticMeshes
                ```
                *   其会用 FPrimitiveSceneInfo* this 创建一个 FBatchingSPDI 栈对象。
                    ```
                    FStaticPrimitiveDrawInterface
                    |->	FBatchingSPDI
                    ```
                    然后通过其内部包含的 FPrimitiveSceneProxy* 调用：
                    ```
                    void FPrimitiveSceneProxy::::DrawStaticElements(FStaticPrimitiveDrawInterface* PDI)
                    |-> void FStaticMeshSceneProxy::DrawStaticElements(FStaticPrimitiveDrawInterface* PDI)
                    ```
                    *   该函数将会根据其包含的 LOD 信息，从`TIndirectArray<FLODInfo> FStaticMeshSceneProxy::LODs`中获取 FLODInfo ，进而获取 UMaterialInterface* ，并通过：
                        ```
                        FMaterialRenderProxy* UMaterial::GetRenderProxy(bool Selected,bool Hovered) const
                        ```
                        以 (false) 为参数获取 FMaterialRenderProxy* ，FMaterialRenderProxy* 调用
                        ```
                        virtual const class FMaterial* FMaterialRenderProxy::GetMaterial(ERHIFeatureLevel::Type InFeatureLevel) const = 0;
                        ```
                        获取 FMaterial* 。获取 FMaterial* 之后读取配置信息，生成配置。
                        
                        构造 FMeshBatch 栈对象，用
                        ```
                        bool FStaticMeshSceneProxy::GetMeshElement(...) const
                        ```
                        初始化该 FMeshBatch 对象。
                        
                        *   都读取 LOD 信息中的 UMaterialInterface* 和 FVertexFactory* ，并保存到 FMeshBatch 中。

                        调用
                        ```
                        void FStaticPrimitiveDrawInterface::DrawMesh(const FMeshBatch& Mesh, float ScreenSize)
                        |-> void FBatchingSPDI::DrawMesh(const FMeshBatch& Mesh, float ScreenSize)
                        ```
                        *   FBatchingSPDI 将自身包含的 FPrimitiveSceneInfo* 指针提取出 FStaticMesh 并添加到 `TIndirectArray<FStaticMesh> FPrimitiveSceneInfo::StaticMeshes`
                
                接下来将会轮询 `TIndirectArray<FStaticMesh> FPrimitiveSceneInfo::StaticMeshes ` 并放置到  `SparseArray<FStaticMesh*> FScene::StaticMeshes` 中。
                并调用下列函数将 FStaticMesh 加入到渲染队列里。（如果是透明材质的话，在此阶段将不会加入到任何队列。
                ```
                void FStaticMesh::AddToDrawLists(FRHICommandListImmediate& RHICmdList, FScene* Scene)
                ```
                接下来就是添加包围盒等一系列工作，因为是非渲染相关，不看了。

    注册完毕后，需要渲染，从下列的渲染主函数开始。
    ```
    uint32 FRenderingThread::Run(void)
        > void RenderingThreadMain( FEvent* TaskGraphBoundSyncEvent )
            > void FTaskGraphImplementation::ProcessThreadUntilRequestReturn(ENamedThreads::Type CurrentThread)
                > void RenderViewFamily_RenderThread(FRHICommandListImmediate& RHICmdList, FSceneRenderer* SceneRenderer)
                    > void FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
    ```
    这里开始就是渲染的主要流程了。。。嘛，略过上面的不看，直接定位到绘制透明物体处：
    ```
    void FDeferredShadingSceneRenderer::RenderTranslucency(FRHICommandListImmediate& RHICmdList)
    ```
    *   首先轮询成员变量 `TArray<FViewInfo> FSceneRenderer::Views` 提取出所有的 `FViewInfo` ，构建 `FDrawingPolicyRenderState` 栈对象，然后调用成员函数：
        ```
        void FDeferredShadingSceneRenderer::DrawAllTranslucencyPasses(
            FRHICommandListImmediate& RHICmdList, 
            const FViewInfo& View, 
            const FDrawingPolicyRenderState& DrawRenderState, 
            ETranslucencyPass::Type TranslucencyPass
            )
        ```
        *   这个函数将会根据 `FTranslucentPrimSet FViewInfo::TranslucentPrimSet` 调用
            ```
            void FTranslucentPrimSet::DrawPrimitives(
                FRHICommandListImmediate& RHICmdList, 
                const FViewInfo& View, 
                const FDrawingPolicyRenderState& DrawRenderState, 
                FDeferredShadingSceneRenderer& Renderer, 
                ETranslucencyPass::Type TranslucenyPassType
                ) const
            ```
            *   这个函数将从 `TArray<FTranslucentSortedPrim,SceneRenderingAllocator> FTranslucentPrimSet::SortedPrims` 里边根据索引提取 FPrimitiveSceneInfo ，然后调用：
                ```
                void FTranslucentPrimSet::RenderPrimitive(
                    FRHICommandList& RHICmdList, 
                    const FViewInfo& View, 
                    const FDrawingPolicyRenderState& DrawRenderState, 
                    FPrimitiveSceneInfo* PrimitiveSceneInfo, 
                    const FPrimitiveViewRelevance& ViewRelevance, 
                    const FProjectedShadowInfo* TranslucentSelfShadow, 
                    ETranslucencyPass::Type TranslucenyPassType
                    ) const
                ```
                *  这个函数里边会创建 FTranslucencyDrawingPolicyFactory::ContextType 栈对象，这个好像跟绘制动态对象有关，如果看的是静态对象的话就可以无视。从 FViewInfo 中分别获取动态网格对象和静态网格对象，并分别调用：
                    ```
                    bool FTranslucencyDrawingPolicyFactory::DrawDynamicMesh(
                        FRHICommandList& RHICmdList, 
                        const FViewInfo& View, 
                        ContextType DrawingContext, 
                        const FMeshBatch& Mesh, 
                        bool bPreFog, 
                        const FDrawingPolicyRenderState& DrawRenderState, 
                        const FPrimitiveSceneProxy* PrimitiveSceneProxy, 
                        FHitProxyId HitProxyId 
                        )

                    bool FTranslucencyDrawingPolicyFactory::DrawStaticMesh(
                        FRHICommandList& RHICmdList, 
                        const FViewInfo& View, 
                        ContextType DrawingContext, 
                        const FStaticMesh& StaticMesh, 
                        const uint64& BatchElementMask, 
                        bool bPreFog, 
                        const FDrawingPolicyRenderState& DrawRenderState, 
                        const FPrimitiveSceneProxy* PrimitiveSceneProxy, 
                        FHitProxyId HitProxyId 
                        )
                    ``` 
                    进行绘制，其静态函数函数内部，将会调用
                    ```
                    bool FTranslucencyDrawingPolicyFactory::DrawMesh(
                        FRHICommandList& RHICmdList, 
                        const FViewInfo& View, 
                        ContextType DrawingContext, 
                        const FMeshBatch& Mesh,  
                        const uint64& BatchElementMask, 
                        const FDrawingPolicyRenderState& DrawRenderState, 
                        bool bPreFog, 
                        const FPrimitiveSceneProxy* PrimitiveSceneProxy, 
                        FHitProxyId HitProxyId 
                        )
                    ```
                    * 在该函数中，会从 FMeshBatch 里边提取 FMaterial ，从 FMaterial 提取 EBlendMode ，设置一系列的渲染撞见，创建 FProcessBasePassMeshParameters 和 FDrawTranslucentMeshAction 栈对象，并以此为参数，调用
                        ```
                        template<typename ProcessActionType> void ProcessBasePassMesh(FRHICommandList& RHICmdList, const FProcessBasePassMeshParameters& Parameters, ProcessActionType&& Action)
                        ``` 
                        此时的参数 `ProcessActionType = FDrawTranslucentMeshAction` 。
                        
                        然后调用 
                        ```
                        template<typename LightMapPolicyType>
                        void FDrawTranslucentMeshAction::Process(
                            FRHICommandList& RHICmdList, 
                            const FProcessBasePassMeshParameters& Parameters, 
                            const LightMapPolicyType& LightMapPolicy, 
                            const typename LightMapPolicyType::ElementDataType& LightMapElementData
                            )
                        ```
                        此时的参数 `LightMapPolicyType = FUniformLightMapPolic`
                        * 在这个函数里边，将会获取 FScene 的数据，创建栈对象 `TBasePassDrawingPolicy<FUniformLightMapPolic>`
                            * 在这个对象的构造函数中，会调用
                                ```
                                template <typename LightMapPolicyType> void GetBasePassShaders(
                                    const FMaterial& Material, 
                                    FVertexFactoryType* VertexFactoryType, 
                                    LightMapPolicyType LightMapPolicy, 
                                    bool bNeedsHSDS,
                                    bool bEnableAtmosphericFog,
                                    bool bEnableSkyLight,
                                    FBaseHS*& HullShader,
                                    FBaseDS*& DomainShader,TBasePassVertexShaderPolicyParamType<typename LightMapPolicyType::VertexParametersType>*& VertexShader,
                                    TBasePassPixelShaderPolicyParamType<typename LightMapPolicyType::PixelParametersType>*& PixelShader
                                    )
                                ```
                                此时 `LightMapPolicyType = FUniformLightMapPolicy` 。是全特化后的模板函数

                                *  此函数将会调用 
                                    ```
                                    template <ELightMapPolicyType Policy>
                                    void GetUniformBasePassShaders(
                                        const FMaterial& Material, 
                                        FVertexFactoryType* VertexFactoryType, 
                                        bool bNeedsHSDS,
                                        bool bEnableAtmosphericFog,
                                        bool bEnableSkyLight,
                                        FBaseHS*& HullShader,
                                        FBaseDS*& DomainShader,
                                        TBasePassVertexShaderPolicyParamType<FUniformLightMapPolicyShaderParametersType>*& VertexShader,
                                        TBasePassPixelShaderPolicyParamType<FUniformLightMapPolicyShaderParametersType>*& PixelShader
                                        )
                                    ```
                                    * 此函数将会调用 
                                        ```
                                        template<typename ShaderType> ShaderType* FMaterial::GetShader(FVertexFactoryType* VertexFactoryType) const
                                        ```
                                        获取 DomainShader ， HullShader ， VertexShader ， PixelShader ，这些结果将会最终将会存放到 `TBasePassDrawingPolicy<LightMapPolicyType>` 中，并调用
                                        ```
                                        TBasePassDrawingPolicy<LightMapPolicyType>::SetupPipelineState
                                        TBasePassDrawingPolicy<LightMapPolicyType>::SetMeshRenderState
                                        ```
                                        设置渲染状态和从 FMeshBatch 获取 FMeshBatchElement 获取 shader 并输入。然后调用
                                        ```
                                        FBasePassReflectionParameters::SetMesh
                                        FMeshMaterialShader::SetMesh
                                        ```
                                        将一些反射数据填入和将一些顶点数据输入Context。
                                        最后调用
                                        ```
                                        FMeshDrawingPolicy::DrawMesh
                                        ```

*   UMaterial 的使用。
    
    UMaterial 在UE4里边是作为一种资源使用，对应的是材质蓝图。 UMaterial 在使用时会被创建为 UMaterialInstance ，传入到 Component 中被使用。 UMaterialInstance 中会通过 UMaterial 获取 FMaterialRenderProxy ，而 FMaterialRenderProxy 能过获取 FMaterial ，最后从 FMaterial 中获取 Shader。

    在 UMaterial 的初始化中，其构造函数执行完毕后，作为构造参数 FObjectInitializer 会析构，并且该析构函数 会调用 
    ```
    UObject::PostInitProperties
    |-> UMaterial::PostInitProperties
    ```
    *   这个方法将会对所有支持的 ERHIFeatureLevel 调用
        ```
        UMaterial::UpdateResourceAllocations
        ```
        *   这个函数将会调用 
            ```
            UMaterial::AllocateResource
            ```
            *   创建一个 FMaterialResource 对象。调用
                ```
                FMaterialResource::SetMaterial
                ```
                输入 UMaterial 和一些环境相关的数据。

    另外在 UMaterial::PostLoad 里边，会对当前引擎使用的 FeatureLevel 获取对应的 FMaterialResource ，并调用 UMaterial::CacheShadersForResources 
	
    调用 FMaterial::CacheShaders 。
		
    *   创建 FMaterialShaderMapId 实体，调用 FMaterial::GetShaderMapId 此方法被FMaterialResource::GetShaderMapId 派生。
			
        调用基类方法 FMaterial::GetShaderMapId 。
		
        *   调用 FMaterial::GetDependentShaderAndVFTypes 。
					
            轮询 FVertexFactoryType::GetTypeList() ，获取 FVertexFactoryType* ，如果 这个需要和材质一起使用的话，那么就轮询 FShaderType::GetTypeList() ，获取 FMeshMaterialShaderType* （里面还可能有其他的东西）。
			
            如果 FVertexFactoryType 和 FShaderType 能相互依赖，既各自调用 FVertexFactoryType::ShouldCache FShaderType::ShouldCache FMaterial::ShouldCache  那么将这个 FVertexFactoryType 和 FMeshMaterialShaderType 加入到 一个输出用的数组中。
			
            轮询 FShaderPipelineType::GetTypeList() ，获取 FShaderPipelineType* ，如果这个 FShaderPipelineType 是网格材质相关的，那么获取 FVertexFactoryType 和 FMeshMaterialShaderType ，如上询问 ShouldCache ，如果需要的话，加入到输出数组。
			
            接下来对剩下的 FMaterialShaderType 材质进行 ShouldCache ，然后加入到数组中。
			最后按名字对三个输出数组进行排序。
		
        将输出数组和渲染环境相关信息存放至 FMaterialShaderMapId 中。
	
    获取到了 FMaterialShaderMapId 数据，调用 UMaterial::GetReferencedFunctionIds 获取函数ID，并且压入 FMaterialShaderMapId 中。
	
    调用 UMaterial::GetReferencedParameterCollectionIds 获取参数 ID 并且压入 FMaterialShaderMapId 中。
	
    调用 UMaterial::GetForceRecompileTextureIdsHash 获取需要重新编译的纹理（？）
	
    如果存在 MaterialInstance，既被创建的是一个 UMaterialInstance 
	*   则获取需要重载的参数id并且压入 FMaterialShaderMapId。
		
        获取静态数据，压入 FMaterialShaderMapId。

调用并返回 FMaterial::CacheShaders
*   如果是 inline shaders ，那么直接从 TMap<FMaterialShaderMapId,FMaterialShaderMap*> FMaterialShaderMap::GIdToMaterialShaderMap[SP_NumPlatforms] 里边用 GameThreadShaderMap 自带的 FMaterialShaderMapId 找到 FMaterialShaderMap 数据并赋值给 GameThreadShaderMap 。
	
    *   使用 GameThreadShaderMap 调用 FMaterialShaderMap::Register 。
		
        *   对包含的所有 FShader 调用 void FShader::BeginInitializeResources() 。
			
            *   该函数直接调用 void BeginInitResource(FRenderResource* Resource)
				*   使用 ENQUEUE_UNIQUE_RENDER_COMMAND_ONEPARAMETER(InitCommand, ...) 发送命令，该命令调用 FRenderResource::InitResource 。
	
    如果不是，那么则从给定的 ShaderMapId 参数寻找 FMaterialShaderMap ，并赋值给 GameThreadShaderMap 。
	
    然后尝试编译。 有兴趣可以看看 FMaterial::BeginCompileShaderMap 。。。
	
    最后将 GameThreadShaderMap 赋值给 RenderingThreadShaderMap 。

* UMaterialInstance 材质变量输入

    `void UMaterialInstance::UMaterialInstance(FName ParameterName, class UTexture* Value);`
        从 `TArray<struct FTextureParameterValue> UMaterialInstance::TextureParameterValues` 查找所有的材质参数。
            如果不存在，就新建一个空的实例。
        判断该实例是否是输入的实例，如果不是，则赋值，并且调用  `GameThread_UpdateMIParameter` 在渲染线程中更新数据。
            -> 在渲染线程中，对调用 `FMaterialInstanceResource* UMaterialInstance::Resources[3]` 调用 `template <typename ValueType> void FMaterialInstanceResource::RenderThread_UpdateParameter(const FName Name, const ValueType& Value)`，在这个函数中将把这个名字的UTexture加入到 `TArray<TNamedParameter<const UTexture*> > FMaterialInstanceResource::TextureParameterArray`中。
        最后调用 `void CacheMaterialInstanceUniformExpressions(const UMaterialInstance* MaterialInstance)`。
            这里面对 `Resources[0]` 调用 `void FMaterialRenderProxy::CacheUniformExpressions_GameThread()` 里面渲染线程内部加入一行命令，对自身调用 `void FMaterialRenderProxy::CacheUniformExpressions()`
                -> 里面会遍历所有的FeatureLevel，然后获取对应的FMaterial，用自身和获取到的 FMaterial 构建 FMaterialRenderContext，并作为参数调用 `void FMaterialRenderProxy::EvaluateUniformExpressions(FUniformExpressionCache& OutUniformExpressionCache, const FMaterialRenderContext& Context, FRHICommandList* CommandListIfLocalMode) const`
                    -> 然后就是重新设置一下啥啥东西的。。。


* 一些宏定义命令分析

    ```
    ENQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_DECLARE_TEMPLATE(TypeName,TemplateParamName,{ParamTypeX, ParamNameX, ParamValueX}...,Code)
    -> 
        template <typename TemplateParamName> \
	    NQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_DECLARE_OPTTYPENAME(TypeName,{ParamTypeX, ParamNameX, ParamValueX}...,typename,Code)

    ENQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_DECLARE_OPTTYPENAME(TypeName,{ParamTypeX, ParamNameX, ParamValueX}...,OptTypename,Code)
    ->
        class EURCMacro_##TypeName : public FRenderCommand \
	    { \
	    public: \
		    EURCMacro_##TypeName({OptTypename TCallTraits<ParamTypeX>::ParamType In##ParamNameX}...): \
            {ParamNameX(In##ParamNameX)}...
		    {} \
		    TASK_FUNCTION(Code) \
		    TASKNAME_FUNCTION(TypeName) \
	    private: \
            {ParamTypeX ParamNameX}...
	    };
    
    TASK_FUNCTION(Code) 
    ->
        void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent) \
		{ \
			FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand(); \
			Code; \
		} 

    TASKNAME_FUNCTION(TypeName) 
    ->
		FORCEINLINE TStatId GetStatId() const \
		{ \
			RETURN_QUICK_DECLARE_CYCLE_STAT(TypeName, STATGROUP_RenderThreadCommands); \
		}

    ENQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_CREATE_TEMPLATE(TypeName,TemplateParamName,{ParamTypeX, ParamValueX}...)
    ->
        ENQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_CREATE(TypeName<TemplateParamName>,{ParamTypeX, ParamValueX}...)

    ENQUEUE_UNIQUE_RENDER_COMMAND_{XXXPARAMETER}_CREATE(TypeName,{ParamTypeX, ParamValueX}...)
    ->
	    { \
		    LogRenderCommand(TypeName); \
		    if(ShouldExecuteOnRenderThread()) \
		    { \
			    CheckNotBlockedOnRenderThread(); \
			    TGraphTask<EURCMacro_##TypeName>::CreateTask().ConstructAndDispatchWhenReady({ParamValueX}...); \
		    } \
		    else \
		    { \
			    EURCMacro_##TypeName TempCommand({ParamValueX}...); \
			    FScopeCycleCounter EURCMacro_Scope(TempCommand.GetStatId()); \
			    TempCommand.DoTask(ENamedThreads::GameThread, FGraphEventRef() ); \
		    } \
	    }

    ```

    未分类

    1，启动CVAR允许转存中间着色器
        在 ConsoleVariables.ini 文件（通常位于 Engine/Config/ConsoleVariables.ini）中，启用下列 Cvar：
            r.ShaderDevelopmentMode = 1
            r.DumpShaderDebugInfo = 1
            r.DumpShaderDebugShortNames = 1
    
    2，在调试时构建SHADERCOMPILEWORKER
        将 ShaderCompileWorker的解决方案属性更改为Debug_Program

    3，生成中间文件
        此刻，您想要生成可调试的文件；启用 Cvar 将允许后续编译转储所生成的文件；要强制重建所有着色器，请在Engine/Shaders/Common.usf 中添加一个空格或进行更改，然后重新运行编辑器。这将重新编译着色器并转储Project/Saved/ShaderDebugInfo 文件夹中的所有中间文件。

    4，转储的着色器的文件夹结构
        让我们分析转储的文件的完整路径：
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        项目的根路径：
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        转储着色器的根路径： 
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        现在，对于每种着色器格式/平台，您可找到一个子文件夹，在本例中，这是 PC D3D Shader Model 5：
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        现在，对于每个材质名称都有一个对应的文件夹，并且有一个名为 Global 的特殊文件夹。在本例中，我们在 `M_Egg` 材质内：     
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        着色器分组在贴图内并按顶点工厂排序，而这些顶点工厂通常最终对应于网格/元件类型；在本例中，有一个“本地顶点工厂”：
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf
        最后，下一路径是特性的排列；因为我们早先已启用 r.DumpShaderDebugShortNames=1，所以名称已压缩（以减小路径长度）。如果将其设置为 0，那么完整路径将是：
            D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\FLocalVertexFactory\TBasePassPSFNoLightMapPolicy\BasePassPixelShader.usf
        在该文件夹中，有一个批处理文件和一个 usf 文件。这个 usf 文件就是要提供给平台编译器的最终着色器代码。您可通过这个批处理文件来调用平台编译器，以查看中间代码。
        使用 SHADERCOMPILEWORKER 进行调试

    从 4.11 版开始，我们为 ShaderCompileWorker (SCW) 添加了一项功能，使其能够调试平台编译器调用；命令行如下所示：

        PathToGeneratedUsfFile -directcompile -format=ShaderFormat -ShaderType -entry=EntryPoint
        PathToGeneratedUsfFile 是 ShaderDebugInfo 文件夹中的最终 usf 文件
        ShaderFormat 是您想要调试的着色器平台格式（在本例中，这是 PCD3D_SM5）
        ShaderType 是 vs/ps/gs/hs/ds/cs 中的一项，分别对应于“顶点”、“像素”、“几何体”、“物体外壳”、“域”和“计算”着色器类型
        EntryPoint 是 usf 文件中此着色器的入口点的函数名称

    例如，D:\UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\M_Egg\LocalVF\BPPSFNoLMPolicy\BasePassPixelShader.usf-format=PCD3D_SM5 -ps -entry=Main

    现在，您可以对 D3D11ShaderCompiler.cpp 中的 CompileD3D11Shader() 函数设置断点，通过命令行运行 SCW，并且应该能够开始了解如何调用平台编译器。

    有关此主题的更多信息，请参阅虚幻引擎文档的以下两个页面：

    [着色器开发](https://docs.unrealengine.com/latest/INT/Programming/Rendering/ShaderDevelopment/index.html)
    [HLSL 交叉编译器](https://docs.unrealengine.com/latest/INT/Programming/Rendering/ShaderDevelopment/HLSLCrossCompiler/index.html)

    编译的时候，如果出现：
    2>  Creating makefile for ShaderCompileWorker (no existing makefile)
2>  Performing full C++ include scan (no include cache file)
2>  Distributing 77 actions to XGE
2>EXEC : Fatal error : Failed to open/create session data
2>  Failed to open/create session data: STATUS_ACCESS_DENIED

或者：
    1>EXEC : Fatal error : Failed to start build.
    1>Failed to connect to Build Service (on local machine): Server is not reachable

类似的错误的话，可以是因为联机编译的问题。

解决方式是将 C:\Users\%USERNAME%\AppData\Roaming\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml: 中设置联机服务器  bAllowXGE 为false 以关闭联机服务器
例：
    ```
        <?xml version="1.0" encoding="utf-8" ?>
            <Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
                <BuildConfiguration>
                    <bAllowXGE>false</bAllowXGE>
                </BuildConfiguration>
            </Configuration>
    ```

一旦虚幻4项目的帧率低于30，UE4将会禁止粒子碰撞以保证帧数保持在30以上。ISO项目默认锁30，需要在Project Settings > GeneralSettings > Framerate中修改 

UE4打包问题

ERROR : System Exception：Couldn't update resource

解决办法：把 Save 和 Intermediate 文件夹下的文件删除，然后重新打包即可。

HZB Steup Mips在GPU Profile里边占用了大量时间。
DX11的问题，将在DX12中修复。
