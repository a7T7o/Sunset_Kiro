# 农田系统重生 - 设计文档

**版本**: 9.0.1
**创建时间**: 2026-02-05

---

## 一、架构概述

### 1.1 当前问题

```
┌─────────────────────────────────────────────────────────────┐
│                    当前架构（有问题）                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FarmTileSaveData          CropController                   │
│  ┌─────────────┐           ┌─────────────┐                  │
│  │ cropId      │ ←─重复──→ │ seedId      │                  │
│  │ growthStage │ ←─重复──→ │ currentStage│                  │
│  │ daysGrown   │ ←─重复──→ │ grownDays   │                  │
│  └─────────────┘           └─────────────┘                  │
│        ↑                         ↑                          │
│        │                         │                          │
│   没有 Save/Load            没有 Save/Load                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 目标架构

```
┌─────────────────────────────────────────────────────────────┐
│                    目标架构（清晰分离）                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FarmTileManager              CropController                │
│  ┌─────────────┐              ┌─────────────┐               │
│  │ 耕地数据    │              │ 作物数据    │               │
│  │ - 位置      │              │ - 种子ID    │               │
│  │ - 浇水状态  │              │ - 生长阶段  │               │
│  │ - 土壤状态  │              │ - 枯萎状态  │               │
│  └──────┬──────┘              └──────┬──────┘               │
│         │                            │                      │
│         ↓                            ↓                      │
│  IPersistentObject            IPersistentObject             │
│  ┌─────────────┐              ┌─────────────┐               │
│  │ Save()      │              │ Save()      │               │
│  │ Load()      │              │ Load()      │               │
│  └─────────────┘              └─────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、组件设计

### 2.1 FarmTileManager 改造

**新增接口实现**：

```csharp
public class FarmTileManager : MonoBehaviour, IPersistentObject
{
    // ===== IPersistentObject 实现 =====
    
    public string PersistentId => "FarmTileManager";
    
    public WorldObjectSaveData Save()
    {
        var saveData = new WorldObjectSaveData
        {
            guid = PersistentId,
            objectType = "FarmTileManager"
        };
        
        // 序列化所有耕地数据
        var farmTiles = new List<FarmTileSaveData>();
        foreach (var layerKvp in farmTilesByLayer)
        {
            foreach (var tileKvp in layerKvp.Value)
            {
                var tile = tileKvp.Value;
                farmTiles.Add(new FarmTileSaveData
                {
                    tileX = tile.cellPosition.x,
                    tileY = tile.cellPosition.y,
                    layer = tile.layerIndex,
                    soilState = (int)tile.moistureState,
                    isWatered = tile.wateredToday
                });
            }
        }
        
        saveData.genericData = JsonUtility.ToJson(new FarmTileListWrapper { tiles = farmTiles });
        return saveData;
    }
    
    public void Load(WorldObjectSaveData data)
    {
        // 清空现有数据
        ClearAllTiles();
        
        // 反序列化耕地数据
        var wrapper = JsonUtility.FromJson<FarmTileListWrapper>(data.genericData);
        foreach (var tileData in wrapper.tiles)
        {
            var cellPos = new Vector3Int(tileData.tileX, tileData.tileY, 0);
            
            // 创建耕地数据
            CreateTileFromSaveData(tileData.layer, cellPos, tileData);
        }
    }
}
```

**新增防御性检查**：

```csharp
public bool CreateTile(int layerIndex, Vector3Int cellPosition)
{
    // ===== P0：强制配置检查 =====
    var tilemaps = GetLayerTilemaps(layerIndex);
    if (tilemaps == null)
    {
        Debug.LogError($"[FarmTileManager] CreateTile 失败: 楼层 {layerIndex} 的 Tilemap 配置为空");
        return false;
    }
    
    if (tilemaps.groundTilemap == null)
    {
        Debug.LogError($"[FarmTileManager] CreateTile 失败: 楼层 {layerIndex} 的 groundTilemap 未配置");
        return false;
    }
    
    // ... 后续逻辑
}
```

### 2.2 CropController 改造

**新增接口实现**：

```csharp
public class CropController : MonoBehaviour, IPersistentObject
{
    // ===== IPersistentObject 实现 =====
    
    [SerializeField] private string persistentId;
    
    public string PersistentId => persistentId;
    
    public WorldObjectSaveData Save()
    {
        var saveData = new WorldObjectSaveData
        {
            guid = persistentId,
            objectType = "Crop",
            prefabId = seedData?.itemID.ToString() ?? "-1"
        };
        
        saveData.SetPosition(transform.position);
        
        // 序列化作物数据
        var cropData = new CropSaveData
        {
            seedId = seedData?.itemID ?? -1,
            currentStage = instanceData?.currentStage ?? 0,
            grownDays = instanceData?.grownDays ?? 0,
            daysWithoutWater = instanceData?.daysWithoutWater ?? 0,
            isWithered = instanceData?.isWithered ?? false,
            quality = instanceData?.quality ?? 0,
            harvestCount = instanceData?.harvestCount ?? 0,
            lastHarvestDay = instanceData?.lastHarvestDay ?? 0,
            layerIndex = layerIndex,
            cellX = cellPosition.x,
            cellY = cellPosition.y
        };
        
        saveData.genericData = JsonUtility.ToJson(cropData);
        return saveData;
    }
    
    public void Load(WorldObjectSaveData data)
    {
        var cropData = JsonUtility.FromJson<CropSaveData>(data.genericData);
        
        // 恢复位置信息
        layerIndex = cropData.layerIndex;
        cellPosition = new Vector3Int(cropData.cellX, cropData.cellY, 0);
        
        // 恢复生长状态
        instanceData = new CropInstanceData(cropData.seedId, 0)
        {
            currentStage = cropData.currentStage,
            grownDays = cropData.grownDays,
            daysWithoutWater = cropData.daysWithoutWater,
            isWithered = cropData.isWithered,
            quality = cropData.quality,
            harvestCount = cropData.harvestCount,
            lastHarvestDay = cropData.lastHarvestDay
        };
        
        // 向 FarmTileManager 注册占用
        var farmTileManager = FarmTileManager.Instance;
        if (farmTileManager != null)
        {
            var tileData = farmTileManager.GetTileData(layerIndex, cellPosition);
            if (tileData != null)
            {
                tileData.SetCrop(instanceData);
            }
        }
        
        isInitialized = true;
        UpdateVisuals();
    }
}
```

### 2.3 SaveDataDTOs 改造

**新增 CropSaveData**：

```csharp
/// <summary>
/// 作物存档数据（存储在 WorldObjectSaveData.genericData 中）
/// </summary>
[Serializable]
public class CropSaveData
{
    public int seedId;
    public int currentStage;
    public int grownDays;
    public int daysWithoutWater;
    public bool isWithered;
    public int quality;
    public int harvestCount;
    public int lastHarvestDay;
    
    // 位置信息
    public int layerIndex;
    public int cellX;
    public int cellY;
}

/// <summary>
/// 耕地列表包装器（用于 JSON 序列化）
/// </summary>
[Serializable]
public class FarmTileListWrapper
{
    public List<FarmTileSaveData> tiles;
}
```

**标记废弃字段**：

```csharp
[Serializable]
public class FarmTileSaveData
{
    public int tileX;
    public int tileY;
    public int layer = 1;
    public int soilState;
    public bool isWatered;
    
    // ===== 废弃字段（保留用于兼容旧存档）=====
    [Obsolete("作物数据已迁移到 CropController，此字段仅用于兼容旧存档")]
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

---

## 三、调用链详解

### 3.1 保存流程

```
SaveManager.SaveGame()
│
├─→ PersistentObjectRegistry.SaveAll()
│   │
│   ├─→ FarmTileManager.Save()
│   │   ├─ 遍历 farmTilesByLayer
│   │   ├─ 生成 FarmTileSaveData 列表
│   │   └─ 序列化到 genericData
│   │
│   └─→ CropController.Save() (每个作物)
│       ├─ 生成 CropSaveData
│       └─ 序列化到 genericData
│
└─→ 写入 JSON 文件
```

### 3.2 加载流程

```
SaveManager.LoadGame()
│
├─→ 读取 JSON 文件
│
├─→ PersistentObjectRegistry.LoadAll()
│   │
│   ├─→ FarmTileManager.Load()
│   │   ├─ 清空现有数据
│   │   ├─ 反序列化 FarmTileSaveData 列表
│   │   ├─ 创建耕地数据
│   │   └─ 放置 Tilemap Tile
│   │
│   └─→ DynamicObjectFactory.CreateCrop() (每个作物)
│       ├─ 实例化 Crop 预制体
│       └─→ CropController.Load()
│           ├─ 反序列化 CropSaveData
│           ├─ 恢复生长状态
│           └─ 向 FarmTileManager 注册占用
│
└─→ 完成
```

---

## 四、正确性属性

### 4.1 配置防御属性

| 属性 | 描述 |
|------|------|
| P1 | groundTilemap 为空时，CreateTile 必须返回 false |
| P2 | groundTilemap 为空时，CanTillAt 必须返回 false |
| P3 | 配置缺失时，必须输出 LogError |

### 4.2 存档属性

| 属性 | 描述 |
|------|------|
| P4 | Save 后 Load，耕地数量必须相等 |
| P5 | Save 后 Load，耕地位置必须相等 |
| P6 | Save 后 Load，浇水状态必须相等 |

### 4.3 作物属性

| 属性 | 描述 |
|------|------|
| P7 | Save 后 Load，作物数量必须相等 |
| P8 | Save 后 Load，作物生长阶段必须相等 |
| P9 | Load 后，作物必须向 FarmTileManager 注册占用 |

---

## 五、测试策略

### 5.1 单元测试

1. **配置防御测试**
   - groundTilemap 为空时 CreateTile 返回 false
   - groundTilemap 为空时 CanTillAt 返回 false

2. **序列化测试**
   - FarmTileSaveData 序列化/反序列化
   - CropSaveData 序列化/反序列化

### 5.2 集成测试

1. **存档流程测试**
   - 创建耕地 → 保存 → 加载 → 验证耕地存在
   - 种植作物 → 保存 → 加载 → 验证作物存在

2. **兼容性测试**
   - 加载旧存档（包含废弃字段）→ 验证不报错

---

## 六、风险评估

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 旧存档不兼容 | 玩家数据丢失 | 保留 [Obsolete] 字段，支持旧存档加载 |
| 作物注册失败 | 作物与耕地不关联 | Load 时检查 FarmTileManager 是否存在 |
| Tilemap 未刷新 | 耕地不显示 | Load 后调用 FarmlandBorderManager 刷新 |

---

## 七、相关文件

| 文件 | 修改内容 |
|------|---------|
| `FarmTileManager.cs` | 实现 IPersistentObject，添加防御检查 |
| `CropController.cs` | 实现 IPersistentObject |
| `SaveDataDTOs.cs` | 新增 CropSaveData，标记废弃字段 |
| `GameInputManager.cs` | 添加详细日志 |
