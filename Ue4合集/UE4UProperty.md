[Officical Document](https://docs.unrealengine.com/en-us/Programming/UnrealArchitecture/Reference/Properties/Specifiers)

#UProperty

UProperty用来标记成员变量，用来记录反射或GC信息。其形式如下：

```cpp
 UPROPERTY(specifier, specifier, ..., meta=(key=value, key=value, ...))
 Type VariableName;
```

没有被标记的成员变量将不会被GC所管理，需要手动管理。

##Specifiers

* **AdvancedDisplay**

    属性显示在详细信息窗格的高级下拉菜单中。

* **AssetRegistrySearchable**

    表示这个属性和它的值将被自动添加到任何一个包含成员变量的素材类实例的素材注册表。在结构体中的属性或参数上使用此关键字是非法的。

* **BlueprintAssignable**

    只能用于多播委任。暴露该属性，能在蓝图中赋值。

* **BlueprintAuthorityOnly**

    只能用于多播委任。在蓝图中，只允许绑定带有`BlueprintAuthorityOnly`标签的`event`。

* **BlueprintCallable**

    只能用于多播委任。暴露该蓝图，能够在蓝图中调用。

* **BlueprintReadOnly**

    此属性可以在蓝图中都读取，但不能修改。

* **BlueprintReadWrite**

    此属性允许在蓝图中被读或写。

* **Config**

    被该标记的变量，能保存在`.ini`中，并在创建时读取。该值不能设置默认值，并且隐含 `BlueprintReadOnly`。例如：

    ```
    [/Script/ShooterGame.ShooterGameInstance]
    Verifier = false
    Muliplayer = true
    ```

* **DuplicateTransient**

    被标记的值，在被制造副本时，将会被重置为默认值。

* **EditAnywhere**

    能被所有地方编辑。与其他控制可见性的标签冲突。

* **EditDefaultsOnly**

    只能在本类中设置，会影响该类所有的派生。

* **EditFixedSize**

    只能标记动态数组，这个将会阻止用户在属性窗口中改变数组的大小。

* **EditInline**

    只能标记`Object`的引用，允许用户编辑这个变量所引用的`Object`。

* **EditInstanceOnly**

    只能在属性窗口中编辑，只影响这个实例。

* **Export**

    只能标记对象或对象数组。表示当对象被复制或导出时，被分配给该属性的对象应完全作为子对象区块来导出，而不是仅仅输出对象引用本身。

* **GlobalConfig**

    同`Config`，但无法在子类中重载它。

* **Instanced**

    只能用于`UCLASS`变量。当被复制时，该变量将会复制一个新的实例，而非变量的一个指针。隐含 `EditInline`和`Export`。

* **Interp**

    表示该值可由`matinee`的浮点或向量属性轨迹来随时间驱动。

* **Localized**

    该变量的值将定义本地值，最常用于字符串。隐含`ReadOnly`。

* **Native**

    该变量为原生变量，将由CPP代码进行序列化，并且带GC。

* **NoClear**

    该值不可为空。

* **NoExport**

    只允许标记在原生类型中标记。这个不能再自动生成的类型中使用。

* **NonPIEDuplicateTransient**

    除了在PIE模式下，其余都为`DuplicateTransient`属性。

* **NonTransactional**

    变量的改变将不记录到编辑器的历史中。

* **NotReplicated**

    跳过同步。

* **Replicated**

    该变量将通过网络同步。

* **RepRetry**

    仅用于结构体属性。如无法被完全发送，请重试复制此属性（例如，对象引用尚无法通过节点网络来进行序列化）。对于简单引用来说，这是一个默认值，但对结构体来说，由于带宽消耗，很多情况下我们不需要。所以除非此标识被定义，否则其会被禁用。

* **SaveGame**

    被标记的变量在SaveGame时其值会被保存。

* **SerializeText**

    Native属性应以文本形式进行序列化（导入文本，导出文本）。

* **SkipSerialization**

    这个将会被跳过序列化，但是还是能够被结构化成文本格式。

* **SimpleDisplay**

    使属性在细节面板中默认可见。

* **TextExportTransient**

    这个属性将不会被导出成文本格式。

* **Transient**

    这个属性将不会被保存或读取。

* **VisibleAnywhere**

    这个属性在所有的属性面板中只显示不编辑。

* **VisibleInstanceOnly**

    这个属性只在实例面板中显示，不编辑。

##Meta KeyValuePair

* **BlueprintGetter=GetterFunctionName**
    
    当蓝图获取该值时，将调用其设置的函数，并将该函数的输出作为该变量的输出。例如：
    
    ```cpp
    UPROPERTY(BlueprintReadWrite, Meta=(BlueprintGetter = "WTF")) uint8 CastShader;

	UFUNCTION(BlueprintPure) uint8 WTF() const { return 255; }
    ```

    在蓝图上获取该数值时，将调用WTF函数并将其函数作为输出。

* **BlueprintSetter=SetterFunctionName**

    同上，不过是作为输入。

* **Category="TopCategory|SubCategory|..."**

    在蓝图编辑器中设置类别。

* **ReplicatedUsing=FunctionName**

    同步时调用该参数。