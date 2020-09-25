### UE4 GamePlayAbilitySystem(GAS)

[参考文档](https://github.com/tranek/GASDocumentation)

1. 前置
   1. 相关缩写
      1. GAS  : Gameplay Ability System  系统总称
      2. GAC  : Gameplay Ability Component 核心功能
      3. GA   : Gameplay Ability  技能类，大部分情况下只有数据没有逻辑
      4. GE   : Gameplay Effect 技能效果，储存数据和一些简单的加减逻辑，影响AS，需要网络同步。
      5. AT   : 作为GameplayEffect的Tick的代替品，由GE创建并且能在多线程环境下跑。
      6. AS   : Attribute Set 属性集，储存相关的数值数据，被GE修改。
      7. GC   : Gameplay Cure 技能效果，被GE创建，能多播到所有的客户端上。
   2. 逻辑流
      1. GAC绑定在对应的Actor上，在GAC上触发GA，由GA触发GE，GE影响AS并产生GC多播到所有客户端上。

2. 重要的相关类
   1. `UGameAblityComponent(GAC)`
   
        GAS的核心，一般作为Character，Controller或者PlayerState任选其一的子组件，用来触发对应的GAS功能。在使用之前需要进行一些设置：
        ```CPP
        AbilitySystem->InitAbilityActorInfo(AOwnerActor, AAvatarActor);
        ```
        【可能有误】这个函数有两个参数，第一个指的是OwnerActor，是用来标识储存这个Component的Actor，第二个参数，指的是AvatarActor，是用来标识GAS所作用的Actor。

        当更换Controller时，GAC 需要及时进行更新，所以一般在 ProccessBy 函数中被调用。

        ```cpp
        void XXXCharactor::PossessedBy(AController* NewController)
        {
            Super::PossessedBy(NewController);
	        if(IsValid(AbilitySystem))
		        AbilitySystem->RefreshAbilityActorInfo();
        }
        ```

        当设置完成后，即可开始技能的调用，但在调用之前，需要注册所能够使用的技能：
        ```CPP
        FGameplayAbilitySpec Space{ UGameplayAbility::StaticClass(), 1, 0 };
        // 只注册技能
        FGameplayAbilitySpecHandle Handle = AbilitySystem->GiveAbility(Space);
        
        // 注册并调用一次，调用完毕后立马删除
        FGameplayAbilitySpecHandle Handle2 = AbilitySystem->GiveAbilityAndActivateOnce(Space); 
        ```
        此时拿到的Handle可以用来让GAS删除某个特定的技能，或者用于触发（并不是必须）。删除使用的时下面这些函数：
        ```cpp
        // 移除指定技能
        UAbilitySystemComponent::ClearAbility

        // 当能力执行完毕后，移除指定能力。如果能力不在执行中，则直接移除；若在执行中，则该能力的输入会被清空，玩家无法再激活或是和该能力交互。
        UAbilitySystemComponent::SetRemoveAbilityOnEnd

        // 移除所有能力，不需要FGameplayAbilitySpecHandle。
        UAbilitySystemComponent::ClearAllAbilities
        ```
        技能的触发方式分为两种，一种是尝试触发，一种是强制触发，分别是：
        ```cpp
        // 不检查，直接运行
        UAbilitySystemComponent::CallActivateAbility

        // 检查，若不满足条件则不执行 
        UAbilitySystemComponent::TryActivateAbility 
        ```
        技能的取消则可以通过调用下面的函数来进行取消。
        ```cpp
        UAbilitySystemComponent::CancleAbility;
        ```

    1. `UGameplayAbility`
        
        用来写实际的技能逻辑。注意，这里大部分应该都是逻辑，而非技能的数据，技能的数据有另外存放的地方。在`UGameplayAbility`（以下见GA）初始化中，应该设置GA的一些设置，重要的有如下几个：

        * ReplicationPolicy(`TEnumAsByte<EGameplayAbilityReplicationPolicy::Type>`)
          * ReplicateNo 不同步（默认）
          * ReplicateYes 同步
        * InstancingPolicy(`TEnumAsByte<EGameplayAbilityInstancingPolicy::Type>`)
          * NonInstanced 不创建实例，全部使用同一个CDO（默认）
          * InstancedPerActor 每一个Actor都会创建一个实例。状态能被保存，并且可以被同步
          * InstancedPerExecution 每一次使用技能的时候都会创建一个实例，可以被同步但并不推荐
        * NetExecutionPolicy(`TEnumAsByte<EGameplayAbilityNetExecutionPolicy::Type>`)
          * LocalPredicted 本地和服务器都可以创建并且执行，如果本地和服务器都有，那么就会尝试在本地进行预测（默认）
          * LocalOnly 只会在本地执行
          * ServerInitiated 只会在服务器初始化，但如果本地有实例的话也会在本地执行
          * ServerOnly 只允许在服务器中执行
        * TriggerSource(`TEnumAsByte<EGameplayAbilityTriggerSource::Type>`)
          * GameplayEvent 从Gameplay Event中触发，和Payload一同。
          * OwnedTagAdded 如果GAS的所有者获得了一个Tag，那么触发一次
          * OwnedTagPresent 如果GAS的所有者获得一个Tag，那么获得的时候触发，并且当所有者的tag去除了，那么就取消

        关于GameplayAbility的一些Tag的设置：
        
        >PS: 用来储存Tag的一般有两个容器，一种是FGameplayTagContainer，可以认为就是一个普通的tag的容器，还有一种是FGameplayTagQuery，可以储存逻辑，比如A&&B这样的，用来与FGameplayTagContainer进行检测。

        * CancelAbilitiesMatchingTagQuery 当技能运行时，若按照设定的逻辑进行检测，若检测为真，会自动cancel。
        * CancelAbilitiesWithTag 同上，但是当包含这里面的tag时会立即Cancel
        * BlockAbilitiesWithTag 堵塞
        * ActivationOwnedTags 当该技能Active时，会自动加上。
        * ActivationRequiredTags 当角色拥有该技能时，才允许被Active。
        * ActivationBlockedTags 如果Actor或者Component拥有这个tag时，将自动堵塞。
        * SourceRequiredTags 当源Actor或者Component拥有改Tags时，将可以被启用。
        * SourceBlockedTags 当Actor或者Component拥有该Tags时，将会被堵塞。
        * TargetRequiredTags 当目标Actor或者Component拥有该tags时，才可以被调用。
        * TargetBlockedTags 当目标Actor或者Component拥有该Tags时，该技能会被堵塞。
        * AbilityTags 当前技能拥有该Tags。
        * 

        一些重要的函数。

          * 外部调用的成员函数：
          ```cpp
            // 由GAS调用，尝试启用GA(不一定能成功启动)
            UGameplayAbility::TryActivateAbility(...)
          ```

          * 可被重载的虚成员函数：
          ```cpp
            // 查询当前Ability是否能够被启用
            virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, 
              const FGameplayAbilityActorInfo* ActorInfo, 
              const FGameplayTagContainer* SourceTags = nullptr, 
              const FGameplayTagContainer* TargetTags = nullptr, 
              OUT FGameplayTagContainer* OptionalRelevantTags = nullptr
            ) const;
            // GA被激活时调用的函数
            virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, 
              const FGameplayAbilityActorInfo* ActorInfo, 
              const FGameplayAbilityActivationInfo ActivationInfo, 
              const FGameplayEventData* TriggerEventData
            );

            // 触发一些冷却，消耗相关的效果，在ActivateAbility中必须调用
            virtual bool CommitAbility(const FGameplayAbilitySpecHandle Handle, 
              const FGameplayAbilityActorInfo* ActorInfo, 
              const FGameplayAbilityActivationInfo ActivationInfo
            );
	
            // 当GA被外部中断时调用的函数
            virtual void CancelAbility(const FGameplayAbilitySpecHandle Handle, 
              const FGameplayAbilityActorInfo* ActorInfo, 
              const FGameplayAbilityActivationInfo ActivationInfo, 
              bool bReplicateCancelAbility
            );

            // 当GA结束时调用的函数
            virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, 
              const FGameplayAbilityActorInfo* ActorInfo, 
              const FGameplayAbilityActivationInfo ActivationInfo, 
              bool bReplicateEndAbility, bool bWasCancelled
            );

          ``` 
        
    2. UAttributeSet（AS）
        
        用来储存CAS中数值属性的相关玩意，GA中产生的效果将直接作用于AttributeSet中。一般的用法为继承UAttributeSet类，然后在类内部定义所需要的一个或多个变量值，这些变量值能够被GameplayEffect捕获并用于数值计算，如：
        
        ```cpp
        UCLASS(Blueprintable, BlueprintType)
        class UCharactorAttributeList : public UAttributeSet
        {
          GENERATED_UCLASS_BODY()
        public:
          UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_Health)
            FGameplayAttributeData Health;
          GAMEPLAYATTRIBUTE_PROPERTY_GETTER(UCharactorAttributeList, Health)
          UFUNCTION()
          virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
        };
        ```
        
        AS 如果创建为 Subobject ，并且与 GAC 在同一个类中，那么 GAC 会自动绑定 AS （只有绑定 GAC 的 AS 才会受到 GE 的作用）。可以在AS的属性同步回调中写一些逻辑游戏，以影响到游戏世界。

    3. UGameplayEffect（GE）

        GE一般不被继承，其重要的属性有 Modifiers 和 Executions ，用来储存和执行对某个属性的修改。Modifiers只支持内置的简单逻辑对属性进行简单修改，而Executions能够自己编写逻辑，对特定的属性进行修改。

        GE的生成实例代码如下：

        ```cpp
        UAbilitySystemComponent* Component = ActorInfo->AbilitySystemComponent.Get();
		    auto Context = Component->MakeEffectContext();
		    Component->ApplyGameplayEffectToSelf(UCustomEffect::StaticClass()->GetDefaultObject<UCustomEffect>(), 0.0, Context);
        ```
      
         * Modifiers
         * Executions

            要使用Executions，首先要继承 UGameplayEffectExecutionCalculation，并且复写执行函数，如下面代码所示：
            ```cpp
            UCLASS()
            class UCustomDamageExecution : public UGameplayEffectExecutionCalculation
            {
	            GENERATED_BODY()

            public:
	            // Constructor and overrides
	            UCustomDamageExecution();
	            virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
            };
            ```

            在执行函数中，首先要绑定AS里的属性。
            ```cpp
            // 定义，这里定义了一个名字，用来储存名为‘Health’的属性（不是属性集，是属性集里面的一个属性），这个名字为Health（强制同名）。
            DECLARE_ATTRIBUTE_CAPTUREDEF(Health);

            // 执行，这里将上面定义的Health，绑定到 UCharactorAttributeList（AS）里边的Health变量。
            // Target表示捕获目标的属性
            DEFINE_ATTRIBUTE_CAPTUREDEF(UCharactorAttributeList, Health, Target, false);
            ```

            捕获完毕后，可以在执行函数中用这个属性去获取属性值或者修改属性值。
            ```cpp
            void UCustomDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, 
            OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput
            ) const
            {
                // HealthDef 由 DECLARE_ATTRIBUTE_CAPTUREDEF(Health) 定义
                ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(HealthDef, EvaluationParameters, 0.0f/*defaultvalue*/);
                // HealthProperty 也是由 DECLARE_ATTRIBUTE_CAPTUREDEF(Health) 定义，定义一个操作
                FGameplayModifierEvaluatedData Operator(HealthProperty, EGameplayModOp::Additive, 1000);
                // 把创建出来的操作压入到输出列表中
                OutExecutionOutput.AddOutputModifier(Operator);
            }
            ```

        关于数据的传输。

          * Set By User

            用户能通过单个GameplayTag来传递一个float类型的数据到GE中。
            
            ```cpp
              // GE中调用
              auto Context = Component->MakeEffectContext();
		          auto Space = Component->MakeOutgoingSpec(UCustomEffect::StaticClass(), 0.0, Context);
		          Space.Data->SetSetByCallerMagnitude(FName(TEXT("Shift.X")), 100.0f);
		          auto hand = Component->ApplyGameplayEffectSpecToSelf(*Space.Data.Get());

              //Executions 中获取
              auto Space = ExecutionParams.GetOwningSpecForPreExecuteMod();
	            auto Damage = Space->GetSetByCallerMagnitude(FName(TEXT("Shift.X")), false, 2.0f);
            ```



        

        