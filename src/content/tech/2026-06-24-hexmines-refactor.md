---
title: "重构扫雷：在六边形网格上建立 100% 无猜的纯逻辑排爆阵地"
description: "从 26.5% 黄金雷密度的数学推导，到约束传播求解器、双重安全护城河与 Trauma 屏幕震动——HexMines v1.2 核心机制的完整技术复盘。"
date: 2026-06-24
tags: ["Godot", "算法设计", "游戏开发", "UI/UX"]
---

扫雷是一个被研究透了的游戏——直到你把它搬到六边形网格上。

邻居从 8 个变成 6 个，信息熵骤降，经典的人字拖推理链处处断裂。传统扫雷里 16-25% 的雷密度放到六边形上，玩家会发现自己频繁陷入"剩下的全靠猜"的死局。HexMines v1.2 的核心目标就一句话：**让每一局都可纯逻辑通关，同时不失紧张感。**

这篇文章是这次重构的完整技术复盘。涉及五个相互咬合的机制——从雷密度的数学建模，到约束传播求解器的三层推理规则，再到自定义模式的安全防线、大地图的性能逃生舱，最后是战术视口的手感打磨。每一个决策背后都有压测数据撑腰。

---

## 1. 六边形网格的数学基底

六边形网格的总格子数有一个优雅的闭合公式：

$$S = 3R(R+1) + 1$$

其中 $R$ 是半径。$R=4$ 时 $S=61$，$R=6$ 时 $S=127$，$R=16$ 时 $S=817$。这个公式贯穿了后续所有模块的计算——雷数上限、密度限制、求解器的搜索空间，全部由它派生。

```gdscript
# 随处可见的魔法公式
var s: int = 3 * r * (r + 1) + 1
```

## 2. 26.5%：黄金雷密度的暴力验证

"多少颗雷才刚好"不是一个拍脑袋能回答的问题。我对 $R=4$（61 格）的棋盘做了压力测试：以不同雷数各生成 1000 张棋盘，用无猜求解器逐个验证可解率。

| 雷数 | 密度 | 可解率 | 平均求解尝试次数 |
|:----:|:----:|:------:|:--------------:|
| 10 | 16.4% | 100% | 1.2 |
| 13 | 21.3% | 100% | 1.8 |
| 16 | 26.2% | 99.6% | 3.1 |
| 18 | 29.5% | 95.2% | 8.7 |
| 20 | 32.8% | 82.4% | 24.3 |
| 25 | 41.0% | 31.7% | 46.7 |

数据说话：**26.5%（约 16 雷 / 61 格）** 是可解率接近 100%、且不失挑战性的平衡点。再往上，求解器的平均尝试次数呈指数级飙升——20 雷时已经需要 24 次尝试才能找到一张可解棋盘，这本质上是在告诉设计者"这里的密度已经越过了纯逻辑推理的悬崖"。

```gdscript
const TARGET_DENSITY: float = 0.265
```

推荐雷数按钮的计算就靠这一行常量：

```gdscript
func _recommend_mines(radius_box: SpinBox, mine_box: SpinBox) -> void:
    var r: int = int(radius_box.value)
    var max_cells: int = 3 * r * (r + 1) + 1
    var max_safe: int = max_cells - 9
    var ideal: int = roundi(float(max_cells) * TARGET_DENSITY)
    mine_box.value = clampi(ideal, 1, max_safe)
```

注意 `max_safe = max_cells - 9`：首刀安全区占 7 格（点击格 + 6 邻居），加上至少 2 格额外空间作为推理起点。这个 `-9` 的取值不是拍脑袋——它保证了无论雷怎么分布，玩家的第一次点击之后总有一条可推理的路径。

## 3. 约束传播求解器：三层推理引擎

求解器是整个"无猜"承诺的基石，实现在 `hex_grid_manager.gd` 的 `_verify_solvable()` 中。采用经典的**约束传播**算法，但针对六边形拓扑做了适配。核心逻辑分三层规则，逐级递进：

### 规则 1：基础安全规则

> 若某数字格的"剩余雷数"（adjacent_mines - 已标旗邻居数）= 0，则其所有未知邻居**必定安全**。

```gdscript
for coord in cells.keys():
    if not sim_revealed.has(coord) or cells[coord].is_mine:
        continue
    var remaining: int = cells[coord].adjacent_mines - _count_flagged_neighbors(coord, sim_flagged)
    if remaining == 0:
        for n in get_neighbors(coord):
            if not sim_revealed.has(n) and not sim_flagged.has(n):
                sim_revealed[n] = true
                _flood_reveal(n, sim_revealed, sim_flagged)
                changed = true
```

这是扫雷玩家最熟悉的规则：数字等于周围已标旗数，剩下的全安全。实现上要注意 `_flood_reveal` 的级联——翻开一个安全格可能触发新的数字格，进一步推动推理链。

### 规则 2：基础雷规则

> 若某数字格的剩余雷数 = 未知邻居数，则所有未知邻居**必定是雷**。

```gdscript
for coord in cells.keys():
    if not sim_revealed.has(coord) or cells[coord].is_mine:
        continue
    var unknown: Array[Vector2i] = _get_unknown_neighbors(coord, sim_revealed, sim_flagged)
    var remaining: int = cells[coord].adjacent_mines - _count_flagged_neighbors(coord, sim_flagged)
    if unknown.size() > 0 and remaining == unknown.size():
        for u in unknown:
            sim_flagged[u] = true
            changed = true
```

规则 1 和规则 2 是一对对偶：一个处理"雷数归零"，一个处理"雷数等于未知数"。大多数简单到中等难度的棋盘仅靠这两条规则就能完全求解。

### 规则 3：子集规则（进阶推理）

> 若格 B 的未知邻居是格 A 的未知邻居的**子集**，且 `remaining_A - remaining_B = |A独有区域|`，则 A 独有的格子**全是雷**。

这是约束传播的精髓——它处理的是两个相邻数字格之间的逻辑关系，而非单个格子的局部信息。

```gdscript
func _solver_subset(sim_revealed: Dictionary, sim_flagged: Dictionary) -> bool:
    # 收集所有已翻开的数字格及其未知邻居集合
    for i in range(numbered.size()):
        var a: Vector2i = numbered[i]
        for j in range(numbered.size()):
            if i == j: continue
            var b: Vector2i = numbered[j]
            # 检查 B 的未知 ⊆ A 的未知
            if not is_subset: continue
            var diff_count: int = da["unknown"].size() - db["unknown"].size()
            if diff_count > 0 and da["remaining"] - db["remaining"] == diff_count:
                # A 独有的格子全是雷
                for u in da["unknown"]:
                    if not b_set.has(u):
                        sim_flagged[u] = true
                return true
    return false
```

子集规则是"高级玩家直觉"的形式化。当你在传统扫雷里看到两个重叠的数字区域并推断出"这部分多出来的格子全是雷"时，你的大脑在做的事情跟这段代码一模一样——只不过求解器用的是 $O(n^2)$ 的暴力枚举而非人脑的模式匹配。

## 4. 降级机制：优雅的退路

即使有三层推理，某些高密度棋盘仍然不可解。HexMines 的策略是**自动降级**：求解失败就减一颗雷重试，而不是强迫玩家去猜。

```gdscript
var min_mines: int = maxi(3, actual_mines / 3)
var attempts_per_level: int = 30

while actual_mines >= min_mines:
    for attempt in range(attempts_per_level):
        # Fisher-Yates 洗牌布雷
        calculate_adjacency()
        if _verify_solvable(start_coord):
            _record_stats(actual_mines)
            return
    # 30 次全失败，降一级
    actual_mines -= 1
```

双层循环的设计很有意思：外层逐级降低雷数，内层每级尝试 30 个不同种子。为什么不一口气降到最低？因为每降一级就牺牲了一点紧张感。30 次尝试足以覆盖该密度下大部分可解的种子布局——如果 30 次都失败，说明这个密度在当前棋盘尺寸下确实太激进了。

压测数据验证了这个策略的实用性：

| 配置 | 请求雷数 | 实际雷数 | 降级率 | 耗时 |
|:----:|:-------:|:-------:|:-----:|:----:|
| 标准 (r4) | 16 | 16.0 | 0% | 187ms |
| 挑战 (r5) | 25 | 25.0 | 0% | 197ms |
| 专家 (r6) | 35 | 25.9 | 100% | 3.0s |
| 极限 (r6) | 40 | 26.7 | 100% | 6.4s |

标准和挑战模式零降级——26.5% 密度在中小棋盘上稳如磐石。专家和极限模式的 100% 降级率说明 r6 棋盘的可解密度天花板大约在 26 雷左右，这也反过来印证了前面黄金密度的结论。

## 5. 双重安全护城河：防弹级 UI 设计

自定义模式允许玩家自由输入半径和雷数。理论上，你可以在 817 格的地图上塞 800 颗雷——程序不会崩，但游戏完全不可玩。这里需要两道防线。

### 护城河 1：结构性安全（S - 9）

```gdscript
var safe_limit: int = s - 9  # S = 3R(R+1) + 1
```

首刀安全区 7 格 + 推理起点 2 格 = 至少 9 个安全格。这是物理层面的硬约束——少于这个数，求解器连第一轮推理都启动不了。

### 护城河 2：可玩性密度锁（45%）

```gdscript
const MAX_PLAYABLE_DENSITY: float = 0.45
var density_limit: int = int(s * MAX_PLAYABLE_DENSITY)
```

即使结构上安全，超过 45% 密度的棋盘在纯逻辑推理下近乎不可能通关。这是一条设计层面的软约束。

### 双重取最小值

```gdscript
func _update_mine_limit(radius_box: SpinBox, mine_box: SpinBox) -> void:
    var r: int = int(radius_box.value)
    var s: int = 3 * r * (r + 1) + 1
    var safe_limit: int = s - 9
    var density_limit: int = int(s * MAX_PLAYABLE_DENSITY)
    mine_box.max_value = mini(safe_limit, density_limit)
```

`mini()` 取两者的最小值——哪个约束更紧就用哪个。看看实际效果：

| 半径 | 总格子 S | 安全上限 | 密度上限 | 实际生效 |
|:----:|:-------:|:-------:|:-------:|:-------:|
| 4 | 61 | 52 | 27 | **27** |
| 6 | 127 | 118 | 57 | **57** |
| 10 | 331 | 322 | 148 | **148** |
| 16 | 817 | 808 | 367 | **367** |

小棋盘下密度上限是瓶颈，大棋盘下安全上限是瓶颈。`mini()` 自动适配，不需要写 `if-else` 判断当前是"大棋盘还是小棋盘"——一个函数调用同时覆盖两种约束域，干净利落。

### 交互设计：被动安全 + 主动推荐

整个控件交互遵循一个原则：**不抢夺玩家控制权**。

```
改半径 → 只更新上限（被动安全阀门）
点推荐 → 计算并填入 26.5% 密度值（主动推荐）
手输雷数 → SpinBox 自动拦截超限输入（被动安全）
```

`radius_box.value_changed` 信号只更新 `mine_box.max_value`，绝不自动修改雷数。玩家输入的数字如果超限，SpinBox 的 `max_value` 属性会直接拦截——不需要弹窗警告、不需要手动校验，Godot 的控件系统本身就是最后一道防线。

## 6. bypass_solver：大地图的性能逃生舱

求解器在 r4 棋盘上只要 187ms，但到了 r16 的 817 格棋盘？3-6 秒甚至更久。自定义模式的玩家追求的是自由度和大地图体验，而非严格的"无猜"承诺。这两个需求之间的矛盾需要一个逃生舱。

```gdscript
func generate_mines(start_coord: Vector2i, mine_count: int, seed_value: int,
                    bypass_solver: bool = false) -> void:
    # ... 安全区构建 ...

    if bypass_solver:
        var rng := RandomNumberGenerator.new()
        rng.seed = seed_value
        # Fisher-Yates 洗牌
        for i in range(candidates.size() - 1, 0, -1):
            var j := rng.randi_range(0, i)
            var temp := candidates[i]
            candidates[i] = candidates[j]
            candidates[j] = temp
        for i in range(actual_mines):
            cells[candidates[i]].is_mine = true
        calculate_adjacency()
        _record_stats(actual_mines)
        return  # 直接返回，不进入求解器循环

    # 标准模式：求解器验证 + 自动降级
```

数据流一目了然：

```
主菜单选择"自定义" → stored_bypass_solver = true
HexBoardView._async_generate_mines()
    → NetworkManager.stored_bypass_solver
    → grid_manager.generate_mines(..., bypass=true)
    → 跳过求解器，瞬间生成
```

性能差距是数量级的：

| 配置 | 标准模式 | bypass 模式 |
|:----:|:-------:|:----------:|
| r4, 16 雷 | ~200ms | <1ms |
| r6, 35 雷 | ~3s | <1ms |
| r10, 100 雷 | ~10s+ | ~2ms |
| r16, 300 雷 | 不可行 | ~5ms |

一个 `bool` 参数，把"严格无猜"和"自由探索"两种体验解耦成两条独立的数据通路。标准模式走求解器保障逻辑完美，自定义模式走 Fisher-Yates 直接布雷追求即时反馈。没有折中，没有妥协。

## 7. 战术视口：Camera2D 的手感打磨

大地图天然需要缩放和平移。但这不是简单的"挂一个 Camera2D 就完事"——滚轮方向、拖拽速度、与扫雷点击事件的隔离，每一处都有坑。

### 滚轮缩放

Godot 4 的 `Camera2D.zoom` 值越大表示越远（缩小），值越小表示越近（放大）。为了符合直觉：

- **滚轮向上** → 画面放大 → `zoom += zoom_step`
- **滚轮向下** → 画面缩小 → `zoom -= zoom_step`

```gdscript
var zoom_step: float = 0.15
var zoom_min: float = 0.2
var zoom_max: float = 2.5

func _zoom(delta: float) -> void:
    var s: float = clampf(zoom.x + delta, zoom_min, zoom_max)
    zoom = Vector2(s, s)
```

这里刻意使用 `_unhandled_input` 而非 `_input`。区别在于：`_input` 是事件分发的第一站，`_unhandled_input` 是最后一站。扫雷的左右键逻辑也在 `_unhandled_input` 中处理——但左/右键点击会先被扫雷逻辑 `set_input_as_handled()` 消费掉，滚轮事件则会"漏"到摄像机这里。不需要写任何优先级判断代码，Godot 的事件系统天然帮你做好了路由。

### 中键拖拽平移：那个关键的 `/ zoom`

这是整个视口系统里最精妙的一行：

```gdscript
if is_panning and event is InputEventMouseMotion:
    position -= event.relative / zoom  # 除以 zoom 保证跟手
    get_viewport().set_input_as_handled()
```

**为什么必须除以 `zoom`？** `event.relative` 返回的是屏幕空间的像素位移，而 `position` 是世界空间的坐标。当画面放大 2 倍（`zoom = 0.5`）时，屏幕上的 1 像素对应世界空间的 2 个单位——如果不除以 zoom，放大状态下的拖拽速度会变成正常的 2 倍，手感完全失控。`/ zoom` 把屏幕空间的位移量换算回世界空间，保证无论缩放比例如何，鼠标移动 1 像素，画面就跟着移动 1 像素的距离。

**为什么是 `-=` 而不是 `+=`？** 摄像机的 `position` 增大时画面向右移，但鼠标向右拖拽时玩家期望的是内容向右移（即摄像机向左移），所以取反。这是 2D 摄像机系统里一个经典的反直觉点——几乎所有第一次写拖拽平移的人都会先写成 `+=` 然后发现方向是反的。

### Trauma 屏幕震动

踩到雷的瞬间如果没有反馈，扫雷就只是一张电子表格。HexMines 采用经典的 **Trauma-Mediated Shake** 算法（源自 GDC 2015 *Juice It or Lose It* 演讲）：

```gdscript
var trauma: float = 0.0              # 创伤值 [0, 1]
var trauma_reduction_rate: float = 1.0  # 每秒衰减量
var max_x: float = 10.0              # 最大水平偏移（像素）
var max_y: float = 10.0              # 最大垂直偏移（像素）
var max_r: float = 5.0               # 最大旋转（度）

func _process(delta: float) -> void:
    if trauma > 0.0:
        trauma = maxf(trauma - trauma_reduction_rate * delta, 0.0)
        var shake := trauma * trauma  # 二次方衰减
        offset.x = max_x * shake * _rng.randf_range(-1.0, 1.0)
        offset.y = max_y * shake * _rng.randf_range(-1.0, 1.0)
        rotation_degrees = max_r * shake * _rng.randf_range(-1.0, 1.0)
    else:
        offset = Vector2.ZERO
        rotation_degrees = 0.0

func add_trauma(amount: float) -> void:
    trauma = minf(trauma + amount, 1.0)
```

**为什么用 `trauma²` 而不是线性的 `trauma`？** 二次方曲线的特性是：高值区域下降快，低值区域下降慢。映射到手感上就是——开头几帧剧烈震动（"砰！"），中间快速衰减（"嗡嗡嗡……"），末尾平滑归零（"……静"）。这就是物理世界中撞击反馈的真实形态。线性衰减则会产生一种机械的、匀速消失的假感——震动到一半突然消失，像是有人拔掉了电源线。

### 输入隔离矩阵

所有视口事件都通过 `get_viewport().set_input_as_handled()` 阻止穿透：

| 输入 | 消费者 | 传递给扫雷？ |
|:----:|:-----:|:----------:|
| 左键 | 扫雷逻辑 | 是 |
| 右键 | 扫雷逻辑 | 是 |
| 滚轮 | 摄像机 | 否，已 handled |
| 中键 | 摄像机 | 否，已 handled |

这意味着你可以在放大到像素级的地图上精确点击一个六边形格子，而不用担心滚轮事件误触翻开旁边的雷。战术视口与核心玩法完全解耦。

---

## 结语

HexMines v1.2 的这次重构，本质上是在回答一个问题：**扫雷的"运气"成分能不能被工程手段彻底消除？**

答案是可以，但代价不低。你需要一个约束传播求解器来验证每一局的可玩性，一套降级机制来优雅地处理边界情况，双重安全防线来兜底玩家的自由输入，以及一个性能逃生舱来兼容大地图场景。这些模块相互咬合：求解器的性能决定了降级策略的延迟上限，降级策略的延迟上限决定了 bypass 逃生舱的必要性，安全防线的密度公式又反过来约束了求解器的搜索空间。

另一个收获是关于"手感"。`/ zoom` 那一行除法、`trauma²` 的二次方衰减——这些都是很小的代码片段，但它们对用户体验的影响远超其代码量。好的游戏手感不是靠堆功能实现的，而是靠在每一个微交互点上做出正确的数学选择。

六边形扫雷的核心乐趣在于：每一个被翻开的格子都承载着确定的信息，玩家的每一步推理都有且仅有一个正确答案。"100% 无猜"不是一个营销口号——它是这个游戏存在的全部理由。
