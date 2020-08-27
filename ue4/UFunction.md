[Officical Document](https://docs.unrealengine.com/zh-CN/Programming/UnrealArchitecture/Reference/Functions/index.html)

#UFunction


UFunction用来标记成员变量，用来记录反射或GC信息。其形式如下：

```cpp
 UFUNCTION([specifier1=setting1, specifier2, ...], [meta(key1="value1", key2, ...)])
ReturnType FunctionName([Parameter1, Parameter2, ..., 
    ParameterN1=DefaultValueN1, ParameterN2=DefaultValueN2]) [const];
```

UFunction用来标记函数使其对蓝图公开，并且也支持绑定到委托，也能充当网络回调，也可以创建自己的控制台命令。

##Specifiers

* **BlueprintAuthorityOnly**

    如果在具有网络权限的机器上运行（服务器、专用服务器或单人游戏），此函数将仅从蓝图代码执行。

* **BlueprintCallable**

    此函数可在蓝图或关卡蓝图图表中执行。

* **BlueprintCosmetic**

    此函数为修饰性的，无法在专用服务器上运行。

* **BlueprintImplementableEvent**

    此函数可在蓝图或关卡蓝图图表中实现。

* **BlueprintNativeEvent**

    此函数旨在被蓝图覆盖掉，但是也具有默认原生实现。用于声明名称与主函数相同的附加函数，但是末尾添加了`Implementation，是写入代码的位置。如果未找到任何蓝图覆盖，该自动生成的代码将调用 Implementation` 方法。

* **BlueprintPure**

    此函数不对拥有它的对象产生任何影响，可在蓝图或关卡蓝图图表中执行。

* **CallInEditor**

    可通过细节（Details）面板`中的按钮在编辑器中的选定实例上调用此函数。

* **Category = "TopCategory|SubCategory|Etc"**

    在蓝图编辑工具中显示时指定函数的类别。使用 | 运算符定义嵌套类别。

* **Client**

    此函数仅在拥有在其上调用此函数的对象的客户端上执行。用于声明名称与主函数相同的附加函数，但是末尾添加了`Implementation`。必要时，此自动生成的代码将调用 `Implementation` 方法。

* **CustomThunk**

    UnrealHeaderTool 代码生成器将不为此函数生成thunk，用户需要自己通过 DECLARE_FUNCTION 或 DEFINE_FUNCTION 宏来提供thunk。

* **Exec**

    此函数可从游戏内控制台执行。仅在特定类中声明时，Exec命令才有效。

* **NetMulticast**

    此函数将在服务器上本地执行，也将复制到所有客户端上，无论该Actor的 NetOwner 为何。

* **Reliable**

    此函数将通过网络复制，并且一定会到达，即使出现带宽或网络错误。仅在与`Client`或`Server`配合使用时才有效。

* **SealedEvent**

    无法在子类中覆盖此函数。SealedEvent`关键词只能用于事件。对于非事件函数，请将它们声明为`static`或`final，以密封它们。

* **ServiceRequest**

    此函数为RPC（远程过程调用）服务请求。这意味着 NetMulticast 和 Reliable。

* **ServiceResponse**

    此函数为RPC服务响应。这意味着 NetMulticast 和 Reliable。

* **Server**

    此函数仅在服务器上执行。用于声明名称与主函数相同的附加函数，但是末尾添加了 _Implementation，是写入代码的位置。必要时，此自动生成的代码将调用 _Implementation 方法。

* **Unreliable**

    此函数将通过网络复制，但是可能会因带宽限制或网络错误而失败。仅在与`Client`或`Server`配合使用时才有效。

* **WithValidation**

    用于声明名称与主函数相同的附加函数，但是末尾需要添加`_Validate`。此函数使用相同的参数，但是会返回`bool`，以指示是否应继续调用主函数。

##Meta KeyValuePair

* **AdvancedDisplay="Parameter1, Parameter2, .."**
    
    以逗号分隔的参数列表将显示为高级引脚（需要UI扩展）。

* **AdvancedDisplay=N**

    用一个数字替代 N ，第N之后的所有参数将显示为高级引脚（需要UI扩展）。举例而言：'AdvancedDisplay=2' 将把前两个之外的所有参数标记为高级。

* **ArrayParm="Parameter1, Parameter2, .."**

    说明 BlueprintCallable 函数应使用一个Call Array Function节点，且列出的参数应被视为通配符数组属性。

* **ArrayTypeDependentParams="Parameter"**

    使用 ArrayParm 时，此说明符将指定一个参数，其将确定 ArrayParm 列表中所有参数的类型。

* **AutoCreateRefTerm="Parameter1, Parameter2, .."**
    
    如列出参数（由引用传递）的引脚未连接，其将拥有一个自动创建的默认项。这是蓝图的一个便利功能，经常在数组引脚上使用。

* **BlueprintAutocast**

    仅能由来自蓝图函数库的静态 BlueprintPure 函数使用。Cast节点将根据返回类型和函数首个参数的类型来自动添加。

* **BlueprintInternalUseOnly**

    此函数是一个内部实现细节，用于实现另一个函数或节点。其从未直接在蓝图图表中公开。

* **BlueprintProtected**

    此函数只能在蓝图中的拥有对象上调用。其无法在另一个实例上调用。

* **CallableWithoutWorldContext**

    用于拥有一个 WorldContext 引脚的 BlueprintCallable 函数，说明函数可被调用，即使其类不实现 GetWorld 函数也同样如此。

* **CommutativeAssociativeBinaryOperator**

    说明 BlueprintCallable 函数应使用Commutative Associative Binary节点。此节点缺少引脚命名，但拥有一个创建额外输入引脚的 添加引脚（Add Pin） 按钮。

* **CompactNodeTitle="Name"**

    说明 BlueprintCallable 函数应在压缩显示模式中显示，并提供在该模式中显示的命名。

* **CustomStructureParam="Parameter1, Parameter2, ..")**

    列出的参数都会被视为通配符。此说明符需要 UFUNCTION-level specifier、CustomThunk，而它们又需要用户提供自定义的 exec 函数。在此函数中，可对参数类型进行检查，可基于这些参数类型执行合适的函数调用。永不应调用基础 UFUNCTION，出现错误时应进行断言或记录。

    >  要声明自定义 exec 函数，使用语法 DECLARE_FUNCTION(execMyFunctionName)，MyFunctionName 为原函数命名。

* **DefaultToSelf**

    用于 BlueprintCallable 函数，说明对象属性的命名默认值应为节点的自我情境。

* **DeprecatedFunction**

    蓝图对此函数进行引用时将引起编译警告，告知用户函数已废弃。可使用 DeprecationMessage 元数据说明符添加到废弃警告消息（如提供说明如何替代已废弃的函数）。

* **DeprecationMessage="Message Text"**

    如果函数已废弃，尝试编译使用此函数的蓝图时，其将被添加到标准废弃警告。

* **DevelopmentOnly**
  
    被标记为 DevelopmentOnly 的函数只会在Development模式中运行。这适用于调试输出之类的功能（但其不应存在于发布产品中）。

* **DisplayName="Blueprint Node Name"**
  
    此节点在蓝图中的命名将被此处提供的值所取代，而非代码生成的命名。

* **ExpandEnumAsExecs="Parameter"**

    用于 BlueprintCallable 函数，说明应为参数使用的 列举 中的每个条目创建一个输入执行引脚。命名参数必须是引擎通过 UENUM 标签识别的一个列举类型。

* **HidePin="Parameter"**

    用于 BlueprintCallable 函数，说明参数引脚应从用户视图中隐藏。注意：使用此方式每个函数只能隐藏一个参数引脚。

* **HideSelfPin**

    隐藏用于指出函数调用所处对象的self引脚。self引脚在与调用蓝图的类兼容的 BlueprintPure 函数上为自动隐藏状态。这通常与 DefaultToSelf 说明符共用。

* **InternalUseParam="Parameter"**

    与 HidePin 相似，这将在用户视图中隐藏命名参数的引脚，只能用于一个函数的一个参数。

* **KeyWords="Set Of Keywords"**

    指定在搜索此函数时可使用的一套关键词，例如合适放置节点在蓝图图表中调用函数。

* **Latent**

    说明一个延迟操作。延迟操作拥有类型为 FLatentActionInfo 的一个参数，此参数由 LatentInfo 说明符命名。

* **LatentInfo="Parameter"**

    用于延迟 BlueprintCallable 函数，说明哪个参数是LatentInfo参数。

* **MaterialParameterCollectionFunction**

    用于 BlueprintCallable 函数，说明应使用材质覆盖节点。

* * **NativeBreakFunc**
    
    用于 BlueprintCallable 函数，说明函数应以标准Break Struct节点的方式进行显示。

* **ShortToolTip="Short tooltip"**

    完整提示文本过长时使用的简短提示文本，例如父类选取器对话。

* **ToolTip="Hand-written tooltip"**

    覆盖从代码注释自动生成的提示文本。

* **UnsafeDuringActorConstruction**

    在Actor构造时调用此函数并非安全操作。

* **WorldContext="Parameter"**

    由 BlueprintCallable 函数使用，说明哪个参数决定运算正在发生的World。

## Function Parameter Specifiers

* **Out**

    声明由引用传递的参数，使函数对其进行修改。

* * **Optional**

    通过任选关键词可使部分函数参数变为任选，便于调用。任选参数的数值（调用方未指定）取决于函数。例如，SpawnActor 函数使用任选位置和旋转，默认为生成的 Actor 根组件的位置和旋转。添加 = [value] 参数可指定任选参数的默认值。例如：function myFunc(optional int x = -1)。在多数情况下，如无数值被传递到任选参数，将使用变量类型的默认值或零（例如 0、false、""、none）。