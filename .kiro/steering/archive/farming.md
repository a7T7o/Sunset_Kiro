---
inclusion: manual
priority: P4
keywords: [农田, 种植, 作物, 浇水, 耕地, FarmTileData, FarmTileManager, CropController, PlacementManager]
lastUpdated: 2026-02-14
archivedFrom: .kiro/steering/farming.md
archivedDate: 2026-01-09
---

# 农田系统规范

> 基于 9.0.1-10.X 实际代码更新（2026-02-14）
> 10.X 纠正：CropManager 已废弃，播种直接 Instantiate seedData.cropPrefab，收获改掉落模式

## 一、核心架构

### 模块职责

| 模块 | 职责 |
|------|------|
| FarmTileManager | 耕地数据管理（多层级 Dictionary、浇水、每日更新） |
| PlacementManager | 统一放置系统（农田/箱子/树苗共用，状态机驱动） |
| PlacementGridCalculator | 网格计算（世界坐标与格子坐标转换） |
| PlacementValidator | 放置验证（障碍物、Layer、农田检测） |
| PlacementLayerDetector | Layer 检测（地面/Sorting Layer/玩家 Layer） |
| FarmToolPreview | 农具预览（锄头/水壶/种子三种模式） |
| PlacementNavigator | 放置导航（ClosestPoint 到达检测） |
| PlayerAutoNavigator | 自动寻路（路径规划、障碍物绕行） |
| FarmlandBorderManager | 耕地边界（15 种边界 Tile + 阴影 Tile） |
| CropController | 作物控制器（状态机、生长、收获掉落、枯萎、存档，完全自治） |
| ~~CropManager~~ | [已废弃] 原集中式管理器，10.X 纠正后不再使用 |

### 数据流

```
播种链路（10.X 纠正后）：
  GameInputManager.ExecutePlantSeed()
    → seedData.cropPrefab（每种种子自己的预制体）
    → Instantiate(cropPrefab, worldPos, container)
    → CropController.Initialize(seedData, instanceData, layerIndex, cellPos)
    → FarmTileData.SetCropData() + FarmTileData.cropController = this

收获链路（10.X 纠正后）：
  IInteractable.OnInteract() → CropController.Harvest()
    → HarvestMature(): WorldSpawnService.SpawnMultiple(dropItemData, quality, amount, pos)
    → HarvestWitheredMature(): SpawnMultiple(witheredDropItemData, 0, 1, pos)

存档重建：
  DynamicObjectFactory.TryReconstructCrop()
    → seedData.cropPrefab → Instantiate → CropController.Load()
```

## 二、多层级系统

每个楼层（LAYER 1/2/3）有独立的 LayerTilemaps：
- farmlandCenterTilemap：耕地中心块
- farmlandBorderTilemap：耕地边界装饰
- waterPuddleTilemapNew：水渍叠加效果
- groundTilemap：地面检测
- propsContainer：作物 GameObject 容器

层级索引：0=LAYER 1, 1=LAYER 2, 2=LAYER 3

## 三、FarmTileData（耕地格子数据）

```csharp
public class FarmTileData
{
    public Vector3Int position;    // 格子坐标
    public int layerIndex;         // 楼层索引
    public bool isTilled;          // 是否已耕作
    public bool wateredToday;      // 今天浇水（记录，第二天生效）
    public bool wateredYesterday;  // 昨天浇水（影响作物生长）
    public float waterTime;        // 浇水时间（小时）
    public SoilMoistureState moistureState;
    public int puddleVariant;      // 水渍变体（0-2）
    public CropInstanceData cropData; // 作物数据（新版）
    
    // 🔥 10.X 纠正新增
    [NonSerialized]
    public CropController cropController; // 运行时控制器引用（替代 CropManager.GetCrop）
}
```

状态查询：CanPlant() = isTilled 且无作物，HasCrop() = cropData 非空
作物查找：通过 `cropController` 字段直接获取（替代已废弃的 CropManager.GetCrop）

## 四、土壤水分系统

SoilMoistureState：Dry / WetWithPuddle（浇水后2h内） / WetDark（2h后）

每日更新（OnDayChanged）：wateredYesterday=wateredToday, wateredToday=false, moistureState=Dry

## 五、放置系统（PlacementManager 状态机）

PlacementState：Idle / Previewing / Locked / Navigating / Placing

操作流程：
1. 选择物品 -> EnterPlacementMode() -> Previewing
2. 左键点击 -> LockPreviewPosition() -> Locked
3. 距离判断：近则直接 ExecutePlacement()，远则 StartNavigation() -> Navigating
4. 导航到达 -> OnNavigationReached() -> ExecutePlacement()
5. 右键取消 -> ExitPlacementMode() -> Idle

中断机制：切换快捷栏、物品用完、外部调用 InterruptFromExternal() 都会退出放置模式。

## 六、预览系统（FarmToolPreview）

FarmPreviewState：None / ValidHoe / InvalidHoe / ValidWatering / InvalidWatering / ValidSeed / InvalidSeed

三种预览模式：
- UpdateHoePreview()：检查是否可耕作（groundTilemap 有 Tile、未耕作、无障碍物、距离内）
- UpdateWateringPreview()：检查是否可浇水（已耕作、未浇水、距离内）
- UpdateSeedPreview()：检查是否可种植（已耕作、无作物、季节匹配、距离内）

预览使用 Ghost Tilemap 显示半透明效果，光标跟随鼠标位置。

## 七、导航系统

PlacementNavigator：
- CalculateNavigationTarget()：用 Bounds.ClosestPoint 计算预览格子边缘最近点
- CheckReached()：玩家中心到预览格子边缘距离 <= triggerDistance（0.8f）
- 超时保护：navigationTimeout = 10s

PlayerAutoNavigator：
- SetDestination()：设置目标点，自动规划路径
- 障碍物绕行：射线检测 + 方向旋转
- 卡住检测：连续位移过小时自动重新规划
- 玩家位置使用 playerCollider.bounds.center（遵守最高优先级规则）

## 八、作物系统

### CropState 状态机

Growing -> Mature -> （收获后）ReGrowing 或销毁
Growing -> WitheredImmature -> （清除）
Mature -> WitheredMature -> （收获枯萎作物）

### CropInstanceData（纯数据）

```csharp
public class CropInstanceData
{
    public int seedDataID;         // 种子 ID
    public int currentStage;       // 当前生长阶段
    public int plantedDay;         // 种植天数
    public int grownDays;          // 实际生长天数
    public int daysWithoutWater;   // 连续未浇水天数
    public bool isWithered;        // 是否枯萎
    public int harvestCount;       // 已收获次数
    public int lastHarvestDay;     // 上次收获天数
    public int quality;            // 品质
}
```

### SeedData（种子 SO）

核心字段：season、isReHarvestable、reHarvestDays、seedsPerBag、cropPrefab（作物预制体引用）

> 🔥 10.X 纠正 + 字段精简：以下字段已从代码中删除：
> - `growthDays`（已被 CropStageConfig.daysToNextStage 替代，直接删除）
> - `harvestCropID`（已被 CropController.dropItemData 替代，直接删除）
> - `harvestAmountRange`（已被 CropController.dropAmount 替代，直接删除）
> - `growthStageSprites[]` 和 `witheredStageSprites[]`（10.0.2 已移除，Sprite 迁移到 Prefab）
>
> 生长天数显示（Tooltip）不再由 SeedData 负责，改由 Controller 配置。

ID 范围：种子 10XX（1000-1099），作物 11XX（1100-1149），枯萎作物 11XX（1150-1199）

### CropStageConfig（作物阶段配置）

```csharp
[System.Serializable]
public struct CropStageConfig
{
    public Sprite normalSprite;    // 该阶段正常 Sprite
    public Sprite witheredSprite;  // 该阶段枯萎 Sprite（可为空，向前回退查找）
    [Range(0, 30)]
    public int daysToNextStage;    // 到下一阶段需要的天数（最后阶段填0）
}
```

配置在 CropController Prefab 的 Inspector 上，参考 TreeController 的 StageConfig 模式。
CropController 通过 `stages[]` 数组读取 Sprite，不再依赖 SeedData。

固定 4 阶段：种子(0) → 幼苗(1) → 生长(2) → 成熟(3)，对应常量 CROP_STAGE_SEED/SPROUT/GROWING/MATURE。

生长天数计算：累加模式（非均分）。grownDays 与 stages[i].daysToNextStage 逐阶段累加比较，确定当前阶段。例如配置 [1,2,2,0] 表示种子1天→幼苗2天→生长2天→成熟。

### 收获机制（10.X 纠正后）

CropController 上配置掉落字段（Inspector）：
- `dropItemData`：收获掉落的 ItemData SO
- `dropAmount`：掉落数量（默认1）
- `dropSpreadRadius`：散布半径（默认0.4f）
- `dropQuality`：品质（默认0=Normal）
- `witheredDropItemData`：枯萎收获掉落的 ItemData SO（可为空）

收获流程：
1. 成熟收获：`WorldSpawnService.SpawnMultiple(dropItemData, dropQuality, dropAmount, position, dropSpreadRadius)`
2. 枯萎收获：`SpawnMultiple(witheredDropItemData, 0, 1, position, dropSpreadRadius)`
3. 可重复收获：重置到 reGrowStage，继续生长
4. 不可重复收获：DestroyCrop()（清除 FarmTileData + 销毁 GameObject）

> 🔥 10.X 纠正：收获从"直接加背包"改为"掉落到地面"，与树木系统统一
> 旧的 DetermineHarvestQuality() 随机品质已废弃，改由 dropQuality 字段控制

### CropData（作物 SO，继承 FoodData）

核心字段：（无活跃专属字段）

> 🔥 已废弃字段（标记 `[Obsolete]`，保留仅为存档兼容）：
> - `seedID`：对应种子 ID（数据流改为单向：SeedData → cropPrefab → CropController → dropItemData）
> - `witheredCropID`：对应枯萎作物 ID（枯萎作物通过 CropController.witheredDropItemData 配置）
> - `harvestExp`：收获经验值（经验值统一归 SeedData.harvestingExp）

### WitheredCropData（枯萎作物 SO，继承 FoodData）

核心字段：（无活跃专属字段）

> 🔥 已废弃的反向引用字段（标记 `[Obsolete]`，保留仅为存档兼容）：
> - `seedID`：对应种子 ID（同 CropData，消除三向绑定）
>
> 🔥 已删除字段：
> - `normalCropID`：对应正常作物 ID（无引用，直接删除）

### 种子袋系统（SeedBagHelper）

动态属性存储在 InventoryItem 上：
- bag_opened：是否已打开
- bag_remaining：袋内剩余种子数
- shelf_expire_day：过期天数

打开种子袋后保质期重算：Min(剩余天数, 打开后最大保质期)

## 九、耕地边界系统（FarmlandBorderManager）

自动计算 15 种边界 Tile（上下左右 + 四角 + 组合），外加阴影 Tile。

工作流程：
1. 耕地创建 -> OnCenterBlockPlaced() -> 放置中心块 + 更新周围边界
2. 耕地移除 -> OnCenterBlockRemoved() -> 移除中心块 + 更新周围边界
3. 预览时 -> GetPreviewTiles() 返回预计变化的 Tile（不实际修改）

## 十、存档集成

FarmTileManager 实现 Save()/Load()：
- Save()：遍历所有层级的 FarmTileData，序列化为 FarmTileSaveData 列表
- Load()：清空现有数据，从 FarmTileSaveData 重建（CreateTileFromSaveData）

CropController 实现 Save()/Load()：
- 使用 genericData + JSON 序列化 CropSaveData
- Save() 写入 cropDataId、growthStage、grownDays、isWatered、isWithered 等
- Load() 从 genericData 反序列化恢复状态

> 🔥 10.0.2 已完成：DynamicObjectFactory 新增 "Crop" 分支，通过 seedId → ItemDatabase → SeedData.cropPrefab 实例化作物 Prefab，再调用 CropController.Load() 恢复状态

## 十一、Inspector 显示规范

`ItemDataEditor`（自定义 Editor，`[CustomEditor(typeof(ItemData), true)]`）通过 `DrawRemainingProperties()` 绘制子类专属字段。

**废弃字段自动隐藏**：`DrawRemainingProperties()` 使用反射检测 `[System.Obsolete]` 标记，自动跳过废弃字段的 Inspector 绘制。无需手动添加 `[HideInInspector]`。

当前被隐藏的废弃字段：
- CropData：`seedID`、`witheredCropID`、`harvestExp`
- WitheredCropData：`seedID`

> 注意：`[Obsolete]` 字段仍存在于序列化数据中（存档兼容），只是不在 Inspector 中显示。
> SeedData 的 `growthDays`、`harvestCropID`、`harvestAmountRange` 已从代码中直接删除（非 Obsolete）。
> WitheredCropData 的 `normalCropID` 已从代码中直接删除。

## 十二、已知问题与待定事项

1. ✅ 10.0.2 重构已完成（Prefab 驱动模式，SeedData 精简，CropStageConfig 新增）
2. ✅ 10.X 纠正已完成：CropManager 废弃、播种链路重写、收获改掉落模式、SeedData 字段精简
3. CropController 的 Initialize 有新旧两个重载，旧版标记 [Obsolete]
4. FarmTileData 中旧版 CropInstance 字段标记 [Obsolete]，保留兼容
5. LayerTilemaps 中旧版 farmlandTilemap/waterPuddleTilemap 标记 [Obsolete]
6. ~~CropManager~~ 整个类已标记 [Obsolete]，不再使用
7. Collect 动画集成待后续迭代（当前收获通过 IInteractable 直接触发，无动画过渡）
