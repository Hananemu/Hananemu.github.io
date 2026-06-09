---
title: "《归序残章》Devlog #01：如何优雅地管理局内卡组，实现全局状态与战斗解耦"
description: "通过 RunManager 实现单局内卡组数据的跨场景管理，彻底解耦战斗模块与局外进度，建立单向数据流架构。"
date: 2026-06-09
series: "归序残章技术拆解"
tags: ["游戏开发", "Unity", "C#", "架构设计"]
---

在开发《归序残章》这样带有肉鸽元素的卡牌游戏时，如何管理数据是一道绕不开的坎。尤其是在单局游戏（Run）的循环中，玩家会在地图、战斗、事件和奖励结算面板之间频繁跳转。

如果状态管理没做好，往往会遇到以下情况：卡组数据在场景切换时丢失，或是战斗模块与局外数据强行绑定，导致加一张奖励卡都要"牵一发而动全身"。本期开发日志，我将复盘近期的一次核心重构，聊聊如何通过建立一个纯粹的数据中枢，彻底剥离原有的高耦合依赖。

### 1. 痛点：被绑架的战斗模块

在早期架构中，战斗系统（`BattleManager`）和结算面板（`CardAssignmentPanel`）严重依赖于一个叫做 `PlayerDeck`` 的类。这意味着：

- 战斗开始前，战斗管理器必须主动去寻找 `PlayerDeck` 抓取牌库。
- 战斗胜利后，结算面板又得直接修改 `PlayerDeck`。

随着多英雄同场机制的引入，这种耦合带来的问题被无限放大。我们需要一个全局的、跨场景的管理器，专门负责单局内的数据流转，让战斗只管战斗，结算只管结算。

### 2. 破局：架构重组与单向数据流

为了解决这个问题，我新建了一个跨场景常驻的全局节点 `RunManager`。它只干一件事：作为关卡单轮数据管理器，维护当前出战的英雄 ID 和对应卡组，并为其他模块提供干净的读写接口。

重构后的数据流向变得异常清晰：

游戏启动 → GameFlowManager.Start()
           └─→ RunManager.StartNewRun(chosenHeroes) 初始化各自卡组（如 15 张）

战斗开始 → BattleManager.OnBattleStart()
           └─→ RunManager.GetMergedRunDeck() 获取合并卡组（如 45 张）并分发

战斗胜利 → 选牌面板 (CardAssignmentPanel)
           ├─→ 读取 RunManager.ActiveHeroIDs (只显示出战英雄)
           └─→ 确认 → RunManager.AddRewardCard(heroID, card) 写入新卡

下一战   → 再次 GetMergedRunDeck() 获得更新后的卡组（46 张）

关卡结束 → RunManager.EndRun()
           └─→ 清空所有临时数据，奖励卡不带回局外

### 3. 核心代码实现

在重构过程中，核心的改动在于切断旧的依赖。我们先看看核心的 `RunManager` 是如何工作的：

```csharp
// RunManager.cs - 挂载于场景根级，负责跨场景生命周期
public class RunManager : MonoBehaviour
{
    public static RunManager Instance { get; private set; }

    // 核心数据结构
    private List<string> _activeHeroIDs = new List<string>();
    private Dictionary<string, List<CardData>> _runHeroDecks = new Dictionary<string, List<CardData>>();

    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }

    // 启动新的一局
    public void StartNewRun(List<HeroData> chosenHeroes)
    {
        _activeHeroIDs.Clear();
        _runHeroDecks.Clear();

        foreach (var hero in chosenHeroes)
        {
            _activeHeroIDs.Add(hero.HeroID);
            _runHeroDecks[hero.HeroID] = new List<CardData>(hero.BaseDeck);
        }
    }

    // 为战斗模块提供干净的只读卡组池
    public List<CardData> GetMergedRunDeck()
    {
        List<CardData> merged = new List<CardData>();
        foreach (var deck in _runHeroDecks.Values)
        {
            merged.AddRange(deck);
        }
        return merged;
    }

    // 奖励结算接口
    public void AddRewardCard(string heroID, CardData newCard)
    {
        if (_runHeroDecks.ContainsKey(heroID))
        {
            _runHeroDecks[heroID].Add(newCard);
        }
    }

    // 单局结束，清理内存
    public void EndRun()
    {
        _activeHeroIDs.Clear();
        _runHeroDecks.Clear();
    }
}
```

随之而来的是极其舒爽的依赖剔除。BattleManager 和 CardAssignmentPanel 不再需要知道 PlayerDeck 是什么。

在 GameFlowManager 中，我们只需在初始阶段传入出战英雄即可：

```csharp
// GameFlowManager.cs
public List<HeroData> chosenHeroes; // 在 Inspector 中拖入英雄数据

private void Start()
{
    // 启动数据流的第一步
    RunManager.Instance.StartNewRun(chosenHeroes);
}
```

### 4. 总结与反思

通过 RunManager 的引入，我们彻底解耦了战斗场景与局外进度。PlayerDeck 现在的职责得以大幅度收束（仅用于局外永久奖励卡池和资源管理），再也不用去趟局内战斗的浑水。

这种数据驱动的设计，让后续不管是添加新的卡牌结算逻辑，还是扩展多角色机制，都能做到只调接口，不碰底层，为《归序残章》后续更复杂的系统开发打下了很好的地基。
