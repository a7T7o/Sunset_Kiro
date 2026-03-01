# 补丁003 设计文档 V3 — 预览系统重构与bug修复

> V3 是在 V2 基础上的增量设计，仅涉及预览系统相关改动。
> V2 的模块 A-G、I 不受影响。V3 替代 V2 的模块 H（预览系统）。
> 来源：`requirementsV3.md`（P1-P4 四个问题）

---

## 目录

1. [模块 J：种子预览位置修复（P4）](#一模块-j种子预览位置修复p4)
2. [模块 K：耕地队列预览8+1完整化（P2）](#二模块-k耕地队列预览81完整化p2)
3. [模块 L：浇水队列预览一致性（P3）](#三模块-l浇水队列预览一致性p3)
4. [模块 M：预览系统视觉重构 — shader + Tilemap（P1）](#四模块-m预览系统视觉重构--shader--tilemapp1)
5. [正确性属性汇总](#五正确性属性汇总)
6. [涉及文件汇总](#六涉及文件汇总)

---

## 一、模块 J：种子预览位置修复（P4）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateSeedPreview`

### 1.1 现状

```csharp
// UpdateSeedPreview 接收 alignedPos（来自 PlacementGridCalculator.GetCellCenter）
// PlacementGridCalculator.GetCellCenter: Mathf.Floor(x) + 0.5f
// 但实际种植使用 tilemaps.GetCellCenterWorld(cellPos)（Tilemap 原生方法）

// 当前代码：
seedGridRenderer.transform.position = alignedPos;      // ← 用的是 PlacementGridCalculator 的坐标
seedPreviewRenderer.transform.position = alignedPos;    // ← 同上
```

### 1.2 根因

`PlacementGridCalculator.GetCellCenter` 是纯数学计算（假设标准1x1网格，origin=(0,0)），不考虑 Tilemap 的实际 Grid 配置。而 `tilemaps.GetCellCenterWorld` 是 Tilemap 原生方法，考虑了 Grid 的 cellSize、cellGap、transform 偏移。

在 UpdateSeedPreview 内部已经通过 `tilemaps.WorldToCell(alignedPos)` 得到了 `cellPos`，可以直接用 `GetCellCenterWorld` 获取正确坐标。

### 1.3 改动

在 UpdateSeedPreview 中，计算出 cellPos 后，使用 `GetCellCenterWorld(layerIndex, cellPos)` 获取正确的格子中心坐标，替代 `alignedPos` 用于 seedGridRenderer 和 seedPreviewRenderer 的位置设置。

```
伪代码：
Vector3Int cellPos = tilemaps.WorldToCell(alignedPos);
Vector3 correctCenter = GetCellCenterWorld(layerIndex, cellPos);  // ← 使用 Tilemap 原生坐标

seedGridRenderer.transform.position = correctCenter;      // ← 修正
seedPreviewRenderer.transform.position = correctCenter;    // ← 修正
```

### 1.4 正确性属性

- CP-J1：种子预览位置（seedGridRenderer + seedPreviewRenderer）必须与 `tilemaps.GetCellCenterWorld(cellPos)` 返回的坐标一致
- CP-J2：修改不影响 UpdateSeedPreview 的其他逻辑（状态判断、颜色设置、锁定机制）

---

## 二、模块 K：耕地队列预览8+1完整化（P2）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`AddQueuePreview`、`RemoveQueuePreview`

### 2.1 现状

```csharp
// AddQueuePreview Till 分支：
var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
if (previewTiles.ContainsKey(cellPos))
    tile = previewTiles[cellPos];  // ← 只取了中心块！其余8个边界tile被丢弃
```

`GetPreviewTiles` 返回一个 `Dictionary<Vector3Int, TileBase>`，包含中心块 + 8个边界变化tile（共最多9个）。但 AddQueuePreview 只取了中心块。

### 2.2 改动

**AddQueuePreview Till 分支**：遍历 `previewTiles` 的所有 key-value，全部放到 `queuePreviewTilemap` 上。

**数据结构扩展**：当前 `queuePreviewPositions` 是 `HashSet<Vector3Int>`，只记录中心点。需要新增一个字典来记录每个耕地队列项关联的所有tile位置，以便 RemoveQueuePreview 时能正确清理。

```
新增字段：
Dictionary<Vector3Int, List<Vector3Int>> tillQueueTileGroups
  key = 中心 cellPos（入队时的主键）
  value = 该次耕地操作影响的所有 tile 位置列表（1+8）
```

**RemoveQueuePreview**：当移除耕地队列项时，从 `tillQueueTileGroups` 获取关联的所有位置，逐个清除 queuePreviewTilemap 上的 tile。

**ClearAllQueuePreviews**：清空 `tillQueueTileGroups`。

### 2.3 伪代码

```
AddQueuePreview(cellPos, layerIndex, Till):
  if queuePreviewPositions.Contains(cellPos): return
  
  previewTiles = FarmlandBorderManager.GetPreviewTiles(layerIndex, cellPos)
  tilePositions = new List<Vector3Int>()
  
  foreach (pos, tile) in previewTiles:
    if tile != null:
      queuePreviewTilemap.SetTile(pos, tile)
      queuePreviewTilemap.SetColor(pos, Color(1,1,1, queuePreviewAlpha))
      tilePositions.Add(pos)
  
  tillQueueTileGroups[cellPos] = tilePositions
  queuePreviewPositions.Add(cellPos)

RemoveQueuePreview(cellPos):
  // 检查是否是耕地队列（有 tileGroup）
  if tillQueueTileGroups.TryGetValue(cellPos, out tilePositions):
    foreach pos in tilePositions:
      queuePreviewTilemap.SetTile(pos, null)
    tillQueueTileGroups.Remove(cellPos)
  else:
    // 浇水或其他：单点清除（原有逻辑）
    queuePreviewTilemap.SetTile(cellPos, null)
  
  // 种子队列检查（原有逻辑不变）
  ...
  queuePreviewPositions.Remove(cellPos)

ClearAllQueuePreviews:
  // 原有逻辑 + 清空 tillQueueTileGroups
  tillQueueTileGroups.Clear()
```

### 2.4 边界tile叠加问题

多个相邻耕地入队时，后入队的边界tile可能覆盖先入队的。这是预期行为——`GetPreviewTiles` 的 predicate 只考虑当前入队位置，不考虑队列中其他位置。完美的叠加需要在每次入队时重新计算所有已入队位置的边界，复杂度较高。

当前方案选择简单实现：每次入队独立计算，接受边界tile可能不完美叠加。这与鼠标跟随预览的行为一致（鼠标跟随也只计算当前位置的8+1）。

### 2.5 正确性属性

- CP-K1：耕地入队后，queuePreviewTilemap 上显示完整的 1+8 个tile
- CP-K2：耕地出队后，关联的所有 tile 位置被正确清除
- CP-K3：ClearAllQueuePreviews 清空 tillQueueTileGroups

---

## 三、模块 L：浇水队列预览一致性（P3）

> 涉及文件：`FarmToolPreview.cs`、`GameInputManager.cs`
> 涉及方法：`AddQueuePreview`、`EnqueueAction`

### 3.1 现状

```csharp
// AddQueuePreview Water 分支：
tile = FarmVisualManager.Instance?.GetRandomPuddleTile();  // ← 随机选择

// 实际浇水（FarmVisualManager.UpdateTileVisual）：
int variant = Mathf.Clamp(tileData.puddleVariant, 0, wetPuddleTiles.Length - 1);
puddleTile = wetPuddleTiles[variant];  // ← 使用存储的 variant 索引
```

### 3.2 方案

在入队时预分配 `puddleVariant`，存储到队列数据中，预览和实际执行都使用同一个 variant。

**方案细节**：
- `FarmActionRequest` 结构体新增 `int puddleVariant` 字段（默认 -1 表示未分配）
- `EnqueueAction` 中，如果是 Water 类型，预分配 `puddleVariant = Random.Range(0, puddleTileCount)`
- `AddQueuePreview` 接收 puddleVariant 参数，使用 `FarmVisualManager.GetPuddleTiles()[variant]` 获取确定的 tile
- `ExecuteFarmAction` 的 Water 分支执行时，将预分配的 variant 写入 `tileData.puddleVariant`

### 3.3 改动涉及

1. `FarmActionRequest`（GameInputManager.cs 内部结构体）：新增 `puddleVariant` 字段
2. `EnqueueAction`（GameInputManager.cs）：Water 类型入队时预分配 variant
3. `AddQueuePreview`（FarmToolPreview.cs）：新增 `puddleVariant` 参数，使用确定性 tile
4. `ExecuteFarmAction`（GameInputManager.cs）：Water 分支将 variant 传递给 FarmTileData

### 3.4 伪代码

```
// EnqueueAction 中：
if (type == Water):
  int puddleCount = FarmVisualManager.Instance?.GetPuddleTiles()?.Length ?? 3
  request.puddleVariant = Random.Range(0, puddleCount)

// AddQueuePreview 签名变更：
AddQueuePreview(cellPos, layerIndex, type, puddleVariant = -1)

// AddQueuePreview Water 分支：
if puddleVariant >= 0:
  var tiles = FarmVisualManager.Instance?.GetPuddleTiles()
  if tiles != null && puddleVariant < tiles.Length:
    tile = tiles[puddleVariant]
else:
  tile = FarmVisualManager.Instance?.GetRandomPuddleTile()  // 兜底

// ExecuteFarmAction Water 分支（tile更新前）：
tileData.puddleVariant = request.puddleVariant >= 0 ? request.puddleVariant : Random.Range(0, 3)
```

### 3.5 正确性属性

- CP-L1：浇水入队时预分配 puddleVariant，预览使用该 variant 对应的 tile
- CP-L2：浇水执行时使用同一个 puddleVariant，确保视觉一致
- CP-L3：puddleVariant 未分配时（-1），兜底使用随机 tile（向后兼容）

---

## 四、模块 M：预览系统视觉重构 — shader + Tilemap（P1）

> 涉及文件：新建 shader 文件、`FarmToolPreview.cs`
> 涉及方法：`EnsureComponents`、`UpdateHoePreview`、`UpdateWateringPreview`、`Hide`

### 4.1 现状

当前方案C实现：
- `hoeOverlayRenderer`：程序化 SpriteRenderer，纯色半透明方块覆盖在格子上
- `waterOverlayRenderer`：同上
- 视觉效果：只是一个绿色/红色方块，看不到实际的耕地/水渍tile样式

### 4.2 新方案：shader 颜色叠加 + ghostTilemap

**核心思路**：
- ghostTilemap（已有）作为鼠标跟随预览层，放置实际的耕地/水渍 tile
- 给 ghostTilemapRenderer 设置自定义 shader material，实现"原始tile颜色 + 叠加色"效果
- 有效时叠加绿色，无效时叠加红色
- queuePreviewTilemap（已有）作为队列预览层，使用默认 material，不叠加颜色

**shader 设计**：

名称：`FarmPreviewOverlay`
路径：`Assets/444_Shaders/Shader/FarmPreviewOverlay.shader`

功能：
- 基于 Sprites-Default shader
- 新增属性 `_OverlayColor`（Color，默认透明）
- 片元着色器：`finalColor.rgb = lerp(spriteColor.rgb, _OverlayColor.rgb, _OverlayColor.a)`
- 即：叠加色的 alpha 控制混合强度，alpha=0 时完全显示原色，alpha=1 时完全显示叠加色
- sprite 自身的 alpha 保持不变（用于 Tilemap 的整体透明度控制）

**Material 管理**：
- 在 EnsureComponents 中创建 Material 实例（`new Material(shader)`）
- 赋给 ghostTilemapRenderer
- 运行时通过 `material.SetColor("_OverlayColor", color)` 切换颜色

### 4.3 三层 Tilemap 架构

```
FarmToolPreview GameObject
├── GhostTilemap          — 鼠标跟随预览层（sortingOrder=9999）
│   └── Material: FarmPreviewOverlay shader
│       └── _OverlayColor: 绿色(0,1,0,0.35) 或 红色(1,0,0,0.35)
├── QueuePreviewTilemap   — 队列预览层（sortingOrder=9998）
│   └── Material: 默认 Sprites-Default（无颜色叠加）
└── 实际显示层            — farmlandCenterTilemap / waterPuddleTilemapNew（场景中已有）
```

### 4.4 UpdateHoePreview 改动

移除方案C的 `hoeOverlayRenderer` 使用，改为：
1. 在 ghostTilemap 上放置 1+8 个耕地预览 tile（已有逻辑，保持不变）
2. 根据 isValid 设置 ghostTilemapRenderer 的 material `_OverlayColor`
   - 有效：`new Color(0, 1, 0, 0.35f)`（绿色叠加）
   - 无效：`new Color(1, 0, 0, 0.35f)`（红色叠加）
3. 移除 `hoeOverlayRenderer.enabled = true` 相关代码

### 4.5 UpdateWateringPreview 改动

移除方案C的 `waterOverlayRenderer` 使用，改为：
1. 在 ghostTilemap 上放置水渍 tile（已有逻辑，保持不变）
2. 根据 isValid 设置 ghostTilemapRenderer 的 material `_OverlayColor`
3. 移除 `waterOverlayRenderer.enabled = true` 相关代码

### 4.6 EnsureComponents 改动

1. 加载 FarmPreviewOverlay shader 并创建 Material
2. 赋给 ghostTilemapRenderer
3. 移除 hoeOverlayRenderer / waterOverlayRenderer 的创建（或保留但不再使用）

### 4.7 Hide 改动

移除 hoeOverlayRenderer / waterOverlayRenderer 的隐藏逻辑（已不再使用）。

### 4.8 方案C组件清理

`hoeOverlayRenderer`、`waterOverlayRenderer`、`overlaySprite` 相关代码标记为废弃或移除：
- `CreateOverlaySprite()` — 不再需要
- `CreateOverlayRenderer()` — 不再需要
- EnsureComponents 中的覆盖层初始化 — 移除

### 4.9 正确性属性

- CP-M1：ghostTilemap 使用 FarmPreviewOverlay shader，支持 `_OverlayColor` 属性
- CP-M2：耕地鼠标跟随预览显示实际耕地 tile + 颜色叠加（绿色/红色）
- CP-M3：浇水鼠标跟随预览显示实际水渍 tile + 颜色叠加（绿色/红色）
- CP-M4：队列预览层（queuePreviewTilemap）不使用 shader，显示原始 tile 颜色
- CP-M5：方案C的 hoeOverlayRenderer / waterOverlayRenderer 不再被启用
- CP-M6：种子预览不受影响（仍使用 seedGridRenderer + seedPreviewRenderer）

---

## 五、正确性属性汇总

| 编号 | 属性 | 模块 |
|------|------|------|
| CP-J1 | 种子预览位置与 tilemaps.GetCellCenterWorld 一致 | J |
| CP-J2 | 修改不影响 UpdateSeedPreview 其他逻辑 | J |
| CP-K1 | 耕地入队后显示完整 1+8 tile | K |
| CP-K2 | 耕地出队后关联 tile 全部清除 | K |
| CP-K3 | ClearAllQueuePreviews 清空 tillQueueTileGroups | K |
| CP-L1 | 浇水入队预分配 puddleVariant 用于预览 | L |
| CP-L2 | 浇水执行使用同一 puddleVariant | L |
| CP-L3 | puddleVariant 未分配时兜底随机 | L |
| CP-M1 | ghostTilemap 使用 FarmPreviewOverlay shader | M |
| CP-M2 | 耕地预览显示实际 tile + 颜色叠加 | M |
| CP-M3 | 浇水预览显示实际 tile + 颜色叠加 | M |
| CP-M4 | 队列预览层不使用 shader | M |
| CP-M5 | 方案C覆盖层不再启用 | M |
| CP-M6 | 种子预览不受影响 | M |

---

## 六、涉及文件汇总

| 文件 | 改动类型 | 涉及模块 |
|------|----------|----------|
| `FarmToolPreview.cs` | 修改 | J、K、L、M |
| `GameInputManager.cs` | 修改 | L（FarmActionRequest + EnqueueAction + ExecuteFarmAction） |
| `FarmPreviewOverlay.shader` | 新建 | M |
| `FarmVisualManager.cs` | 无改动 | （GetPuddleTiles 已在 V2 中暴露） |
| `FarmlandBorderManager.cs` | 无改动 | （GetPreviewTiles 已有） |
