# Knowledge Memo - Gameplay Ability System

##2023/2/27
4.5.13 自定义应用条件 Custom Application Requirement
CustomApplicationRequirement (CAR) 类给设计者提供了是否能够应用GameplayEffect的高级控制手段（有别于简单的标签控制）。 可以通过在蓝图中重载CanApplyGameplayEffect()或者在C++中重载CanApplyGameplayEffect_Implementation()实现。

何时需要使用CARs？比如：

Target需要有一定数量的属性时
Target需要GameplayEffect堆叠到一定数量时
除此之外CARs还能够做更多事情，比如检查Target是否应用了一个GameplayEffect 的实例，在应用一个新实例时如果同类型的实例已存在则只改变其持续时间（CanApplyGameplayEffect()要返回false）。