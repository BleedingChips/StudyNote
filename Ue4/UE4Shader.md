（示例版本，4.17）
* 着色器变量

    * 定义变量块类型
        ```cpp
        BEGIN_UNIFORM_BUFFER_STRUCT(StructTypeName, PrefixKeywords)
        BEGIN_UNIFORM_BUFFER_STRUCT_WITH_CONSTRUCTOR(StructTypeName, PrefixKeywords)
        ```
        其中`StructTypeName`这个表示变量的类型，`PrefixKeywords`表示类型的`attributes`([见cppreference](https://en.cppreference.com/w/cpp/language/attributes))。
        其中`BEGIN_UNIFORM_BUFFER_STRUCT_WITH_CONSTRUCTOR`需要自己在外部添加`StructTypeName::StructTypeName()`的实现。

    * 定义包含的变量
        ```cpp
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_ARRAY_EX(MemberType,MemberName,ArrayDecl,Precision)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_ARRAY(MemberType,MemberName,ArrayDecl)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER(MemberType, MemberName)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_EX(MemberType,MemberName,Precision)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_SRV(ShaderType,MemberName)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_SAMPLER(ShaderType,MemberName)
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_TEXTURE(ShaderType,MemberName)
        //NOT SUPPORTED YET
        //DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER_UAV(ShaderType,MemberName)
        ```
        `MemberType`：变量的类型
        `MemberName`：变量的名字
        `ArrayDecl`：数组声明符，如`[10086]`，若是非数组，则留空。
        `Precision`：储存精度描述符，控制变量在USF中的储存精度，其为
            
        ```cpp
        namespace EShaderPrecisionModifier
        {
	        enum Type
	        {
		        Float, // 32位
		        Half, // 16位
		        Fixed // 定点数
	        };
        };
        ```

        中的一种。赋值的时候要带上namespace说明符。默认为Float。

    * 结束定义
        ```cpp
        END_UNIFORM_BUFFER_STRUCT(StructTypeName)
        ```

    * 实现变量
        ```cpp
        IMPLEMENT_UNIFORM_BUFFER_STRUCT(StructTypeName,ShaderVariableName)
        ```
        一般需要放在.cpp文件中。这里将会声明变量的反射信息。并且之后该变量可以在**任何**`.usf`文件中被使用。
    
    * 示例
        ```cpp
        //.h
        BEGIN_UNIFORM_BUFFER_STRUCT(FValueType01, )
        DECLARE_UNIFORM_BUFFER_STRUCT_MEMBER(float, FloatValue)
        END_UNIFORM_BUFFER_STRUCT(FValueType01)

        //.cpp
        IMPLEMENT_UNIFORM_BUFFER_STRUCT(FValueType01,FValueType01Instance)

        //.usf 直接使用，不需要声明
        float p = FValueType01Instance.FloatValue;
        ```
        注：任然需要在`FShader`的派生类中，将其值传入。
    
* 着色器定义

    * 基本定义：
        ```cpp
        //.h
        class FShaderXXX : public FGobalShader //1
        {
            DECLARE_SHADER_TYPE(FComputeShaderDeclaration, Global); //2
        public:
            FShaderXXX() {} // 在 DECLARE_SHADER_TYPE 中使用的默认构造函数

            // 在 DECLARE_SHADER_TYPE 中使用的带初始化的构造函数，只会在usf重新编译，或者序列化失败后被调用。
            // 获取usf文件反射信息的位置。
            FShaderXXX(const ShaderMetaType::CompiledShaderInitializerType& Initializer) : 
                FGlobalShader(Initializer)
            {
                //...
            }

            // 在 IMPLEMENT_SHADER_TYPE 中使用的成员函数。
            static bool ShouldCache(EShaderPlatform Platform) 
            { return IsFeatureLevelSupported(Platform, ERHIFeatureLevel::SM5); } 

            // 在编译时调用，可选，用以覆盖 FGobalShader::ModifyCompilationEnvironment
            static void ModifyCompilationEnvironment(EShaderPlatform Platform, FShaderCompilerEnvironment& OutEnvironment) 
            {
                FGlobalShader::ModifyCompilationEnvironment(Platform, OutEnvironment);
                OutEnvironment.CompilerFlags.Add(CFLAG_StandardOptimization);
            } 
        };

        //.cpp
        IMPLEMENT_SHADER_TYPE(, FShaderXXX, TEXT("/Plugin/ComputeShader/Private/ComputeShaderExample.usf"), TEXT("MainComputeShader"), SF_Compute); // 3
        ```

        1. 
        2. `DECLARE_SHADER_TYPE(ShaderClass, ShaderMetaTypeShortcut)`，其中 `ShaderMetaTypeShortcut`将会合成成为`F+ShaderMetaTypeShortcut+ShaderType`，在示例中，则为`FGobalShaderType`。在当前版本中，共有`FGlobalShaderType`，`FMaterialShaderType`，`FNiagaraShaderType`，`FMeshMaterialShaderType`，`FGlobalShaderType`。大概是用来控制着色器的编译。
        3. 实现着色器。
        
            `IMPLEMENT_SHADER_TYPE(TemplatePrefix,ShaderClass,SourceFilename,FunctionName,Frequency)`。`TemplatePrefix`暂时不确定有啥用，一般缺省；`ShaderClass`着色器的类型名。`SourceFilename`为`.usf`文件的路径；`FunctionName`入口函数名字；`Frequency`频率？更倾向于是着色器的类型。定义在`enum EShaderFrequency`中，有效值为`SF_Vertex`,`SF_Hull`,`SF_Domain`,`SF_Pixel`,`SF_Geometry`,`SF_Compute`。

* 使用着色器

    * 发起一个命令到渲染线程。
        ```
        // 在逻辑线程中调用的函数
        void FunctionCallInLogic()
        {
            ENQUEUE_UNIQUE_RENDER_COMMAND( // 1
                FPixelShaderRunner,
                {
                    // 保证运行在渲染线程中
                    check(IsInRenderingThread());

                    // 获取RHI 接口。
                    FRHICommandListImmediate& RHICmdList = GRHICommandList.GetImmediateCommandList();

                    // 获取着色器引用 （ERHIFeatureLevel::Type FeatureLevel）
                    TShaderMapRef<FVertexShaderExample> VertexShader(GetGlobalShaderMap(FeatureLevel));
                    
                    // 获取图形环境
                    FGraphicsPipelineStateInitializer GraphicsPSOInit;
                    RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);

                    //...设置渲染选项

                    // 设置着色器
                    GraphicsPSOInit.BoundShaderState.PixelShaderRHI = GETSAFERHISHADER_PIXEL(*PixelShader);

                    // 应用所有设置
                    SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit);

                    // ...执行绘制
                }
            );
        }
        ```
    
        1. 该宏其有以下形式：
            ```cpp
            // 无参数的调用
            ENQUEUE_UNIQUE_RENDER_COMMAND(TypeName,Code)
            // 带一个参数的调用
            ENQUEUE_UNIQUE_RENDER_COMMAND_ONEPARAMETER(TypeName,ParamType1,ParamName1,ParamValue1,Code)
            // ...

            // 带六个参数的调用
            ENQUEUE_UNIQUE_RENDER_COMMAND_SIXPARAMETER(TypeName,ParamType1,ParamName1,ParamValue1,ParamType2,ParamName2,ParamValue2,ParamType3,ParamName3,ParamValue3,ParamType4,ParamName4,ParamValue4,ParamType5,ParamName5,ParamValue5,ParamType6,ParamName6,ParamValue6,Code)
            ```
            * `TypeName`表示在当前逻辑下会生成一个 `class  EURCMacro_ + TypeName : public FRenderCommand`的类型，用以储存变量名并生成一个闭包。并且产生相应的LOG。
            * `ParamType`表示储存值的类型。
            * `ParamName`表示在`code`中所能访问的变量名。
            * `ParamValue`表示用该值传递。
            * `Code`，需要带有`{}`，为执行的代码。里边只可以调用`ParamName`声明的名字，如同在一个成员函数内部一样。

* 着色器反射

    着色器反射信息一般保存在`FShader`的派生类的成员变量内，通常用于绑定裸信息，既非通过定义着色器变量定义的类型。一般不存特定信息。

    * `FShaderParameter` 绑定Buffer的。（HLSL下大概为CBuffer）
    * `FShaderResourceParameter` 绑定纹理和贴图资源的。（HLSL下为Texture和Sampler）
    * `FRWShaderParameter` 绑定UAV和SRV的。
    * `FShaderUniformBufferParameter` 绑定着色器变量的。
    * `template<typename TBufferStruct> class TShaderUniformBufferParameter` 同上，绑定特定着色器变量。


