# 补丁005 设计文档 — 种子并入放置系统 + 房子不标红修复

> 工作区：10.1.5补丁005
> 基于：补丁005V1.md（事实核查通过）
> 创建时间：2026-02-25
> 状态：待审核

---

## 一、补丁概述

| 任务 | 描述 | 复杂度 | 独立性 |
|------|------|--------|--------|
| 任务A | 修复耕地预览对障碍物（房子等）不标红的视觉路由 bug | 小 | 完全独立，先行修复 |
| 任务B | 种子从 FIFO 队列彻底剥离，接入放置系统 | 中-大 | 主体工程 |

---

## 二、任务A：修复房子不标红视觉路由 bug

### 2.1 问题根因

文件：`FarmToolPreview.cs` → `UpdateHoePreview`

```
hasObstacle = PlacementValidator.HasFarmingObstacle(cellCenter)  // 房子 → true
canTill = FarmTileManager.Instance.CanTillAt(layerIndex, cellPos) // 有地面tile → true
isValid = !hasObstacle && (canTill || hasCrop)                    // false
```

视觉路由分支1：`if (canTill && ...)` → 进入 1+8 绿色预览（错误）
应该进入分支3（红方框），因为 `isValid=false`。

### 2.2 修复方案

视觉路由条件修改：
- 分支1：`canTill` → `isValid && !hasCrop`
- Shader 染色：`canTill` → `isValid && !hasCrop`

### 2.3 场景验证矩阵

| 场景 | 修改前 | 修改后 |
|------|--------|--------|
| 空地（可耕种） | 1+8绿色 ✅ | 1+8绿色 ✅ |
| 房子下方 | 1+8绿色 ❌ | 红方框 ✅ |
| 已有作物 | 空白 ✅ | 空白 ✅ |
| 石头/树木 | 红方框 ✅ | 红方框 ✅ |
| 已在队列中 | 红方框 ✅ | 红方框 ✅ |

### 2.4 涉及文件

仅 `FarmToolPreview.cs` 的 `UpdateHoePreview` 方法。

---

## 三、任务B：种子从 FIFO 剥离并接入放置系统

### 3.1 架构决策

种子走 FIFO 是架构错位：
- FIFO 核心价值 = 串行化有动画的异步操作
- 种子是唯一的同步递归类型（无动画、直接执行、递归 `ProcessNextAction`）
- 种子预览系统（SpriteRenderer）与耕地预览（Tilemap）完全不同
- 放置系统已有完整基础设施，树苗是成功案例

### 3.2 类型系统分析（锐评排雷结果）

代码事实（当前会话已验证）：
- `SeedData` 继承自 `ItemData`（不是 `PlaceableItemData`）
- `isPlaceable` 是 `ItemData` 基类的 `public bool` 字段
- `EnterPlacementMode(ItemData item)` 接收 `ItemData` 类型，无强转
- `ExecutePlacement` 中用 `is PlaceableItemData` 安全模式匹配，不匹配则跳过
- 结论：`SeedData` 设 `isPlaceable=true` 后可安全进入放置系统，无类型强转风险

### 3.3 六层适配方案

#### 3.3.1 数据层：SeedData 标记可放置

`SeedData.cs`：在 `OnValidate` 中设置 `isPlaceable = true`。
`HotbarSelectionService` 自动走 `isPlaceable` 路由调用 `PlacementManager.EnterPlacementMode`。

#### 3.3.2 路由层：移除 SeedData 拦截

`PlacementManager.cs` → `EnterPlacementMode`：删除 `if (item is SeedData) return;` 拦截块。

#### 3.3.3 验证层：种子专用放置验证

`PlacementValidator.cs`：新增 `ValidateSeedPlacement(SeedData, Vector3 worldPos)` 方法。

**layerIndex 获取方式**（🔴 锐评002 排雷）：
- 方法内部调用 `FarmTileManager.Instance.GetCurrentLayerIndex(worldPos)` 获取楼层索引
- 再通过 `GetLayerTilemaps(layerIndex)` 获取 tilemaps，计算 cellPos
- 与 `FarmToolPreview.UpdateSeedPreview` 中的获取方式完全同源
- **注意**：`PlacementLayerDetector.GetLayerAtPosition` 返回的是 Unity 物理 Layer（如 "LAYER 1" 的 int 编号），与农田系统的 `layerIndex`（楼层索引 0/1/2）完全不同，不可混用

验证项：
1. 目标位置有耕地数据（`tileData != null`）
2. 耕地可种植（`tileData.CanPlant()`）
3. 无障碍物（`!HasFarmingObstacle`）
4. 季节匹配（`IsCorrectSeason`）

选择方案α（`PlacementValidator` 新增方法）而非方案β（`SeedData` 重写 `CanPlaceAt`），原因：
- 种子验证需要农田系统的 `layerIndex`，`CanPlaceAt` 只接收 `Vector3`
- 种子验证与农田系统强耦合，放在 `PlacementValidator` 更合适
- 保持 `SeedData` 作为纯数据类

`PlacementManager` 中：`UpdatePreview` 和 `ExecutePlacement` 检测 `SeedData` 类型走专用验证。

#### 3.3.4 预览层：种子预览迁移

`PlacementPreview.cs` → `Show(ItemData item)`：检测 `SeedData` 时：
- 从 `seedData.cropPrefab` 的 `CropController` 获取第一阶段 sprite 作为预览图
- 底部对齐偏移复用 `AlignSpriteBottom` 模式
- 格子方框颜色由 `PlacementGridCell` 的 valid/invalid 状态控制（已有机制）

关键适配：种子没有 `placementPrefab`，需要从 `cropPrefab` 获取预览 sprite。

#### 3.3.5 执行层：种子专用放置执行

`PlacementManager.cs`：新增 `ExecuteSeedPlacement(SeedData)` 方法。

在 `ExecutePlacement` 中增加种子专用分支：
```
if (currentPlacementItem is SeedData seedData)
{
    ExecuteSeedPlacement(seedData);
    return;
}
```

`ExecuteSeedPlacement` 核心逻辑（从 `GameInputManager.ExecutePlantSeed` 完整迁移）：
1. **layerIndex 获取**（🔴 锐评002 排雷）：`int layerIndex = FarmTileManager.Instance.GetCurrentLayerIndex(position);` → `var tilemaps = GetLayerTilemaps(layerIndex)` → `Vector3Int cellPos = tilemaps.WorldToCell(position)`
2. `FarmTileManager.Instance.GetTileData(layerIndex, cellPos)` 获取耕地数据
2. `tileData.CanPlant()` 二次验证
3. 季节检查
4. `SeedBagHelper.ConsumeSeed` 消耗种子（保质期链路）
5. `Instantiate(seedData.cropPrefab)` 实例化作物
6. `CropController.Initialize` 初始化
7. `tileData.SetCropData(instanceData)` 绑定耕地数据

与通用放置的差异（不走通用路径的原因）：
- 不走 `InstantiatePlacementPrefab`（用 `cropPrefab` 不是 `placementPrefab`）
- 物品扣除走 `SeedBagHelper`（种子袋保质期链路），不是 `inventory.RemoveItem`
- 需要 `CropController.Initialize` + `tileData.SetCropData` 农田数据绑定
- 不需要 `SyncLayerToPlacedObject`（作物 Layer 由 `CropController.Initialize` 内部处理）
- 不需要放置历史（不支持撤销）

#### 3.3.6 导航层：复用放置系统导航

放置系统已有完整导航流程：`OnLeftClick` → `LockPreviewPosition` → `StartNavigation` → `OnNavigationReached` → `ExecutePlacement`。种子接入后自动复用，无需额外适配。

#### 3.3.7 连续放置：种子用完检测

`PlacementManager.cs` → `HasMoreItems`：增加种子袋检测分支。

```
if (currentPlacementItem is SeedData)
{
    var seedItem = inventoryService.GetInventoryItem(currentSnapshot.slotIndex);
    return seedItem != null && !seedItem.IsEmpty 
        && SeedBagHelper.GetRemaining(seedItem) > 0;
}
```

### 3.4 FIFO 触点清理清单（24个触点）

| 编号 | 触点 | 文件 | 处理方式 |
|------|------|------|----------|
| T1 | `FarmActionType.PlantSeed` 枚举值 | GameInputManager.cs | 保留（标记 Obsolete） |
| T2 | `TryPlantSeed` 方法 | GameInputManager.cs | 删除 |
| T3 | `ExecutePlantSeed` 方法 | GameInputManager.cs | 逻辑迁移后删除 |
| T4 | `TryEnqueueSeed` 方法 | GameInputManager.cs | 删除 |
| T5 | `HandleUseCurrentTool` 种子分支 | GameInputManager.cs | 删除 |
| T6 | `TryEnqueueFromCurrentInput` 种子分支 | GameInputManager.cs | 删除 |
| T7 | `ProcessNextAction` PlantSeed 二次验证 | GameInputManager.cs | 删除 |
| T8 | `ExecuteFarmAction` PlantSeed 分支 | GameInputManager.cs | 删除 |
| T9 | `HasSeedRemaining` 方法 | GameInputManager.cs | 保留（可能被引用） |
| T10 | `UpdateSeedPreview` (GIM 中转) | GameInputManager.cs | 删除 |
| T11 | `UpdatePreviews` 种子路由分支 | GameInputManager.cs | 改为走放置系统路由 |
| T12 | `_cachedSeedData` 字段 | GameInputManager.cs | 删除 |
| T13 | `UpdateSeedPreview` (FTP 实现) | FarmToolPreview.cs | 删除 |
| T14 | `AddQueuePreview` 种子分支 | FarmToolPreview.cs | 删除 |
| T15 | `RemoveQueuePreview` 种子分支 | FarmToolPreview.cs | 删除 |
| T16 | `ClearAllQueuePreviews` 种子回收 | FarmToolPreview.cs | 删除 |
| T17 | `PromoteToExecutingPreview` 种子分支 | FarmToolPreview.cs | 删除 |
| T18 | `RemoveExecutingPreview` 种子分支 | FarmToolPreview.cs | 删除 |
| T19 | 种子预览字段（5个） | FarmToolPreview.cs | 删除 |
| T20 | `GetOrCreateSeedQueueRenderer` 方法 | FarmToolPreview.cs | 删除 |
| T21 | `UpdateCursorForSeed` 方法 | FarmToolPreview.cs | 删除 |
| T22 | `isSeedMode` 字段 | FarmToolPreview.cs | 删除 |
| T23 | `EnterPlacementMode` SeedData 拦截 | PlacementManager.cs | 删除 |
| T24 | `HotbarSelectionService.EquipCurrentTool` | HotbarSelectionService.cs | 无需改动（自动路由） |

### 3.5 FIFO 清理后自然消失的 Bug

- `queuePreviewPositions` 残留 → 幽灵占位（种子不再进入队列）
- 种子预览 SpriteRenderer 残留（种子不再使用 FTP 对象池）

### 3.6 涉及文件总表

| 文件 | 修改类型 | 说明 |
|------|----------|------|
| `FarmToolPreview.cs` | 修改 | 任务A：视觉路由修复 |
| `SeedData.cs` | 修改 | `isPlaceable = true` |
| `PlacementManager.cs` | 修改+新增 | 移除拦截 + `ExecuteSeedPlacement` |
| `PlacementValidator.cs` | 新增 | `ValidateSeedPlacement` 方法 |
| `PlacementPreview.cs` | 修改 | `Show` 增加 SeedData 预览分支 |
| `GameInputManager.cs` | 删除 | 移除种子相关 10 个触点 |
| `FarmToolPreview.cs` | 删除 | 移除种子预览系统 10 个触点 |

---

## 四、风险评估

| 风险 | 等级 | 缓解措施 |
|------|------|----------|
| 种子袋保质期链路迁移遗漏 | 🟡 中 | `ExecuteSeedPlacement` 完整复用 `ExecutePlantSeed` 逻辑 |
| 放置系统预览与种子预览视觉差异 | 🟢 低 | `PlacementGridCell` 已有颜色机制 |
| FIFO 清理误删共用逻辑 | 🟡 中 | 逐触点清理，每个独立验证 |
| 连续种植手感变化 | 🟡 中 | 放置系统 Preview→Lock→Execute 循环与 FIFO 连续入队不同，需测试 |

---

## 五、待审核确认点

1. 任务A 修复方案（`canTill` → `isValid && !hasCrop`）是否认可？
2. 种子验证采用方案α（`PlacementValidator` 新增方法）是否认可？
3. `FarmActionType.PlantSeed` 枚举值保留（标记 Obsolete）是否认可？
4. 连续种植交互变化（从"连续入队批量执行"变为"逐个放置"）是否接受？
