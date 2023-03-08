# Knowledge Memo - Gameplay Ability System

## 2023/2/27

### 自定义应用条件 Custom Application Requirement
CustomApplicationRequirement (CAR) 类给设计者提供了是否能够应用GameplayEffect的高级控制手段（有别于简单的标签控制）。 可以通过在蓝图中重载CanApplyGameplayEffect()或者在C++中重载CanApplyGameplayEffect_Implementation()实现。

何时需要使用CARs？比如：

Target需要有一定数量的属性时
Target需要GameplayEffect堆叠到一定数量时
除此之外CARs还能够做更多事情，比如检查Target是否应用了一个GameplayEffect 的实例，在应用一个新实例时如果同类型的实例已存在则只改变其持续时间（CanApplyGameplayEffect()要返回false）。

### 技能消耗 Cost Gameplay Effect
GameplayAbilities 可以指定一个处理技能消耗的GameplayEffect。如果无法满足消耗，则技能不会被激活。Cost GE必须是一个Instant GameplayEffect其中可以有一个或多个Modifiers用于减去技能所需的属性消耗。默认情况下，Cost GEs是支持预测的，所以最好不要使用ExecutionCalculations，建议只使用MMCs完成消耗计算。

刚开始，你可能会为每一个带有消耗的GA创建一个对应的Cost GE。高阶方法是，对于多个GAs复用一个Cost GE，仅通过GA的消耗数据（消耗值定义在GA上）修改从Cost GE创建出的GameplayEffectSpec。仅能用于Instanced Abilities。

两种使用Cost GE的方法：
 
Use an MMC，这是最简单的方法。 创建一个MMC从GameplayAbility实例中读取消耗值：

    float UPGMMC_HeroAbilityCost::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
    {
	    const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());

		if (!Ability)
		{
			return 0.0f;
		}

		return Ability->Cost.GetValueAtLevel(Ability->GetAbilityLevel());
	}

在这个示例中，通过在派生的GameplayAbility 中添加FScalableFloat保存GA的消耗：

	UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cost")
	FScalableFloat Cost;

### 游戏效果容器 Gameplay Effect Containers
Epic官方的Action RPG 示例项目实现了一个叫FGameplayEffectContainer的结构，对于包含GameplayEffects 和TargetData极为方便。它自动化了一些工作，像根据GameplayEffects 创建GameplayEffectSpecs，设置GameplayEffectContext的默认值。在GameplayAbility中创建一个GameplayEffectContainer并且将它传递给生成的炮弹是非常简单和直接的。我并没有在示例项目中实现GameplayEffectContainers，但还是强烈建议了解这个并考虑将其添加到你的项目中。

要访问GameplayEffectContainers中的GESpecs，需要展开FGameplayEffectContainer然后通过索引GESpecs可以得到具体的GESpec。这需要在刚开始就知道你想访问的GESpec索引是多少。

GameplayEffectContainers还包括可选的目标选取方式。

### 激活技能 Activating Abilities
如果为一个GameplayAbility分配了输入操作，当输入按下并且GameplayTag 满足技能就会被释放。这并不总是期望的GameplayAbility激活方式，ASC还提供了其他四种激活GameplayAbilities的方法：通过GameplayTag，GameplayAbility类，GameplayAbilitySpec句柄（Handle）和事件。通过事件激活一个GameplayAbility允许你传递一些事件数据（Payload）。

	UFUNCTION(BlueprintCallable, Category = "Abilities")
	bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);
	
	UFUNCTION(BlueprintCallable, Category = "Abilities")
	bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation = true);
	
	bool TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation = true);
	
	bool TriggerAbilityFromGameplayEvent(FGameplayAbilitySpecHandle AbilityToTrigger, FGameplayAbilityActorInfo* ActorInfo, FGameplayTag Tag, const FGameplayEventData* Payload, UAbilitySystemComponent& Component);
	
	FGameplayAbilitySpecHandle GiveAbilityAndActivateOnce(const FGameplayAbilitySpec& AbilitySpec);

要通过事件激活一个GameplayAbility，需要在GameplayAbility中添加一个Triggers，分配一个GameplayTag，选择GameplayEvent。然后通过UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)发送事件。通过事件激活一个GameplayAbility允许你传递一些事件数据（Payload）。

GameplayAbility Trigger也可以在GameplayTag添加或删除时激活GameplayAbility。

注意： 当通过事件激活一个GameplayAbility，你必须在技能蓝图中使用ActivateAbilityFromEvent节点，并且ActivateAbility节点不能存在于你的蓝图中。如果技能蓝图中ActivateAbility节点存在，它将始终被调用，ActivateAbilityFromEvent则不会被调用。

注意： 不要忘记在GameplayAbility结束时调用EndAbility()，除非你想要一个一直运行的被动技能。

本地预测GameplayAbilities的激活序列过程：

Owning client 调用TryActivateAbility()
调用InternalTryActivateAbility()
调用CanActivateAbility()检查GameplayTag、消耗、冷却等决定是否能够释放技能
调用CallServerTryActivateAbility()并且传递生成好的Prediction Key
调用CallActivateAbility()
调用PreActivate()
调用ActivateAbility() 最终施放技能
Server receives CallServerTryActivateAbility()

调用ServerTryActivateAbility()
调用InternalServerTryActivateAbility()
调用InternalTryActivateAbility()
调用CanActivateAbility()
调用ClientActivateAbilitySucceed()在服务器确定激活成功时更新ActivationInfo并且广播OnConfirmDelegate委托（这不同于输入确认）
调用CallActivateAbility()
调用PreActivate()
调用ActivateAbility() 最终施放技能
如果服务器激活技能失败，将会调用ClientActivateAbilityFailed()并立即终止客户端的GameplayAbility并回退任何可预测的修改。

## 2023/3/2
如何在3DUI上做点击
启用Recive Hardware Input选项，就能用鼠标像UI一样点击，注意这里警告如果是做VR游戏，这个不要选

![](https://pic2.zhimg.com/80/v2-505696ad26172782d613864986acb559_720w.webp)


## 2023/3/8
在Standalone的情况下，GamplayEffect中配置GameplayCue，GameplayCue触发时会跑两次OnActive和两次OnRemove。在打开ds的情况下是正常的，通过断点猜测Standalone时GAS内部的RPC导致多跑了一次。