# 补丁004V2 设计文档（designV4）

> 基于 `补丁004V2全面分析报告.md` 的 6 大问题（P1~P6）
> 在 designV3（模块 H/I/J/K）基础上新增模块 L（增量差集计算）
> 前置条件：V1 代码修改已完成（design.md 模块 A-G）
> 创建时间：2026-02-21

---

## 一、问题总结与修复目标

| 编号 | 问题 | 修复模块 | 严重程度 |
|------|------|---------|---------|
| P1 | ClearActionQueue 清空执行状态 → 动画播完不创建耕地 + 执行预览残留 | 模块 H | 🔴 严重 |
| P2 | PromoteToExecutingPreview 时机过早（出队时而非动画开始时） | 模块 I | 🔴 严重 |
| P3 | Ghost 预览显示最终 tile 而非增量 tile（已存在部分被多余染绿） | 模块 L | 🔴 严重 |
| P4 | canTill=false 时无任何视觉反馈 | 模块 J | 🟡 中等 |
| P5 | 批量操作时队列预览之间 tile 互相覆盖 | 已知限制 | 🟡 不修复 |
| P6 | Sorting Order 需确认 | 模块 K | 🟢 验证性 |

设计约束：
- 不修改 V1 已完成的模块 A-G 核心架构
- 只在 V1 基础上做增量修复
- 新增并兼容，不破坏已有逻辑

---

## 二、模块 H：ClearActionQueue 执行状态保护

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`ClearActionQueue`（第2531行）

### 2.1 现状问题

当前 `ClearActionQueue` 无条件清空所有状态，包括 `_pendingTileUpdate`、`_currentProcessingRequest`、`_isExecutingFarming`。WASD 中断时动画仍在播放，但这些状态被清空导致：
- `_pendingTileUpdate = null` → Update 中进度监听条件不满足 → tile 创建永远不触发
- `_currentProcessingRequest = default` → `RemoveExecutingPreview((0,0,0))` → 执行预览残留
- `_isExecutingFarming = false` → 保护条件失效

### 2.2 修复方案

在 `ClearActionQueue` 中检查 `_isExecutingFarming`：

```csharp
public void ClearActionQueue()
{
    _farmActionQueue.Clear();
    _queuedPositions.Clear();
    _isProcessingQueue = false;
    
    if (!_isExecutingFarming)
    {
        _currentProcessingRequest = default;
        _pendingTileUpdate = null;
        _tileUpdateTriggered = false;
        _isExecutingFarming = false;
        _currentHarvestTarget = null;
    }
    // else: 动画执行中，保留执行状态，动画完成后由回调正常清理
    
    FarmToolPreview.Instance?.ClearAllQueuePreviews();
}
```

### 2.3 正确性属性

| 编号 | 属性 |
|------|------|
| CP-H1 | WASD 中断时，`_isExecutingFarming` 为 true 时保留 `_pendingTileUpdate`/`_currentProcessingRequest` |
| CP-H2 | 动画播完后 tile 正常创建（`_pendingTileUpdate` 未被清空） |
| CP-H3 | 动画完成后执行预览正常清除（`_currentProcessingRequest` 保留正确的 cellPos） |
| CP-H4 | 队列中等待的操作仍被正常清空（`_farmActionQueue.Clear()` 和 `ClearAllQueuePreviews()` 正常执行） |


---

## 三、模块 I：PromoteToExecutingPreview 时机修正

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`ProcessNextAction`（第2319行）、`ExecuteFarmAction`（第2425行）

### 3.1 现状问题

当前 `ProcessNextAction` 第2380-2383行：
```csharp
_isExecutingFarming = true;
FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
```
在出队后立即执行。远距离操作时导航途中预览已被提升为执行预览，WASD 打断导航后执行预览被保护不被清除，但操作不会执行 → 预览永远残留。

用户核心观点：**执行 = 动画开始的瞬间，导航途中只是前置行为**。

### 3.2 修复方案

从 `ProcessNextAction` 删除这两行，移到 `ExecuteFarmAction` 开头。

ProcessNextAction 中删除：
```csharp
// 删除：_isExecutingFarming = true;
// 删除：FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
```

ExecuteFarmAction 开头新增：
```csharp
private void ExecuteFarmAction(FarmActionRequest request)
{
    _isExecutingFarming = true;
    FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
    // ... 原有 switch 分支 ...
}
```

### 3.3 P1+P2 联动效果

P2 修正后 `_isExecutingFarming` 只在 `ExecuteFarmAction` 中设为 true：
- 导航途中 WASD → `ClearActionQueue` → `_isExecutingFarming == false` → 全部清空 ✅
- 动画执行中 WASD → `ClearActionQueue` → `_isExecutingFarming == true` → P1 保护生效 ✅

### 3.4 PlantSeed 分支

PlantSeed 无动画，`ExecuteFarmAction` 开头 Promote → PlantSeed 分支直接执行 → `RemoveExecutingPreview` → `_isExecutingFarming = false`。同一帧内完成，不存在 WASD 中断窗口。

### 3.5 正确性属性

| 编号 | 属性 |
|------|------|
| CP-I1 | 执行预览只在 `ExecuteFarmAction` 中才被保护 |
| CP-I2 | 导航途中被打断的操作随队列一起清空 |
| CP-I3 | `_isExecutingFarming` 只在动画开始时设为 true |
| CP-I4 | 近距离操作行为不变（ProcessNextAction 直接调用 ExecuteFarmAction） |

---

## 四、模块 L：Ghost 预览增量差集计算（核心新增）

> 涉及文件：`FarmlandBorderManager.cs`、`FarmToolPreview.cs`
> 涉及方法：新增 `ParseDirections`/`IsBorderTile`/`IsShadowTile`，修改 `UpdateHoePreview`，`SelectBorderTile` 改为 public

### 4.1 核心认知：tile 可拆分

项目有完整的 16 种边界 tile + 4 种阴影 tile，每个方向组合都有独立 Sprite 资源。`B_LR` = `B_L` + `B_R` 的合成，可以拆开。

用户核心原则：**预览 + 实际 = 最终效果**。预览只显示增量，不与实际层内容重叠。

### 4.2 已识别的增量计算错误（来自交互矩阵V2）

| 场景 | 错误位置 | 实际 tile | 预览最终 | 当前 ghost（错误） | 正确 ghost | 差集 |
|------|---------|----------|---------|-----------|-----------|------|
| 水平直接相邻 | (-1,-1) | `B_U` | `B_UR` | `B_UR` | `B_R` | `{U,R}-{U}={R}` |
| 水平隔一格 | M=(0,0) | `B_L` | `B_LR` | `B_LR` | `B_R` | `{L,R}-{L}={R}` |
| 斜角直接相邻 | (-1,0) | `B_D` | `B_DR` | `B_DR` | `B_R` | `{D,R}-{D}={R}` |
| 斜角直接相邻 | (0,-1) | `B_L` | `B_UL` | `B_UL` | `B_U` | `{U,L}-{L}={U}` |

### 4.3 新增方法：FarmlandBorderManager

#### 4.3.1 ParseDirections

将 tile 引用解析为方向四元组 `(hasU, hasD, hasL, hasR)`。逐一比较 tile 引用与 16 个边界 tile 字段。

```csharp
public (bool hasU, bool hasD, bool hasL, bool hasR) ParseDirections(TileBase tile)
{
    if (tile == null) return (false, false, false, false);
    // 单方向
    if (tile == borderU) return (true, false, false, false);
    if (tile == borderD) return (false, true, false, false);
    if (tile == borderL) return (false, false, true, false);
    if (tile == borderR) return (false, false, false, true);
    // 双方向对边
    if (tile == borderUD) return (true, true, false, false);
    if (tile == borderLR) return (false, false, true, true);
    // 双方向相邻
    if (tile == borderUL) return (true, false, true, false);
    if (tile == borderUR) return (true, false, false, true);
    if (tile == borderDL) return (false, true, true, false);
    if (tile == borderDR) return (false, true, false, true);
    // 三方向
    if (tile == borderUDL) return (true, true, true, false);
    if (tile == borderUDR) return (true, true, false, true);
    if (tile == borderULR) return (true, false, true, true);
    if (tile == borderDLR) return (false, true, true, true);
    // 四方向
    if (tile == borderUDLR) return (true, true, true, true);
    return (false, false, false, false);
}
```

#### 4.3.2 IsBorderTile / IsShadowTile

```csharp
public bool IsBorderTile(TileBase tile)
{
    if (tile == null) return false;
    var dirs = ParseDirections(tile);
    return dirs.hasU || dirs.hasD || dirs.hasL || dirs.hasR;
}

public bool IsShadowTile(TileBase tile)
{
    return tile == shadowLU || tile == shadowRU || tile == shadowLD || tile == shadowRD;
}
```

#### 4.3.3 SelectBorderTile 改为 public

当前是 `private`，增量计算需要从差集方向生成增量 tile，改为 `public`。

### 4.4 UpdateHoePreview 差异化过滤改造

当前逻辑（第487-501行）：
```csharp
if (kvp.Value == actualTile) continue;  // 相同跳过
ghostTilemap.SetTile(kvp.Key, kvp.Value);  // 不同则显示预览tile（最终状态）
```

改造为增量差集计算：
```
foreach (kvp in previewTiles):
    if kvp.Value == null: continue
    actualTile = GetActualTile(kvp.Key)
    if kvp.Value == actualTile: continue  // 无差异
    
    if actualTile == null:
        // 全新位置，直接显示预览 tile
        ghost.SetTile(kvp.Key, kvp.Value)
    elif borderManager.IsShadowTile(actualTile):
        // 阴影→边界，完全替换（Sorting Order 覆盖）
        ghost.SetTile(kvp.Key, kvp.Value)
    elif borderManager.IsBorderTile(actualTile) && borderManager.IsBorderTile(kvp.Value):
        // 边界→边界，计算增量差集
        var actualDirs = borderManager.ParseDirections(actualTile)
        var previewDirs = borderManager.ParseDirections(kvp.Value)
        bool deltaU = previewDirs.hasU && !actualDirs.hasU
        bool deltaD = previewDirs.hasD && !actualDirs.hasD
        bool deltaL = previewDirs.hasL && !actualDirs.hasL
        bool deltaR = previewDirs.hasR && !actualDirs.hasR
        var deltaTile = borderManager.SelectBorderTile(deltaU, deltaD, deltaL, deltaR)
        if deltaTile != null:
            ghost.SetTile(kvp.Key, deltaTile)
    else:
        // 其他情况（中心块替换等），直接覆盖
        ghost.SetTile(kvp.Key, kvp.Value)
```

### 4.5 正确性属性

| 编号 | 属性 |
|------|------|
| CP-L1 | 实际层为空时，ghost 显示完整预览 tile |
| CP-L2 | 两者都是边界 tile 时，ghost 只显示方向差集的增量 tile |
| CP-L3 | 阴影 tile 替换时，ghost 显示预览 tile 覆盖实际层 |
| CP-L4 | 中心块替换时，ghost 显示中心块覆盖实际层 |
| CP-L5 | `ParseDirections` 正确解析所有 16 种边界 tile |
| CP-L6 | 增量方向为空集时不在 ghost 层放置 tile |


---

## 五、模块 J：canTill=false 红色反馈

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`（第403行）

### 5.1 现状问题

`canTill = false` 时：
- 不进入 `if (canTill && ...)` 分支 → ghostTilemap 无 tile
- `cursorRenderer.enabled = false`（耕地模式不显示光标）
- shader 叠加色设置了红色但无载体 → 什么都不显示

### 5.2 修复方案

在 `else` 分支中启用 `cursorRenderer` 并调用 `UpdateCursor`：

```csharp
else
{
    _currentGhostTileData?.Clear();
    
    // V2 模块J：canTill=false 时显示光标反馈
    if (cursorRenderer != null)
    {
        cursorRenderer.enabled = true;
        UpdateCursor(layerIndex, cellPos);
        // UpdateCursor 根据 currentState 设置颜色：Valid=绿色（枯萎作物），Invalid=红色（已有耕地）
    }
}
```

### 5.3 正确性属性

| 编号 | 属性 |
|------|------|
| CP-J1 | 鼠标在已有耕地上时显示红色方框（`canTill=false` + `!canClearWithered` → Invalid） |
| CP-J2 | 鼠标在枯萎作物上时显示绿色方框（`canClearWithered=true` → Valid） |
| CP-J3 | 可锄地时仍然不显示光标（`canTill=true` 分支中 `cursorRenderer.enabled = false` 不变） |

---

## 六、模块 K：Sorting Order 确认

> 涉及文件：`FarmToolPreview.cs`（EnsureComponents 方法）

### 6.1 代码事实

`EnsureComponents` 中：
- `ghostTilemapRenderer.sortingOrder = 9999`
- `queuePreviewTilemapRenderer.sortingOrder = 9998`
- `cursorRenderer.sortingOrder = 10000`

`farmlandBorderTilemap` 的 Sorting Order 由场景配置决定，通常远低于 9999。

### 6.2 结论

ghostTilemap（9999）> farmlandBorderTilemap → 覆盖关系成立。任务执行时实际确认即可。

### 6.3 正确性属性

| 编号 | 属性 |
|------|------|
| CP-K1 | ghostTilemap Sorting Order（9999）> farmlandBorderTilemap Sorting Order |

---

## 七、完整数据流（V2 修正版）

```
1. 鼠标移动 → UpdateHoePreview
   ├─ canTill=true：
   │   ├─ GetPreviewTiles → 增量差集过滤（模块 L） → ghostTilemap 显示增量 tile
   │   ├─ cursorRenderer.enabled = false
   │   └─ 缓存 CurrentGhostTileData（增量 tile，非最终 tile）
   └─ canTill=false：
       ├─ ghostTilemap 无 tile
       ├─ cursorRenderer.enabled = true → 红色/绿色方框（模块 J）
       └─ _currentGhostTileData.Clear()

2. 玩家点击 → TryEnqueueFarmTool → EnqueueAction
   ├─ 复制 CurrentGhostTileData 快照
   ├─ AddQueuePreview → queuePreviewTilemap 显示队列预览
   └─ ghost 继续跟随鼠标

3. ProcessNextAction 出队
   ├─ 不调用 PromoteToExecutingPreview（模块 I）
   ├─ 不设置 _isExecutingFarming = true（模块 I）
   ├─ 近距离 → 直接 ExecuteFarmAction
   └─ 远距离 → StartFarmingNavigation → 到达后 ExecuteFarmAction

4. ExecuteFarmAction（动画开始瞬间）
   ├─ _isExecutingFarming = true（模块 I）
   ├─ PromoteToExecutingPreview(cellPos)（模块 I）
   ├─ FaceTarget → RequestAction → 开始动画
   └─ _pendingTileUpdate = request

5. WASD 中断（导航途中）
   ├─ ClearActionQueue → _isExecutingFarming == false → 全部清空（模块 I+H 联动）
   └─ 操作还在队列预览中，被正常清空 ✅

6. WASD 中断（动画执行中）
   ├─ ClearActionQueue → _isExecutingFarming == true → 保留执行状态（模块 H）
   ├─ 动画继续播放 → tile 正常创建 ✅
   └─ OnFarmActionAnimationComplete → RemoveExecutingPreview(正确 cellPos) ✅

7. 动画正常完成
   ├─ TillAt → UpdateBorderAt（数据层落地）
   ├─ RemoveExecutingPreview(cellPos)
   └─ ProcessNextAction（取下一个）
```

---

## 八、正确性属性完整汇总

### V2 新增属性（18条）

| 编号 | 属性 | 模块 |
|------|------|------|
| CP-H1 | WASD 中断时正在执行的动画操作不受影响 | H |
| CP-H2 | 动画播完后 tile 正常创建 | H |
| CP-H3 | 动画完成后执行预览正常清除 | H |
| CP-H4 | 队列中等待的操作仍被正常清空 | H |
| CP-I1 | 执行预览只在 ExecuteFarmAction 中才被保护 | I |
| CP-I2 | 导航途中被打断的操作随队列一起清空 | I |
| CP-I3 | `_isExecutingFarming` 只在动画开始时设为 true | I |
| CP-I4 | 近距离操作行为不变 | I |
| CP-L1 | 实际层为空时 ghost 显示完整预览 tile | L |
| CP-L2 | 两者都是边界 tile 时 ghost 只显示方向差集增量 tile | L |
| CP-L3 | 阴影 tile 替换时 ghost 显示预览 tile 覆盖实际层 | L |
| CP-L4 | 中心块替换时 ghost 显示中心块覆盖实际层 | L |
| CP-L5 | ParseDirections 正确解析所有 16 种边界 tile | L |
| CP-L6 | 增量方向为空集时不在 ghost 层放置 tile | L |
| CP-J1 | 鼠标在已有耕地上时显示红色方框 | J |
| CP-J2 | 鼠标在枯萎作物上时显示绿色方框 | J |
| CP-J3 | 可锄地时仍然不显示光标 | J |
| CP-K1 | ghostTilemap Sorting Order > farmlandBorderTilemap | K |

### V1 保留属性（不变）

CP-A1~A2、CP-B1~B3、CP-C1~C4、CP-D1~D3、CP-E1~E3、CP-F1~F2、CP-G1~G6

---

## 九、涉及文件汇总

| 文件 | 改动类型 | 涉及模块 |
|------|----------|----------|
| `GameInputManager.cs` | 修改 | H（ClearActionQueue 保护）、I（Promote 时机修正） |
| `FarmToolPreview.cs` | 修改 | L（增量差集过滤）、J（canTill=false 反馈） |
| `FarmlandBorderManager.cs` | 修改 | L（新增 ParseDirections/IsBorderTile/IsShadowTile，SelectBorderTile 改 public） |

