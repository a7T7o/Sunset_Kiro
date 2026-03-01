# 补丁004 设计文档 V2 — 三层预览架构重构

> V2 按「修改目标（文件 × 方法/区域）」为主轴重组。每个模块自包含——现状代码摘要、所有改动、改动后伪代码、正确性属性。
> 来源：`补丁004全面分析报告.md`（R1-R8 八个修改点）
> 用户确认：异议认可（实际落地通过数据层驱动，不直接复制 tile）

---

## 目录

1. [全局架构：三层预览分离概览](#一全局架构三层预览分离概览)
2. [模块 A：FarmToolPreview — 移除 LockPosition 机制（R2）](#二模块-a)
3. [模块 B：FarmToolPreview.UpdateHoePreview — Ghost 差异化过滤（R1）](#三模块-b)
4. [模块 C：FarmToolPreview.UpdateWateringPreview — 进入新格子才随机（R3）](#四模块-c)
5. [模块 D：FarmToolPreview — 执行预览层数据结构与方法（R6/R8）](#五模块-d)
6. [模块 E：FarmToolPreview.ClearAllQueuePreviews — 边界残留修复（R5/R7）](#六模块-e)
7. [模块 F：FarmToolPreview.AddQueuePreview — 接收 Ghost 数据（R4）](#七模块-f)
8. [模块 G：GameInputManager — 移除 Lock/Unlock + 集成执行预览（R2/R6/R7/R8）](#八模块-g)
9. [完整数据流](#九完整数据流)
10. [正确性属性汇总](#十正确性属性汇总)
11. [涉及文件汇总](#十一涉及文件汇总)

---

## 一、全局架构：三层预览分离概览

### 1.1 当前架构问题

```
当前（有 bug）：
ghost 预览 ──→ 每帧 GetPreviewTiles 全量 1+8，不过滤已有耕地
              ──→ LockPosition 冻结 ghost，职责混淆
队列预览   ──→ AddQueuePreview 独立调用 GetPreviewTiles，与 ghost 无数据关系
              ──→ ClearAllQueuePreviews 只清中心块，边界残留
执行预览   ──→ 不存在，正在执行的预览留在队列中，WASD 中断时被一起清空
```

### 1.2 目标架构

```
┌─ 第一层：Ghost 预览（鼠标跟随层）─────────────────────┐
│ · 始终跟随鼠标，永不锁定（移除 LockPosition）          │
│ · 差异化显示：对比 tilledTilemap，只显示变化的 tile     │
│ · 浇水：进入新格子随机一次，不移出不变                  │
│ · 缓存 CurrentGhostTileData 供入队时复制               │
│ · Tilemap: ghostTilemap + FarmPreviewOverlay shader    │
├─ 第二层：队列预览（已入队等待执行）────────────────────┤
│ · 入队时复制 ghost 的 CurrentGhostTileData（不独立计算）│
│ · 静态显示，半透明，无 shader 叠加色                   │
│ · WASD 中断时清空（跳过执行预览）                      │
│ · Tilemap: queuePreviewTilemap                        │
├─ 第三层：执行预览（正在执行的操作）────────────────────┤
│ · ProcessNextAction 出队时从队列预览转移                │
│ · 受保护，WASD 中断不清除                              │
│ · 动画完成后清除（tile 已落地，视觉无缝）              │
│ · 复用 queuePreviewTilemap（通过 executingTileGroups 区分）│
└──────────────────────────────────────────────────────┘
         ↓ 实际落地
   TillAt → UpdateBorderAt（数据层驱动视觉）
```

---

## 二、模块 A：FarmToolPreview — 移除 LockPosition 机制（R2）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法/字段：`LockPosition`、`UnlockPosition`、`_isLocked` 等

### 2.1 现状代码摘要

```csharp
// 字段（约第 163-166 行）
private bool _isLocked = false;
private Vector3 _lockedWorldPosition;
private Vector3Int _lockedCellPos;
private int _lockedLayerIndex;

// LockPosition（约第 392 行）：锁定 ghost 到指定位置
// - 设置 _isLocked = true
// - 锄头模式下执行一次 GhostTilemap 渲染（补丁002 P3 修复）
// - 调用 UpdateCursor

// UnlockPosition（约第 443 行）：解锁 ghost
// - 设置 _isLocked = false

// UpdateHoePreview（约第 510 行）：
// if (_isLocked) return;  ← 锁定时跳过视觉更新

// UpdateWateringPreview（约第 619 行）：
// if (_isLocked) return;  ← 同上
```

### 2.2 改动

1. 删除 4 个锁定字段：`_isLocked`、`_lockedWorldPosition`、`_lockedCellPos`、`_lockedLayerIndex`
2. 删除 `LockPosition()` 方法整体
3. 删除 `UnlockPosition()` 方法整体
4. `UpdateHoePreview` 中移除 `if (_isLocked) return;`（约第 510 行）
5. `UpdateWateringPreview` 中移除 `if (_isLocked) return;`（约第 619 行）
6. `UpdateSeedPreview` 中如有锁定相关逻辑也一并移除
7. `LockPosition` 中的"锁定时渲染 ghost 1+8"逻辑不再需要——ghost 每帧自动更新，队列预览通过 `AddQueuePreview` 接收 ghost 数据

### 2.3 `IsLocked` 属性处理

当前代码中可能有 `IsLocked` 公开属性被外部引用。需要搜索确认：
- 如果 `GameInputManager` 中有 `farmPreview.IsLocked` 判断 → 一并移除
- 如果其他脚本引用 → 替换为 false 或移除判断

### 2.4 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-A1 | ghost 预览永不被锁定，每帧都执行视觉更新 | 移除 `_isLocked` 和 `if (_isLocked) return` |
| CP-A2 | 移除后不影响 `UpdateRealtimeData`（实时数据始终更新） | `UpdateRealtimeData` 在 `_isLocked` 检查之前调用，不受影响 |

---

## 三、模块 B：FarmToolPreview.UpdateHoePreview — Ghost 差异化过滤（R1）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`（约第 460 行）

### 3.1 现状代码摘要

```csharp
// 当前逻辑（约第 525-545 行）：
if (canTill && FarmlandBorderManager.Instance != null)
{
    var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
    foreach (var kvp in previewTiles)
    {
        if (kvp.Value != null)
        {
            ghostTilemap.SetTile(kvp.Key, kvp.Value);  // ← 全量放置，不过滤
            currentPreviewPositions.Add(kvp.Key);
        }
    }
}
```

问题：`GetPreviewTiles` 返回 1+8 个 tile，全部放到 ghostTilemap，不区分哪些位置已有相同 tile。

### 3.2 改动

在遍历 `previewTiles` 时，逐个对比 `tilledTilemap` 上的实际 tile，只放置差异 tile。

### 3.3 改动后伪代码

```csharp
// 替换原有的 foreach 循环（约第 525-545 行）
if (canTill && FarmlandBorderManager.Instance != null)
{
    var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
    
    // 🔴 R1：获取实际 tilledTilemap 用于差异对比
    var tilemaps = FarmTileManager.Instance?.GetLayerTilemaps(layerIndex);
    Tilemap actualTilemap = tilemaps?.tilledTilemap;
    
    // 缓存差异化结果供入队时复制
    if (_currentGhostTileData == null)
        _currentGhostTileData = new Dictionary<Vector3Int, TileBase>();
    else
        _currentGhostTileData.Clear();
    
    foreach (var kvp in previewTiles)
    {
        if (kvp.Value == null) continue;
        
        // 差异化过滤：对比实际 Tilemap
        TileBase actualTile = actualTilemap?.GetTile(kvp.Key);
        if (kvp.Value == actualTile) continue;  // 相同则跳过
        
        ghostTilemap.SetTile(kvp.Key, kvp.Value);
        currentPreviewPositions.Add(kvp.Key);
        _currentGhostTileData[kvp.Key] = kvp.Value;
    }
}
else
{
    // 不可锄地时清空缓存
    _currentGhostTileData?.Clear();
}
```

### 3.4 新增字段和属性

```csharp
// 差异化 ghost tile 数据缓存（供入队时复制）
private Dictionary<Vector3Int, TileBase> _currentGhostTileData;

/// <summary>
/// 当前 ghost 预览的差异化 tile 数据快照（耕地模式）。
/// 入队时复制此数据到队列预览，确保三层数据一致。
/// </summary>
public Dictionary<Vector3Int, TileBase> CurrentGhostTileData => _currentGhostTileData;
```

### 3.5 关键设计决策

- 对比的是 `tilledTilemap`（耕地 Tilemap），不是 groundTilemap
- `GetPreviewTiles` 的计算逻辑不修改（DC-1 设计约束），差异过滤在调用方完成
- 中心块一定是差异（因为中心块位置当前没有耕地 tile，`actualTile` 为 null）
- `_currentGhostTileData` 每帧更新，入队时读取的是最新的 ghost 数据

### 3.6 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-B1 | ghost 预览只包含与 tilledTilemap 不同的 tile | `kvp.Value == actualTile` 时 continue |
| CP-B2 | 中心块始终显示（因为 actualTile 为 null） | `CanTillAt` 为 true 意味着该位置未耕作 |
| CP-B3 | `CurrentGhostTileData` 与 ghostTilemap 上的 tile 一一对应 | 同一循环中同步更新 |

---

## 四、模块 C：FarmToolPreview.UpdateWateringPreview — 进入新格子才随机（R3）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateWateringPreview`（约第 565 行）、`Hide`（约第 786 行）

### 4.1 现状代码摘要

```csharp
// 当前逻辑（约第 635-642 行）：
if (isValid)
{
    var puddleTile = FarmVisualManager.Instance?.GetRandomPuddleTile();  // ← 每帧随机
    if (puddleTile != null)
    {
        ghostTilemap.SetTile(cellPos, puddleTile);
        currentPreviewPositions.Add(cellPos);
    }
}
```

问题：每帧调用 `GetRandomPuddleTile()`，鼠标停在同一格子时每帧随机不同样式。

### 4.2 改动

新增 `_lastWateringCellPos` 和 `_cachedPuddleVariant` 字段，只在 cellPos 变化时重新随机。

### 4.3 新增字段和属性

```csharp
// 浇水 ghost 缓存（进入新格子才随机）
private Vector3Int _lastWateringCellPos = new Vector3Int(int.MinValue, int.MinValue, 0);
private int _cachedPuddleVariant = -1;

/// <summary>
/// 当前浇水 ghost 的 puddleVariant（供入队时复制）。
/// </summary>
public int CurrentPuddleVariant => _cachedPuddleVariant;
```

### 4.4 改动后伪代码

```csharp
// 替换原有的 if (isValid) 块（约第 635-642 行）
if (isValid)
{
    // 🔴 R3：进入新格子才随机
    if (cellPos != _lastWateringCellPos)
    {
        var puddleTiles = FarmVisualManager.Instance?.GetPuddleTiles();
        int count = puddleTiles != null ? puddleTiles.Length : 3;
        _cachedPuddleVariant = Random.Range(0, count);
        _lastWateringCellPos = cellPos;
    }
    
    // 使用缓存的 variant 获取确定性 tile
    var tiles = FarmVisualManager.Instance?.GetPuddleTiles();
    if (tiles != null && _cachedPuddleVariant >= 0 && _cachedPuddleVariant < tiles.Length)
    {
        ghostTilemap.SetTile(cellPos, tiles[_cachedPuddleVariant]);
        currentPreviewPositions.Add(cellPos);
    }
}
```

### 4.5 Hide 方法改动

```csharp
// 在 Hide() 中重置浇水缓存
_lastWateringCellPos = new Vector3Int(int.MinValue, int.MinValue, 0);
_cachedPuddleVariant = -1;
```

### 4.6 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-C1 | 浇水 ghost 在同一格子内不重新随机 | `cellPos != _lastWateringCellPos` 判断 |
| CP-C2 | 鼠标移出再移回时重新随机 | 移出时 cellPos 变化，`_lastWateringCellPos` 更新；移回时又变化，重新随机 |
| CP-C3 | `CurrentPuddleVariant` 与 ghostTilemap 上的 tile 对应 | 同一代码块中同步更新 |
| CP-C4 | Hide 后重置缓存，下次显示时重新随机 | `Hide()` 中重置 `_lastWateringCellPos` |

---

## 五、模块 D：FarmToolPreview — 执行预览层数据结构与方法（R6/R8）

> 涉及文件：`FarmToolPreview.cs`
> 新增方法：`PromoteToExecutingPreview`、`RemoveExecutingPreview`

### 5.1 新增数据结构

```csharp
// 执行预览追踪（正在执行动画的操作，受保护不被 WASD 清除）
private Dictionary<Vector3Int, List<Vector3Int>> executingTileGroups 
    = new Dictionary<Vector3Int, List<Vector3Int>>();

// 浇水执行预览（单点）
private HashSet<Vector3Int> executingWaterPositions = new HashSet<Vector3Int>();

// 种子执行预览
private List<(Vector3Int cellPos, SpriteRenderer renderer)> executingSeedPreviews 
    = new List<(Vector3Int, SpriteRenderer)>();
```

### 5.2 PromoteToExecutingPreview 方法

```csharp
/// <summary>
/// 将队列预览提升为执行预览（ProcessNextAction 出队时调用）。
/// tile 保留在 queuePreviewTilemap 上，只是追踪数据从队列转移到执行。
/// </summary>
public void PromoteToExecutingPreview(Vector3Int cellPos)
{
    // 耕地：从 tillQueueTileGroups 转移到 executingTileGroups
    if (tillQueueTileGroups.TryGetValue(cellPos, out var tilePositions))
    {
        executingTileGroups[cellPos] = tilePositions;
        tillQueueTileGroups.Remove(cellPos);
    }
    // 浇水：从 queuePreviewPositions 转移到 executingWaterPositions
    else if (queuePreviewPositions.Contains(cellPos) && !activeSeedQueuePreviews.Exists(x => x.cellPos == cellPos))
    {
        executingWaterPositions.Add(cellPos);
    }
    // 种子：从 activeSeedQueuePreviews 转移到 executingSeedPreviews
    else
    {
        int idx = activeSeedQueuePreviews.FindIndex(x => x.cellPos == cellPos);
        if (idx >= 0)
        {
            executingSeedPreviews.Add(activeSeedQueuePreviews[idx]);
            activeSeedQueuePreviews.RemoveAt(idx);
        }
    }
    
    // 从队列追踪中移除（但 tile 保留在 tilemap 上）
    queuePreviewPositions.Remove(cellPos);
}
```

### 5.3 RemoveExecutingPreview 方法

```csharp
/// <summary>
/// 清除执行预览（动画完成后调用，此时 tile 已落地，视觉无缝）。
/// </summary>
public void RemoveExecutingPreview(Vector3Int cellPos)
{
    // 耕地：清除关联的所有 tile
    if (executingTileGroups.TryGetValue(cellPos, out var tilePositions))
    {
        if (queuePreviewTilemap != null)
        {
            foreach (var pos in tilePositions)
                queuePreviewTilemap.SetTile(pos, null);
        }
        executingTileGroups.Remove(cellPos);
        return;
    }
    
    // 浇水：清除单点
    if (executingWaterPositions.Contains(cellPos))
    {
        queuePreviewTilemap?.SetTile(cellPos, null);
        executingWaterPositions.Remove(cellPos);
        return;
    }
    
    // 种子：回收 SpriteRenderer
    int idx = executingSeedPreviews.FindIndex(x => x.cellPos == cellPos);
    if (idx >= 0)
    {
        var entry = executingSeedPreviews[idx];
        entry.renderer.enabled = false;
        seedQueuePool.Add(entry.renderer);
        executingSeedPreviews.RemoveAt(idx);
    }
}
```

### 5.4 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-D1 | PromoteToExecutingPreview 后，tile 保留在 tilemap 上 | 只转移追踪数据，不调用 SetTile(null) |
| CP-D2 | RemoveExecutingPreview 清除 tilemap 上的 tile | 遍历 tilePositions 调用 SetTile(null) |
| CP-D3 | 执行预览在动画完成前不被清除 | ClearAllQueuePreviews 跳过 executingTileGroups（模块 E） |

---

## 六、模块 E：FarmToolPreview.ClearAllQueuePreviews — 边界残留修复（R5/R7）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`ClearAllQueuePreviews`（约第 1130 行）

### 6.1 现状代码摘要

```csharp
public void ClearAllQueuePreviews()
{
    // 🔴 BUG：只遍历 queuePreviewPositions（中心块），不清除 tillQueueTileGroups 中的边界 tile
    if (queuePreviewTilemap != null)
    {
        foreach (var pos in queuePreviewPositions)
            queuePreviewTilemap.SetTile(pos, null);
    }
    // 回收种子队列预览...
    // queuePreviewPositions.Clear();
    // tillQueueTileGroups.Clear();  ← 只清了追踪数据，没清 tilemap 上的边界 tile
}
```

### 6.2 改动后完整伪代码

```csharp
public void ClearAllQueuePreviews()
{
    if (queuePreviewTilemap != null)
    {
        // 🔴 R5：清除耕地队列的所有 tile（中心块 + 边界），跳过执行预览
        foreach (var kvp in tillQueueTileGroups)
        {
            if (executingTileGroups.ContainsKey(kvp.Key)) continue;  // R7：跳过执行预览
            foreach (var pos in kvp.Value)
                queuePreviewTilemap.SetTile(pos, null);
        }
        
        // 清除浇水等单点队列预览，跳过执行预览
        foreach (var pos in queuePreviewPositions)
        {
            if (executingWaterPositions.Contains(pos)) continue;  // R7：跳过执行预览
            // 跳过已在 tillQueueTileGroups 中处理过的（耕地中心块）
            if (tillQueueTileGroups.ContainsKey(pos)) continue;
            queuePreviewTilemap.SetTile(pos, null);
        }
    }
    
    // 回收种子队列预览（跳过执行中的）
    for (int i = activeSeedQueuePreviews.Count - 1; i >= 0; i--)
    {
        var entry = activeSeedQueuePreviews[i];
        // 检查是否已提升为执行预览
        if (executingSeedPreviews.Exists(x => x.cellPos == entry.cellPos)) continue;
        entry.renderer.enabled = false;
        seedQueuePool.Add(entry.renderer);
        activeSeedQueuePreviews.RemoveAt(i);
    }
    
    // 清空队列追踪数据（不清 executing* 数据）
    tillQueueTileGroups.Clear();
    queuePreviewPositions.Clear();
    // activeSeedQueuePreviews 中剩余的是已提升为执行预览的，也清空（追踪已转移）
    activeSeedQueuePreviews.Clear();
}
```

### 6.3 关键设计决策

- `tillQueueTileGroups` 遍历时清除每个 tile 位置（修复边界残留 R5）
- `executingTileGroups` / `executingWaterPositions` / `executingSeedPreviews` 中的 tile 不被清除（执行预览保护 R7）
- 清空追踪数据时不清 `executing*` 系列

### 6.4 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-E1 | ClearAllQueuePreviews 后 queuePreviewTilemap 上只剩执行预览的 tile | 遍历 tillQueueTileGroups 和 queuePreviewPositions 清除，跳过 executing* |
| CP-E2 | 耕地边界 tile 被正确清除（修复残留 bug） | 遍历 `tillQueueTileGroups[cellPos]` 的每个位置 |
| CP-E3 | 执行预览不受 ClearAllQueuePreviews 影响 | `executingTileGroups.ContainsKey` 跳过 |

---

## 七、模块 F：FarmToolPreview.AddQueuePreview — 接收 Ghost 数据（R4）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`AddQueuePreview`（约第 1025 行）

### 7.1 现状代码摘要

```csharp
// Till 分支（约第 1068-1082 行）：
else if (type == FarmActionType.Till)
{
    if (FarmlandBorderManager.Instance != null)
    {
        var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
        // ← 独立调用 GetPreviewTiles，与 ghost 是两次独立计算
        // ...
    }
}
```

### 7.2 改动

`AddQueuePreview` 新增可选参数 `Dictionary<Vector3Int, TileBase> ghostTileData`。耕地入队时优先使用 ghost 数据，兜底才独立计算。

### 7.3 改动后签名和伪代码

```csharp
public void AddQueuePreview(Vector3Int cellPos, int layerIndex, FarmActionType type, 
    int puddleVariant = -1, Dictionary<Vector3Int, TileBase> ghostTileData = null)
{
    if (queuePreviewPositions.Contains(cellPos)) return;
    
    if (type == FarmActionType.PlantSeed)
    {
        // 种子队列预览：保持不变
        // ...
    }
    else if (type == FarmActionType.Till)
    {
        // 🔴 R4：优先使用 ghost 数据，兜底独立计算
        Dictionary<Vector3Int, TileBase> tilesToPlace;
        if (ghostTileData != null && ghostTileData.Count > 0)
        {
            tilesToPlace = ghostTileData;
        }
        else
        {
            // 兜底：独立计算（向后兼容）
            tilesToPlace = new Dictionary<Vector3Int, TileBase>();
            if (FarmlandBorderManager.Instance != null)
            {
                var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
                foreach (var kvp in previewTiles)
                {
                    if (kvp.Value != null)
                        tilesToPlace[kvp.Key] = kvp.Value;
                }
            }
        }
        
        var tilePositions = new List<Vector3Int>();
        foreach (var kvp in tilesToPlace)
        {
            if (kvp.Value != null && queuePreviewTilemap != null)
            {
                queuePreviewTilemap.SetTile(kvp.Key, kvp.Value);
                queuePreviewTilemap.SetColor(kvp.Key, new Color(1f, 1f, 1f, queuePreviewAlpha));
                tilePositions.Add(kvp.Key);
            }
        }
        tillQueueTileGroups[cellPos] = tilePositions;
    }
    else if (type == FarmActionType.Water)
    {
        // 浇水队列预览：保持现有 puddleVariant 逻辑不变
        // ...（现有代码不变）
    }
    
    queuePreviewPositions.Add(cellPos);
}
```

### 7.4 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-F1 | 队列预览的 tile 数据与入队时的 ghost 数据一致 | `ghostTileData` 参数传递 |
| CP-F2 | 未提供 ghostTileData 时兜底独立计算（向后兼容） | `else` 分支调用 `GetPreviewTiles` |

---

## 八、模块 G：GameInputManager — 移除 Lock/Unlock + 集成执行预览（R2/R6/R7/R8）

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`ProcessNextAction`、`OnFarmActionAnimationComplete`、`OnCollectAnimationComplete`、`ClearActionQueue`、`HandleMovement`、`CancelFarmingNavigation`、`TryEnqueueFarmTool`、`EnqueueAction`

### 8.1 移除所有 LockPosition / UnlockPosition 调用（R2）

需要移除的调用点（当前会话中实际读取确认）：

| 位置 | 当前代码 | 改动 |
|------|---------|------|
| `ProcessNextAction`（约第 2453 行） | `FarmToolPreview.Instance?.LockPosition(request.worldPos, request.cellPos, request.layerIndex)` | 删除整行 |
| `ProcessNextAction` 队列空分支（约第 2403 行） | `FarmToolPreview.Instance?.UnlockPosition()` | 删除整行 |
| `HandleMovement` WASD 中断（约第 460 行） | `FarmToolPreview.Instance?.UnlockPosition()` | 删除整行 |
| `HandleMovement` autoNavigator 分支（约第 492 行） | `farmPreviewNav.UnlockPosition()` | 删除整行 |
| `CancelFarmingNavigation` 内部 | 搜索确认是否有 `UnlockPosition` 调用 | 如有则删除 |

### 8.2 ProcessNextAction 出队时提升为执行预览（R6）

```csharp
// 在 ProcessNextAction 中，原来 LockPosition 的位置替换为：
// 🔴 R6：出队时将队列预览提升为执行预览
FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
```

完整改动位置：在 `_isExecutingFarming = true;` 之后，距离判断之前。

### 8.3 OnFarmActionAnimationComplete 清除执行预览（R8）

```csharp
// 当前代码（约第 2573 行）：
FarmToolPreview.Instance?.RemoveQueuePreview(_currentProcessingRequest.cellPos);

// 改为：
FarmToolPreview.Instance?.RemoveExecutingPreview(_currentProcessingRequest.cellPos);
```

### 8.4 OnCollectAnimationComplete 清除执行预览（R8）

```csharp
// 当前代码（约第 2596 行）：
FarmToolPreview.Instance?.RemoveQueuePreview(_currentProcessingRequest.cellPos);

// 改为：
FarmToolPreview.Instance?.RemoveExecutingPreview(_currentProcessingRequest.cellPos);
```

### 8.5 PlantSeed 分支清除执行预览（R8）

```csharp
// ExecuteFarmAction 中 PlantSeed 分支（约第 2533 行）：
// 当前没有调用 RemoveQueuePreview（种子无动画，直接执行）
// 需要在 ProcessNextAction 之前添加：
FarmToolPreview.Instance?.RemoveExecutingPreview(request.cellPos);
```

### 8.6 TryEnqueueFarmTool 传递 Ghost 数据（R4）

```csharp
// 当前 TryEnqueueFarmTool（约第 2689 行）：
// 耕地入队时，读取 CurrentGhostTileData 传递给 EnqueueAction

private void TryEnqueueFarmTool(ToolData tool)
{
    var farmPreview = FarmToolPreview.Instance;
    if (farmPreview == null || !farmPreview.IsValid()) return;

    var type = tool.toolType == ToolType.Hoe ? FarmActionType.Till : FarmActionType.Water;
    
    // 🔴 R4：耕地入队时读取 ghost 数据
    Dictionary<Vector3Int, TileBase> ghostData = null;
    if (type == FarmActionType.Till)
    {
        ghostData = farmPreview.CurrentGhostTileData;
        // 复制一份快照（ghost 数据每帧更新，入队后不应被后续帧覆盖）
        if (ghostData != null)
            ghostData = new Dictionary<Vector3Int, TileBase>(ghostData);
    }
    
    // 🔴 R3/补丁003：浇水入队时使用 ghost 缓存的 variant
    int variant = -1;
    if (type == FarmActionType.Water)
    {
        variant = farmPreview.CurrentPuddleVariant;
        // 兜底：如果 ghost 没有缓存（异常情况），随机分配
        if (variant < 0)
        {
            var puddleTiles = FarmVisualManager.Instance?.GetPuddleTiles();
            int puddleCount = puddleTiles != null ? puddleTiles.Length : 3;
            variant = Random.Range(0, puddleCount);
        }
    }
    
    EnqueueAction(new FarmActionRequest
    {
        type = type,
        cellPos = farmPreview.CurrentCellPos,
        layerIndex = farmPreview.CurrentLayerIndex,
        worldPos = farmPreview.CurrentCursorPos,
        targetCrop = null,
        puddleVariant = variant
    }, ghostData);
}
```

### 8.7 EnqueueAction 新增 ghostTileData 参数（R4）

```csharp
// 签名变更：
private void EnqueueAction(FarmActionRequest request, Dictionary<Vector3Int, TileBase> ghostTileData = null)
{
    // ... 防重复逻辑不变 ...
    
    _queuedPositions.Add(key);
    _farmActionQueue.Enqueue(request);
    
    // 🔴 R4：传递 ghostTileData 给 AddQueuePreview
    FarmToolPreview.Instance?.AddQueuePreview(
        request.cellPos, request.layerIndex, request.type, 
        request.puddleVariant, ghostTileData);
    
    if (!_isProcessingQueue && !_isQueuePaused)
        ProcessNextAction();
}
```

### 8.8 WASD 中断逻辑调整（R7）

```csharp
// HandleMovement 中的 WASD 中断（约第 456-463 行）：
if (hasWASD && hasActiveQueue)
{
    ClearActionQueue();
    CancelFarmingNavigation();
    // 🔴 R2：移除 UnlockPosition 调用（ghost 永不锁定）
    // FarmToolPreview.Instance?.UnlockPosition();  ← 删除
    ToolActionLockManager.Instance?.ForceUnlock();
}
```

`ClearActionQueue` 内部调用 `ClearAllQueuePreviews()`，已在模块 E 中修复为跳过执行预览。

### 8.9 ClearActionQueue 调整

```csharp
// 当前 ClearActionQueue（约第 2600 行）：
// 移除内部的 UnlockPosition 调用（如有）
// ClearAllQueuePreviews 已在模块 E 中修复
```

### 8.10 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-G1 | 项目中无任何 `LockPosition` / `UnlockPosition` 调用 | 全局搜索确认 |
| CP-G2 | 出队时调用 `PromoteToExecutingPreview` | `ProcessNextAction` 中替换 `LockPosition` |
| CP-G3 | 动画完成后调用 `RemoveExecutingPreview`（替代 `RemoveQueuePreview`） | `OnFarmActionAnimationComplete` / `OnCollectAnimationComplete` |
| CP-G4 | 耕地入队时传递 ghost 数据快照 | `TryEnqueueFarmTool` 中复制 `CurrentGhostTileData` |
| CP-G5 | 浇水入队时使用 ghost 缓存的 `CurrentPuddleVariant` | `TryEnqueueFarmTool` 中读取 |
| CP-G6 | WASD 中断不影响执行预览 | `ClearAllQueuePreviews` 跳过 `executingTileGroups` |

---

## 九、完整数据流

### 9.1 耕地完整流程

```
1. 鼠标移动 → UpdateHoePreview
   ├─ GetPreviewTiles(layerIndex, cellPos) → 获取 1+8 tile
   ├─ 对比 tilledTilemap 实际 tile → 过滤出差异 tile（R1）
   ├─ 差异 tile 放到 ghostTilemap → 显示差异化预览
   └─ 缓存到 CurrentGhostTileData

2. 玩家点击 → TryEnqueueFarmTool → EnqueueAction
   ├─ 复制 CurrentGhostTileData 快照（R4）
   ├─ AddQueuePreview(cellPos, layerIndex, Till, ghostTileData=快照)
   │   ├─ 快照中的 tile 放到 queuePreviewTilemap
   │   └─ 记录到 tillQueueTileGroups[cellPos]
   └─ ghost 继续跟随鼠标（不锁定，R2）

3. ProcessNextAction 出队
   ├─ PromoteToExecutingPreview(cellPos)（R6）
   │   ├─ 从 tillQueueTileGroups 移到 executingTileGroups
   │   └─ tile 保留在 queuePreviewTilemap 上
   └─ ExecuteFarmAction → 开始动画

4. WASD 中断（如果发生）
   ├─ ClearActionQueue → ClearAllQueuePreviews
   │   ├─ 清除队列中未执行的预览
   │   └─ 跳过 executingTileGroups 中的（R7）
   └─ 执行预览保留，动画继续

5. 动画完成 → OnFarmActionAnimationComplete
   ├─ TillAt → UpdateBorderAt（数据层落地）
   ├─ RemoveExecutingPreview(cellPos)（R8）
   └─ 实际 tile 已在 tilledTilemap 上，视觉无缝
```

### 9.2 浇水完整流程

```
1. 鼠标移动 → UpdateWateringPreview
   ├─ cellPos 变化时随机 puddleVariant，缓存（R3）
   ├─ 使用缓存的 variant 获取 tile
   ├─ tile 放到 ghostTilemap
   └─ 缓存 CurrentPuddleVariant

2. 玩家点击 → TryEnqueueFarmTool → EnqueueAction
   ├─ 读取 CurrentPuddleVariant（R4 浇水版）
   ├─ AddQueuePreview(cellPos, layerIndex, Water, puddleVariant)
   └─ ghost 继续跟随鼠标

3. ProcessNextAction 出队
   ├─ PromoteToExecutingPreview(cellPos)
   └─ ExecuteFarmAction → 开始动画

4. 动画完成
   ├─ SetWatered（数据层落地）
   ├─ RemoveExecutingPreview(cellPos)
   └─ 水渍 tile 已在 waterPuddleTilemap 上
```

---

## 十、正确性属性汇总

| 编号 | 属性 | 模块 |
|------|------|------|
| CP-A1 | ghost 预览永不被锁定，每帧都执行视觉更新 | A |
| CP-A2 | 移除锁定不影响 UpdateRealtimeData | A |
| CP-B1 | ghost 预览只包含与 tilledTilemap 不同的 tile | B |
| CP-B2 | 中心块始终显示 | B |
| CP-B3 | CurrentGhostTileData 与 ghostTilemap 一致 | B |
| CP-C1 | 浇水 ghost 在同一格子内不重新随机 | C |
| CP-C2 | 移出再移回时重新随机 | C |
| CP-C3 | CurrentPuddleVariant 与 ghostTilemap 对应 | C |
| CP-C4 | Hide 后重置缓存 | C |
| CP-D1 | PromoteToExecutingPreview 后 tile 保留 | D |
| CP-D2 | RemoveExecutingPreview 清除 tile | D |
| CP-D3 | 执行预览在动画完成前不被清除 | D |
| CP-E1 | ClearAllQueuePreviews 后只剩执行预览 tile | E |
| CP-E2 | 耕地边界 tile 被正确清除 | E |
| CP-E3 | 执行预览不受 ClearAllQueuePreviews 影响 | E |
| CP-F1 | 队列预览 tile 与入队时 ghost 数据一致 | F |
| CP-F2 | 未提供 ghostTileData 时兜底独立计算 | F |
| CP-G1 | 无任何 LockPosition/UnlockPosition 调用 | G |
| CP-G2 | 出队时调用 PromoteToExecutingPreview | G |
| CP-G3 | 动画完成后调用 RemoveExecutingPreview | G |
| CP-G4 | 耕地入队传递 ghost 数据快照 | G |
| CP-G5 | 浇水入队使用 ghost 缓存的 variant | G |
| CP-G6 | WASD 中断不影响执行预览 | G |

---

## 十一、涉及文件汇总

| 文件 | 改动类型 | 涉及模块 |
|------|----------|----------|
| `FarmToolPreview.cs` | 修改 | A（移除锁定）、B（差异化过滤）、C（浇水缓存）、D（执行预览层）、E（ClearAllQueuePreviews 修复）、F（AddQueuePreview 接收 ghost 数据） |
| `GameInputManager.cs` | 修改 | G（移除 Lock/Unlock、集成执行预览、传递 ghost 数据） |

---

## 十二、异议说明

实际落地通过数据层驱动（`TillAt` → `UpdateBorderAt`），不直接复制 tile。用户已认可此异议。视觉结果一致（计算逻辑相同），但驱动方式必须是"数据层更新 → 视觉层跟随"。
