---
title: "《黄金替罪羊》核心循环设计"
description: "从原型到可玩：2D 解谜游戏核心玩法循环的架构设计与技术选型。"
date: 2026-06-05
tags: ["Unity", "游戏设计", "架构", "解谜"]
---

## 设计目标

《黄金替罪羊》的核心体验是"用替身化解危机"。玩家需要在有限资源下，通过放置"替罪羊"来转移危险、解锁路径。

## 技术架构

### 状态管理层

采用分层状态机管理游戏流程：

```
GameManager
├── ExplorationState    (探索阶段)
├── PuzzleState         (解谜阶段)
└── TransitionState     (过渡动画)
```

### 交互系统

所有可交互对象实现统一接口：

```csharp
public interface IPuzzleInteractable
{
    void OnInteract(PlayerController player);
    bool CanInteract(PlayerController player);
    InteractionType GetInteractionType();
}
```

### 关卡数据驱动

关卡配置使用 ScriptableObject，支持运行时热重载，加速迭代。

## 遇到的问题

物理交互层与 Tilemap 碰撞体的 Z-fighting 问题，通过自定义 Sorting Layer + 编辑器工具批量修复。

## 下一步

实现"替罪羊"实体的 AI 寻路与危险感知系统。
