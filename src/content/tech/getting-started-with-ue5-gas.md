---
title: "UE5 Gameplay Ability System 入门指南"
description: "深入解析 GAS 核心概念：Ability、Attribute、GameplayEffect 的协作机制与实战配置。"
date: 2026-06-05
tags: ["UE5", "GAS", "C++", "架构设计"]
---

## 为什么选择 GAS

Gameplay Ability System 是 Unreal Engine 内置的技能框架，专为复杂战斗系统设计。相比手写状态机，GAS 提供了：

- **声明式技能定义**：通过 DataAsset 配置技能属性
- **内置预测与回滚**：网络同步开箱即用
- **可组合的 GameplayEffect**：Buff/Debuff 系统天然支持

## 核心组件

### GameplayAbility

技能的最小执行单元。每个 Ability 包含：

```cpp
UCLASS()
class UGA_MeleeAttack : public UGameplayAbility
{
    GENERATED_BODY()
public:
    virtual void ActivateAbility(...) override;
    virtual void EndAbility(...) override;
};
```

### AttributeSet

定义角色的数值属性（生命值、攻击力等）：

```cpp
UCLASS()
class UCombatAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadOnly)
    FGameplayAttributeData Health;
};
```

### GameplayEffect

属性修改器，支持 Instant / Duration / Infinite 三种生命周期。

## 下一步

后续文章将深入探讨 GameplayTag 的层级设计与技能取消窗口的实现。
