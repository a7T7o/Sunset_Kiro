# 农田系统重生 - 需求文档

**版本**: 9.0.1
**创建时间**: 2026-02-05
**来源**: Code Reaper 锐评034 执行令

---

## 背景

用户反馈"锄地不显示任何内容"，经过代码审查和锐评分析，发现以下问题：

1. **配置防御缺失**：`LayerTilemaps.IsValid()` 不检查 `groundTilemap`，可能导致"虚空锄地"
2. **作物数据双重存储**：`FarmTileSaveData` 和 `CropInstanceData` 存储相同数据
3. **存档系统未接入**：`FarmTileManager` 和 `CropController` 没有实现 `IPersistentObject`

---

## 用户故事

### US-1：配置防御（P0）

**作为** 开发者
**我希望** 农田系统在配置缺失时显式报错并阻断逻辑
**以便** 快速定位配置问题，避免"虚空锄地"

**验收标准**:
- AC-1.1：`FarmTileManager.CreateTile()` 在 `groundTilemap` 为空时返回 false 并输出错误日志
- AC-1.2：`FarmTileManager.CanTillAt()` 在 `groundTilemap` 为空时返回 false
- AC-1.3：`GameInputManager.TryTillSoil()` 输出详细日志，明确是 `CanTillAt` 失败还是 `CreateTile` 失败

---

### US-2：FarmTileManager 存档集成（P1）

**作为** 玩家
**我希望** 耕地数据在保存/加载时正确恢复
**以便** 重启游戏后耕地状态不丢失

**验收标准**:
- AC-2.1：`FarmTileManager` 实现 `IPersistentObject` 接口
- AC-2.2：`Save()` 方法将所有耕地数据序列化为 `FarmTileSaveData` 列表
- AC-2.3：`Load()` 方法从 `FarmTileSaveData` 列表恢复耕地数据
- AC-2.4：加载后 Tilemap 正确显示耕地 Tile

---

### US-3：作物身份统一化（P1）

**作为** 开发者
**我希望** 作物数据只存储在 `CropController` 中
**以便** 消除数据双重存储，简化架构

**验收标准**:
- AC-3.1：`FarmTileSaveData` 中的 `cropId`, `cropGrowthStage`, `cropQuality`, `daysGrown`, `daysWithoutWater` 字段标记为 `[Obsolete]`
- AC-3.2：`CropController` 实现 `IPersistentObject` 接口
- AC-3.3：`CropController.Save()` 存储作物生长状态
- AC-3.4：`CropController.Load()` 恢复作物生长状态并向 `FarmTileManager` 注册占用
- AC-3.5：加载后作物 GameObject 正确显示

---

## 数据结构变更

### FarmTileSaveData（变更后）

```csharp
[Serializable]
public class FarmTileSaveData
{
    public int tileX;
    public int tileY;
    public int layer = 1;
    public int soilState;           // 土地状态
    public bool isWatered;          // 是否已浇水
    
    // ===== 以下字段标记为 [Obsolete] =====
    [Obsolete("作物数据已迁移到 CropController")]
    public int cropId = -1;
    [Obsolete("作物数据已迁移到 CropController")]
    public int cropGrowthStage;
    [Obsolete("作物数据已迁移到 CropController")]
    public int cropQuality;
    [Obsolete("作物数据已迁移到 CropController")]
    public int daysGrown;
    [Obsolete("作物数据已迁移到 CropController")]
    public int daysWithoutWater;
}
```

### CropSaveData（新增）

```csharp
[Serializable]
public class CropSaveData
{
    public int seedId;              // 种子 ID
    public int currentStage;        // 当前生长阶段
    public int grownDays;           // 已生长天数
    public int daysWithoutWater;    // 连续未浇水天数
    public bool isWithered;         // 是否枯萎
    public int quality;             // 品质
    public int harvestCount;        // 已收获次数
    public int lastHarvestDay;      // 上次收获日期
    
    // 位置信息
    public int layerIndex;          // 楼层索引
    public int cellX;               // 格子 X
    public int cellY;               // 格子 Y
}
```

---

## 调用链设计

### 保存流程

```
SaveManager.Save()
  → FarmTileManager.Save()
      → 遍历 farmTilesByLayer
      → 生成 List<FarmTileSaveData>
  → CropController.Save() (每个作物)
      → 生成 CropSaveData
      → 存入 WorldObjectSaveData.genericData
```

### 加载流程

```
SaveManager.Load()
  → FarmTileManager.Load()
      → 从 List<FarmTileSaveData> 恢复耕地数据
      → 放置 Tilemap Tile
  → CropController.Load() (每个作物)
      → 从 CropSaveData 恢复生长状态
      → 向 FarmTileManager 注册占用
```

---

## 优先级

| 优先级 | 用户故事 | 说明 |
|--------|---------|------|
| P0 | US-1 配置防御 | 修复当前 bug |
| P1 | US-2 存档集成 | 必须完成 |
| P1 | US-3 作物统一化 | 必须完成 |

---

## 约束

1. **兼容性**：旧存档必须能够加载（通过 [Obsolete] 字段兼容）
2. **不破坏现有功能**：修改不能影响其他系统
3. **参考模式**：参考 `TreeController` 的 `IPersistentObject` 实现

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `Assets/YYY_Scripts/Farm/FarmTileManager.cs` | 耕地管理器 |
| `Assets/YYY_Scripts/Farm/CropController.cs` | 作物控制器 |
| `Assets/YYY_Scripts/Farm/Data/LayerTilemaps.cs` | 楼层 Tilemap 配置 |
| `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` | 存档数据结构 |
| `Assets/YYY_Scripts/Controller/Input/GameInputManager.cs` | 输入管理器 |
