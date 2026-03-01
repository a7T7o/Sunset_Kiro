# 10.0.2 农作物视觉层 Prefab 驱动重构 — 设计文档

## 一、架构概览

### 改动前（当前）

```
SeedData (SO)
├── growthStageSprites[]     ← 生长阶段 Sprite
├── witheredStageSprites[]   ← 枯萎阶段 Sprite
├── growthDays / season / ...  ← 数值配置
└── (无 cropPrefab)

CropController (Prefab 上的 MonoBehaviour)
├── 读取 seedData.growthStageSprites[index] 显示 Sprite
├── 读取 seedData.witheredStageSprites[index] 显示枯萎
└── 业务逻辑（状态机/收获/枯萎/存档）
```

### 改动后（目标）

```
SeedData (SO) — 精简版
├── cropPrefab               ← 新增：引用作物预制体
├── growthDays / season / ...  ← 数值配置（不变）
└── (删除 growthStageSprites / witheredStageSprites)

CropController (Prefab 上的 MonoBehaviour) — 自包含版
├── CropStageConfig[] stages  ← 新增：阶段配置数组（Inspector 上配置）
│   ├── [0] normalSprite + witheredSprite  (种子阶段)
│   ├── [1] normalSprite + witheredSprite  (小苗阶段)
│   ├── [2] normalSprite + witheredSprite  (成长阶段)
│   └── [3] normalSprite + witheredSprite  (成熟阶段)
├── 读取 stages[index].normalSprite 显示 Sprite
├── 读取 stages[index].witheredSprite 显示枯萎
└── 业务逻辑（状态机/收获/枯萎/存档）← 完全不动
```

## 二、新增数据结构

### CropStageConfig 结构体

```csharp
[System.Serializable]
public struct CropStageConfig
{
    [Tooltip("该阶段的正常 Sprite")]
    public Sprite normalSprite;

    [Tooltip("该阶段的枯萎 Sprite（可为空，为空时使用最后一个非空的枯萎 Sprite）")]
    public Sprite witheredSprite;
}
```

设计决策：
- 使用 struct 而非 class，与 TreeController 的 StageConfig 保持一致
- witheredSprite 允许为空（幼苗阶段可能没有独立枯萎图）
- 未来可扩展更多字段（粒子特效引用、碰撞体大小等）

### 放置位置

新建文件 `Assets/YYY_Scripts/Farm/Data/CropStageConfig.cs`，与 CropInstanceData 同目录。

## 三、CropController 改动详情

### 3.1 新增字段

```csharp
[Header("=== 阶段 Sprite 配置 ===")]
[Tooltip("每个生长阶段的 Sprite 配置（在 Prefab Inspector 上配置）")]
[SerializeField] private CropStageConfig[] stages;
```

### 3.2 修改 GetCurrentSprite()

改动前：从 `seedData.growthStageSprites` 读取
改动后：从 `stages[]` 读取

逻辑：
1. 枯萎状态 → 查找 `stages[currentStage].witheredSprite`，如果为空则向前查找最近的非空枯萎 Sprite
2. 正常状态 → 返回 `stages[currentStage].normalSprite`

### 3.3 修改 UpdateGrowthStage()

改动前：`int totalStages = seedData.growthStageSprites?.Length ?? 1;`
改动后：`int totalStages = stages?.Length ?? 1;`

### 3.4 修改 IsMatureStage()

改动前：`return instanceData.currentStage >= seedData.growthStageSprites.Length - 1;`
改动后：`return stages != null && stages.Length > 0 && instanceData.currentStage >= stages.Length - 1;`

### 3.5 修改 HarvestMature() 中的重复收获阶段计算

改动前：`int reGrowStage = Mathf.Max(0, (seedData.growthStageSprites?.Length ?? 1) - 2);`
改动后：`int reGrowStage = Mathf.Max(0, (stages?.Length ?? 1) - 2);`

### 3.6 修改 GetTotalStages()

改动前：`return seedData?.growthStageSprites?.Length ?? 0;`
改动后：`return stages?.Length ?? 0;`

### 3.7 修改 ResetForReHarvest() 中的阶段计算

改动前：`int totalStages = seedData.growthStageSprites?.Length ?? 1;`
改动后：`int totalStages = stages?.Length ?? 1;`

### 3.8 不动的部分

以下方法/逻辑完全不动：
- CropState 枚举
- IsValidTransition 静态方法
- TryTransitionState
- OnDayChanged / HandleGrowingDay / HandleMatureDay
- OnSeasonChanged
- Harvest / HarvestMature（除重复收获阶段计算外）/ HarvestWitheredMature
- DetermineHarvestQuality
- DropItemToWorld
- ClearWitheredImmature / DestroyCrop
- Initialize（两个重载）
- RestoreStateFromData
- Save / Load / PersistentId 相关
- IInteractable 实现
- 成熟闪烁 Update 逻辑

## 四、SeedData 改动详情

### 4.1 删除字段

```csharp
// 删除以下两个字段
public Sprite[] growthStageSprites;
public Sprite[] witheredStageSprites;
```

### 4.2 新增字段

```csharp
[Header("=== 作物预制体 ===")]
[Tooltip("作物预制体（包含 CropController + 阶段 Sprite 配置）")]
public GameObject cropPrefab;
```

### 4.3 修改 OnValidate()

- 移除 growthStageSprites 长度验证
- 新增 cropPrefab 非空验证
- 新增 cropPrefab 上是否有 CropController 组件的验证

## 五、CropManager 改动

### 5.1 TryHarvest 中的重复收获阶段计算

改动前：`int reGrowStage = Mathf.Max(1, seedData.growthStageSprites.Length - 3);`
改动后：从 controller 获取总阶段数 `int reGrowStage = Mathf.Max(1, controller.GetTotalStages() - 3);`

## 六、CropInstance（Obsolete）改动

CropInstance 已标记 `[Obsolete]`，其中的 `IsMature()` 和 `GetCurrentSprite()` 引用了 `seedData.growthStageSprites`。

方案：在这些方法上也标记 `[Obsolete]` 并添加编译警告抑制，保持编译通过但不修改逻辑。
由于 CropInstance 整个类已废弃，不值得为它适配新结构。改为让它在编译时不报错即可。

## 七、DynamicObjectFactory 改动

### 7.1 新增 TryReconstructCrop 方法

```
流程：
1. 解析 CropSaveData（从 genericData）
2. 获取 seedId → 通过 ItemDatabase 找到 SeedData
3. 使用 SeedData.cropPrefab 实例化作物 GameObject
4. 设置位置、GUID
5. 调用 CropController.Load() 恢复完整状态
6. 注册到 PersistentObjectRegistry
```

### 7.2 TryReconstruct 新增分支

在 "Chest" 分支后新增 "Crop" 分支。

## 八、Tool_BatchItemSOGenerator 改动

### 8.1 DrawSeedSettings()

- 新增 cropPrefab 的 ObjectField 绘制
- 当前工具中没有 Sprite 数组的绘制逻辑（CreateSeedData 中也没有设置），所以无需删除

### 8.2 CreateSeedData()

- 新增 `data.cropPrefab = seedCropPrefab;` 赋值

## 九、存档兼容性分析

| 存档字段 | 改动前 | 改动后 | 影响 |
|---------|--------|--------|------|
| seedId | int | int | 无变化 |
| currentStage | int | int | 无变化 |
| grownDays | int | int | 无变化 |
| isWithered | bool | bool | 无变化 |
| genericData | JSON | JSON | 无变化 |
| prefabId | seedData.itemID.ToString() | seedData.itemID.ToString() | 无变化 |
| objectType | "Crop" | "Crop" | 无变化 |

结论：存档格式完全不变，旧存档可以正常加载。

## 十、正确性属性

### P1：阶段配置完整性
对于任意 CropController，如果 stages 数组非空且长度 > 0，
则 GetTotalStages() == stages.Length。

### P2：Sprite 读取一致性
对于任意有效的 currentStage（0 <= currentStage < stages.Length），
GetCurrentSprite() 在非枯萎状态下返回 stages[currentStage].normalSprite。

### P3：枯萎 Sprite 回退
对于枯萎状态，如果 stages[currentStage].witheredSprite 为空，
GetCurrentSprite() 应返回最近的非空 witheredSprite（向前查找）。

### P4：业务逻辑不变性
状态转换规则（IsValidTransition）在重构前后完全一致。
（此属性由现有 16 个 PBT 测试保护）

### P5：存档往返一致性
Save() 产生的数据经过 Load() 后，CropController 的状态与 Save 前一致。
（此属性由现有存档测试保护）

## 十一、测试框架

使用 NUnit（Unity Test Runner EditMode），与现有 CropSystemTests.cs 保持一致。
