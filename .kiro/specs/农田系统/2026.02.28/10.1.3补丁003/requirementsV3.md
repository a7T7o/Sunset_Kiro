# 补丁003 需求文档 V3 — 预览系统重构与bug修复

> 来源：用户验收补丁003 V2实施后的反馈（会话1续25~续27）
> 工作区：`.kiro/specs/农田系统/10.1.1补丁003/`
> 前置：tasksV2 全部18个任务已完成，本V3是在V2基础上的增量修复与重构

---

## 一、背景

补丁003 V2 实施完成后，用户在 Unity 中验收测试发现以下问题：
1. 预览系统方案C（程序化 SpriteRenderer 覆盖层）视觉效果极差，只是纯色方块
2. 队列预览逻辑不完整，耕地队列缺少8+1边界tile
3. 浇水队列预览与实际浇水视觉不一致
4. 种子预览位置与实际种植位置存在偏移

---

## 二、需求清单

### P1：预览系统视觉重构（方案C → shader + 三层Tilemap）

**用户原话**：
> 你的这个耕地预览方案C做的太菜了，就只是一个简单的绿色方块，浇水也是一个方块，那还不如设计shader然后双层tilemap呢

**现状问题**：
- `hoeOverlayRenderer` / `waterOverlayRenderer` 是程序化 SpriteRenderer，显示为纯色半透明方块
- 没有使用实际的耕地/水渍tile，视觉效果与最终结果完全不同
- 用户认为这不符合"预览应该接近实际效果"的设计理念

**用户期望方案**：
- 第1层（跟随鼠标预览）：使用 shader 实现颜色叠加效果，显示实际tile + 颜色标识（绿色=有效/红色=无效）
- 第2层（队列预览）：无 shader，普通显示实际tile（已入队的操作预览）
- 第3层（实际显示）：无 shader，正常的游戏显示层

**验收标准**：
- AC-P1.1：耕地鼠标跟随预览显示实际耕地tile + 绿色/红色颜色叠加
- AC-P1.2：浇水鼠标跟随预览显示实际水渍tile + 绿色/红色颜色叠加
- AC-P1.3：队列预览显示实际tile，无颜色叠加，与鼠标跟随预览视觉区分
- AC-P1.4：方案C的 hoeOverlayRenderer / waterOverlayRenderer 被移除或废弃
- AC-P1.5：种子预览保持现有放置系统复刻方案（绿框+作物sprite），不受此重构影响

---

### P2：耕地队列预览缺少8+1边界tile（严重bug）

**现状问题**：
- `AddQueuePreview` 的 Till 分支只从 `GetPreviewTiles` 取了中心块tile（`previewTiles[cellPos]`）
- 但 `UpdateHoePreview` 中的鼠标跟随预览会放置完整的 1+8 个tile（中心块 + 8个边界变化tile）
- 导致队列预览只显示一个中心块，缺少周围8格的边界变化

**根因代码**（AddQueuePreview 第1076-1082行）：
```csharp
var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
if (previewTiles.ContainsKey(cellPos))
    tile = previewTiles[cellPos];  // ← 只取了中心块！
```

**期望行为**：
- 队列预览应该像鼠标跟随预览一样，放置完整的 1+8 个tile

**验收标准**：
- AC-P2.1：耕地入队后，queuePreviewTilemap 上显示完整的 1+8 个tile（中心块 + 8个边界变化）
- AC-P2.2：多个耕地队列项的边界tile正确叠加（后入队的会影响先入队的边界）

---

### P3：浇水队列预览与实际浇水视觉不一致（严重bug）

**现状问题**：
- `AddQueuePreview` 的 Water 分支使用 `GetRandomPuddleTile()` 随机选择水渍tile
- 但实际浇水后 `FarmVisualManager.UpdateTileVisual` 使用 `tileData.puddleVariant` 索引选择水渍tile
- `puddleVariant` 是在浇水时由 `FarmTileManager` 随机生成并存储到 `FarmTileData` 中的
- 预览时的随机和实际浇水时的随机是两次独立随机，结果大概率不同

**根因分析**：
- 预览阶段：`GetRandomPuddleTile()` → `wetPuddleTiles[Random.Range(0, length)]`
- 实际浇水：`tileData.puddleVariant` → `wetPuddleTiles[Clamp(variant, 0, length-1)]`
- 两者使用不同的随机源，视觉不一致

**期望行为**：
- 浇水队列预览应该与实际浇水后的视觉效果一致
- 方案：在入队时预分配 puddleVariant，预览和实际执行使用同一个 variant

**验收标准**：
- AC-P3.1：浇水队列预览显示的水渍tile与实际浇水后显示的水渍tile完全一致
- AC-P3.2：预分配的 variant 在队列执行时被正确传递给 FarmTileData

---

### P4：种子预览位置与实际种植位置不一致（bug）

**现状问题**：
- 种子预览使用 `PlacementGridCalculator.GetCellCenter(rawWorldPos)` 计算 `alignedPos`
  - 算法：`Mathf.Floor(x) + 0.5f, Mathf.Floor(y) + 0.5f`
- 实际种植使用 `tilemaps.GetCellCenterWorld(cellPos)` 获取世界坐标
  - 算法：`farmlandCenterTilemap.GetCellCenterWorld(cellPos)`（Tilemap 内置方法）
- 预览中 `seedGridRenderer.transform.position = alignedPos` 和 `seedPreviewRenderer.transform.position = alignedPos`
- 如果 Tilemap 的 origin/cellSize 不是标准 (0,0)/(1,1)，两个坐标系会产生偏移

**用户确认**：种植后的位置是对的，预览的位置是错的

**根因**：
- `PlacementGridCalculator.GetCellCenter` 是纯数学计算（假设标准网格），不考虑 Tilemap 的实际配置
- `tilemaps.GetCellCenterWorld` 是 Tilemap 原生方法，考虑了 Grid 的 cellSize、cellGap、transform 偏移
- 两者在非标准网格配置下会产生偏移

**期望行为**：
- 种子预览位置应该与实际种植位置完全一致

**验收标准**：
- AC-P4.1：种子预览（seedGridRenderer + seedPreviewRenderer）的位置与实际种植位置完全重合
- AC-P4.2：修复方案应统一使用 `tilemaps.GetCellCenterWorld(cellPos)` 作为坐标源

---

## 三、优先级

| 编号 | 问题 | 优先级 | 类型 |
|------|------|--------|------|
| P4 | 种子预览位置偏移 | 🔴 高 | bug修复（简单） |
| P2 | 耕地队列预览缺少8+1 | 🔴 高 | bug修复（中等） |
| P3 | 浇水队列预览不一致 | 🔴 高 | bug修复（中等） |
| P1 | 预览系统视觉重构 | 🟡 中 | 重构（复杂） |

---

## 四、约束与注意事项

1. V3 三件套与 V2 独立运行，不覆盖 V2 文档
2. 种子预览保持现有放置系统复刻方案，不受 P1 重构影响
3. 先设计后实现：P1 涉及 shader 设计，需要确认方案后再动手
4. 新增并兼容：不能破坏 V2 已实现的其他功能（延迟执行、时序修复、长按分支等）
