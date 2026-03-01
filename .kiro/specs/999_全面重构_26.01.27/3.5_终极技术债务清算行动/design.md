# Phase 3.5 终极技术债务清算行动 - 设计文档

## 架构概述

本次重构涉及三个独立的战役，每个战役有明确的边界和目标。

---

## 战役 A: 数据层净化

### 设计目标

实现 Inspector 面板的动态字段显示，同时保留数据验证的安全网。

### 组件设计

#### 1. ItemData.cs（保留 OnValidate）

```
职责：
- 存储物品基础数据
- 提供核心数据验证（OnValidate）

OnValidate 保留的验证：
- ID 范围检查（0-9999）
- 名称非空检查
- 图标非空检查
- 价格逻辑检查（售价 < 买价）
- 放置配置完整性检查
```

#### 2. ItemDataEditor.cs（新建）

```
位置：Assets/Editor/Inspectors/ItemDataEditor.cs
职责：
- 根据 category 动态显示/隐藏字段
- 提供更友好的 Inspector 体验

字段分组：
┌─────────────────────────────────────────┐
│ Basic Info（始终显示）                   │
│ - itemID, itemName, description, category│
├─────────────────────────────────────────┤
│ Visuals（始终显示）                      │
│ - icon, bagSprite, worldPrefab          │
├─────────────────────────────────────────┤
│ Economy（始终显示）                      │
│ - buyPrice, sellPrice, maxStackSize     │
├─────────────────────────────────────────┤
│ Equipment Config（仅 Equipment 显示）    │
│ - equipmentType                         │
├─────────────────────────────────────────┤
│ Placement Config（仅 Placeable 显示）    │
│ - isPlaceable, placementType, etc.      │
├─────────────────────────────────────────┤
│ Consumable Config（仅 Consumable 显示）  │
│ - consumableType                        │
└─────────────────────────────────────────┘
```

### 类图

```
ItemData (ScriptableObject)
├── OnValidate() [保留核心验证]
└── 字段分组...

ItemDataEditor (UnityEditor.Editor)
├── OnInspectorGUI()
│   ├── DrawBasicInfo()
│   ├── DrawVisuals()
│   ├── DrawEconomy()
│   ├── DrawEquipmentConfig() [条件显示]
│   ├── DrawPlacementConfig() [条件显示]
│   ├── DrawConsumableConfig() [条件显示]
│   └── DrawRemainingProperties() [★ 子类兼容]
└── GetCategoryFromProperty()
```

### 子类兼容性设计（关键！）

```
问题：如果 EquipmentData 等子类将来需要独特的 Inspector 逻辑，
      ItemDataEditor 会因为继承关系强制覆盖子类的编辑器。

解决方案：在 OnInspectorGUI 末尾绘制所有未被手动处理的属性

代码示例：
```csharp
private void DrawRemainingProperties(SerializedObject so)
{
    // 获取所有已处理的属性名
    var handledProperties = new HashSet<string>
    {
        "m_Script", "itemID", "itemName", "description", "category",
        "icon", "bagSprite", "worldPrefab", "buyPrice", "sellPrice",
        "maxStackSize", "equipmentType", "isPlaceable", "placementType",
        // ... 其他已处理的属性
    };
    
    // 遍历所有属性，绘制未处理的
    var iterator = so.GetIterator();
    bool enterChildren = true;
    while (iterator.NextVisible(enterChildren))
    {
        enterChildren = false;
        if (!handledProperties.Contains(iterator.name))
        {
            EditorGUILayout.PropertyField(iterator, true);
        }
    }
}
```

效果：
- 子类新增的字段会自动显示在 Inspector 底部
- 不会因为父类 Editor 而"吞掉"子类字段
- 子类可以创建自己的 CustomEditor 覆盖父类
```

---

## 战役 B: 存档系统补完

### 设计目标

1. 排除不应保存的玩家子物体（Tool）
2. 让 TreeController 支持持久化
3. 读档后强制刷新 UI

### 组件设计

#### 1. SaveManager.cs 修改

```
修改点 1：CollectPlayerData() 中排除 Tool
- 检测子物体名称是否包含 "Tool"
- 如果是，跳过该子物体

修改点 2：LoadGame() 末尾添加 UI 刷新
- 调用 InventoryPanelUI.Instance?.Refresh()
- 调用 ToolbarUI.Instance?.Refresh()（如果存在）
```

#### 2. TreeController.cs 修改

```
新增接口：IPersistentObject

需要实现的成员：
- PersistentId: string（使用 persistentId 字段）
- ObjectType: string（返回 "Tree"）
- ShouldSave: bool（返回 gameObject.activeInHierarchy）
- Save(): WorldObjectSaveData
- Load(WorldObjectSaveData data): void

存档数据结构：
{
    guid: string,
    objectType: "Tree",
    customData: {
        stageIndex: int,
        health: int,
        stumpHealth: int,
        state: TreeState,
        season: Season,
        plantedDay: int,
        daysInCurrentStage: int
    }
}

Load 后处理：
- 调用 UpdateStageVisual() 立即刷新视觉
- 调用 UpdateCollider() 更新碰撞体
```

#### 2.1 GUID 初始化策略（关键！）

```
★ 场景预置树（Pre-placed）：
- 在 OnValidate() 中检查 persistentId
- 如果为空或重复，自动生成新 GUID
- 仅在编辑器模式下执行

★ 动态生成树（Runtime）：
- 在 Awake() 中检查 persistentId
- 如果为空（动态生成的树），生成新 GUID
- 树苗长成树时，由 SaplingController 传递 GUID

代码示例：
```csharp
[SerializeField] private string persistentId;

#if UNITY_EDITOR
private void OnValidate()
{
    // 仅在编辑器模式下自动生成 GUID
    if (string.IsNullOrEmpty(persistentId))
    {
        persistentId = System.Guid.NewGuid().ToString();
        UnityEditor.EditorUtility.SetDirty(this);
    }
}
#endif

private void Awake()
{
    // 运行时检查（动态生成的树）
    if (string.IsNullOrEmpty(persistentId))
    {
        persistentId = System.Guid.NewGuid().ToString();
    }
}
```

★ 编辑器工具（可选）：
- 菜单项：Tools/Tree/Generate All GUIDs
- 遍历场景中所有 TreeController
- 为空 GUID 的树生成新 GUID
- 标记场景为 dirty
```

### 序列图

```
SaveGame 流程：
┌─────────┐    ┌─────────────┐    ┌──────────────────┐
│SaveMgr  │    │PersistentReg│    │TreeController    │
└────┬────┘    └──────┬──────┘    └────────┬─────────┘
     │                │                     │
     │ CollectAll()   │                     │
     │───────────────>│                     │
     │                │ Save()              │
     │                │────────────────────>│
     │                │<────────────────────│
     │                │   WorldObjectData   │
     │<───────────────│                     │
     │  List<Data>    │                     │

LoadGame 流程：
┌─────────┐    ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐
│SaveMgr  │    │PersistentReg│    │TreeController    │    │InventoryUI │
└────┬────┘    └──────┬──────┘    └────────┬─────────┘    └──────┬──────┘
     │                │                     │                     │
     │ RestoreAll()   │                     │                     │
     │───────────────>│                     │                     │
     │                │ Load(data)          │                     │
     │                │────────────────────>│                     │
     │                │                     │ UpdateStageVisual() │
     │                │                     │─────────┐           │
     │                │                     │<────────┘           │
     │<───────────────│                     │                     │
     │                │                     │                     │
     │ Refresh()      │                     │                     │
     │────────────────────────────────────────────────────────────>│
```

---

## 战役 C: 农田工具距离检测

### 设计目标

在农田工具逻辑中添加距离检测，修复"站在耕地上却浇不到水"的 Bug。

### 组件设计

#### GameInputManager.cs 修改

```
新增配置字段：
[Header("农田工具设置")]
[SerializeField] private float farmToolReach = 1.5f;

修改方法：TryTillSoil(Vector3 worldPosition)
- 在现有逻辑前添加距离检测
- 计算玩家中心到目标格子中心的距离
- 超出 farmToolReach 时返回 false

修改方法：TryWaterTile(Vector3 worldPosition)
- 同上

距离计算方式：
1. 获取玩家 Collider 中心（GetPlayerCenter()）
2. 获取目标格子世界中心（tilemaps.GetCellCenterWorld(cellPosition)）
3. 计算 Vector2.Distance（忽略 Z 轴）
4. 比较 distance <= farmToolReach
```

### 代码示例

```csharp
private bool TryTillSoil(Vector3 worldPosition)
{
    var farmTileManager = FarmGame.Farm.FarmTileManager.Instance;
    if (farmTileManager == null) return false;
    
    int layerIndex = farmTileManager.GetCurrentLayerIndex(worldPosition);
    var tilemaps = farmTileManager.GetLayerTilemaps(layerIndex);
    if (tilemaps == null) return false;
    
    Vector3Int cellPosition = tilemaps.WorldToCell(worldPosition);
    
    // ★ 新增：距离检测
    Vector2 playerCenter = GetPlayerCenter();
    Vector3 cellCenter = tilemaps.GetCellCenterWorld(cellPosition);
    float distance = Vector2.Distance(playerCenter, cellCenter);
    
    if (distance > farmToolReach)
    {
        if (showDebugInfo)
            Debug.Log($"[GameInputManager] 锄地失败：距离过远 {distance:F2} > {farmToolReach}");
        return false;
    }
    
    // 现有逻辑...
    if (!farmTileManager.CanTillAt(layerIndex, cellPosition))
        return false;
    
    return farmTileManager.CreateTile(layerIndex, cellPosition);
}
```

---

## 测试策略

### 战役 A 测试

1. 创建不同类型的 ItemData SO（Equipment、Seed、Placeable）
2. 验证 Inspector 面板只显示相关字段
3. 使用批量工具修改 SO，验证 OnValidate 仍然触发

### 战役 B 测试

1. 保存游戏，检查存档文件中是否包含 Tree 数据
2. 修改树木状态（砍伐、成长），保存后读档
3. 验证树木视觉状态立即恢复
4. 验证 Tool 子物体不在存档中

### 战役 C 测试

1. 站在耕地旁边，使用锄头/水壶
2. 站在远离耕地的位置，使用锄头/水壶（应失败）
3. 验证箱子交互、NPC 对话等功能不受影响

---

## 风险评估

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| Custom Editor 导致 Inspector 卡顿 | 低 | 使用 SerializedProperty 缓存 |
| TreeController 持久化数据不完整 | 中 | 详细测试所有树木状态 |
| 距离检测影响游戏体验 | 低 | farmToolReach 可配置 |
