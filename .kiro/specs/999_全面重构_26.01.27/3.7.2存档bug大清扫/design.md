# 动态对象重建系统 - 设计文档

## 独立思考与审视

### 锐评认同的部分

1. **核心问题诊断正确**：动态对象在重启后不存在，GUID 无法匹配
2. **解决方向正确**：需要"重建"能力，从"恢复数据"升级为"恢复存在 + 恢复数据"
3. **prefabId 是必要的**：需要知道用哪个预制体重建

### 我的疑虑与补充

#### 疑虑 1：prefabId 的来源问题

**锐评假设**：TreeController 知道自己的 prefabId
**实际情况**：TreeController 没有存储预制体信息的字段

**解决方案**：在 TreeController 中新增 `prefabId` 字段，在放置时由 PlacementManager 设置

#### 疑虑 2：场景层级问题

**锐评未提及**：重建的对象应该放在哪个层级下？

**解决方案**：在 WorldObjectSaveData 中已有 `layer` 字段，重建时根据 layer 查找对应的 Props 父物体

#### 疑虑 3：预制体存放位置

**实际情况**：预制体在 `Assets/222_Prefabs/Tree/` 目录，不在 Resources 目录
**结论**：必须使用 PrefabRegistry ScriptableObject，不能用 Resources.Load

#### 疑虑 4：TreeSaveData 扩展字段

**锐评要求**：增加 `season`、`isStump`、`stumpHealth`
**我的分析**：
- `season`：Load() 后会被 SeasonManager 覆盖，不需要存档
- `isStump`：可以从 `state` 字段推断
- `stumpHealth`：确实需要，当前缺失

**结论**：只需要增加 `stumpHealth`

---

## 架构概述

### 核心思想

将存档加载逻辑从"恢复数据"升级为"恢复存在 + 恢复数据"：

```
原流程: FindByGuid(guid) → Load(data)
新流程: FindByGuid(guid) → 找不到? → Instantiate(prefab) → Load(data)
```

### 组件关系

```
┌─────────────────────────────────────────────────────────────┐
│                      SaveManager                             │
│  - LoadGame() 调用 Registry.RestoreAllFromSaveData()        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                PersistentObjectRegistry                      │
│  - RestoreAllFromSaveData() 遍历存档数据                     │
│  - 找不到 GUID → 调用 DynamicObjectFactory 重建             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 DynamicObjectFactory (新增)                  │
│  - TryReconstruct(data) 根据 prefabId 实例化对象            │
│  - 依赖 PrefabRegistry 查找预制体                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   PrefabRegistry (新增)                      │
│  - ScriptableObject，存储 prefabId → Prefab 映射            │
│  - 支持编辑器配置和运行时查询                                │
│  - 必须使用（预制体不在 Resources 目录）                     │
└─────────────────────────────────────────────────────────────┘
```

## 详细设计

### 1. 数据结构升级

#### 1.1 WorldObjectSaveData（已有字段，需启用）

```csharp
[Serializable]
public class WorldObjectSaveData
{
    // ... 现有字段 ...
    
    /// <summary>预制体 ID（用于动态重建）</summary>
    public string prefabId;  // 已存在，当前未使用
}
```

#### 1.2 TreeSaveData 扩展

```csharp
[Serializable]
public class TreeSaveData
{
    // ... 现有字段 ...
    
    /// <summary>当前季节（0=春, 1=夏, 2=秋, 3=冬）</summary>
    public int season;
    
    /// <summary>是否为树桩状态</summary>
    public bool isStump;
    
    /// <summary>树桩血量</summary>
    public int stumpHealth;
    
    /// <summary>是否已渐变到下一季节</summary>
    public bool hasTransitionedToNextSeason;
    
    /// <summary>渐变时的植被季节</summary>
    public int transitionVegetationSeason;
}
```

#### 1.3 DropDataDTO（新增 - 锐评006排雷补丁）

```csharp
/// <summary>
/// 掉落物存档数据（用于 genericData 的 JSON 序列化）
/// 🛡️ 封印一：必须加上 [Serializable] 特性
/// </summary>
[Serializable]
public class DropDataDTO
{
    /// <summary>物品 ID</summary>
    public int itemId;
    
    /// <summary>品质等级</summary>
    public int quality;
    
    /// <summary>数量</summary>
    public int amount;
}
```

### 2. PrefabRegistry（新增）

```csharp
/// <summary>
/// 预制体注册表
/// 用于存档系统根据 prefabId 查找预制体
/// </summary>
[CreateAssetMenu(fileName = "PrefabRegistry", menuName = "FarmGame/Data/PrefabRegistry")]
public class PrefabRegistry : ScriptableObject
{
    [Serializable]
    public class PrefabEntry
    {
        public string prefabId;
        public GameObject prefab;
    }
    
    [SerializeField] private List<PrefabEntry> entries = new List<PrefabEntry>();
    
    private Dictionary<string, GameObject> _cache;
    
    /// <summary>
    /// 根据 prefabId 获取预制体
    /// </summary>
    public GameObject GetPrefab(string prefabId)
    {
        if (_cache == null) BuildCache();
        _cache.TryGetValue(prefabId, out var prefab);
        return prefab;
    }
    
    private void BuildCache()
    {
        _cache = new Dictionary<string, GameObject>();
        foreach (var entry in entries)
        {
            if (!string.IsNullOrEmpty(entry.prefabId) && entry.prefab != null)
            {
                _cache[entry.prefabId] = entry.prefab;
            }
        }
    }
}
```

**存放位置**: `Assets/111_Data/Database/PrefabRegistry.asset`

### 3. DynamicObjectFactory（新增）

```csharp
/// <summary>
/// 动态对象工厂
/// 负责根据存档数据重建动态对象
/// </summary>
public static class DynamicObjectFactory
{
    private static PrefabRegistry _registry;
    
    /// <summary>
    /// 初始化工厂（在游戏启动时调用）
    /// </summary>
    public static void Initialize(PrefabRegistry registry)
    {
        _registry = registry;
    }
    
    /// <summary>
    /// 尝试重建动态对象
    /// 🛡️ 封印二：回退逻辑的防腐层
    /// </summary>
    /// <returns>重建的对象，失败返回 null</returns>
    public static IPersistentObject TryReconstruct(WorldObjectSaveData data)
    {
        // === 处理掉落物 ===
        if (data.objectType == "Drop")
        {
            return TryReconstructDrop(data);
        }
        
        // === 处理树木 ===
        if (data.objectType == "Tree")
        {
            return TryReconstructTree(data);
        }
        
        // 其他类型暂不支持重建
        return null;
    }
    
    /// <summary>
    /// 重建掉落物
    /// </summary>
    private static IPersistentObject TryReconstructDrop(WorldObjectSaveData data)
    {
        // 解析 DropDataDTO
        if (string.IsNullOrEmpty(data.genericData))
        {
            Debug.LogWarning($"[DynamicObjectFactory] 掉落物数据为空: guid={data.guid}");
            return null;
        }
        
        var dropData = JsonUtility.FromJson<DropDataDTO>(data.genericData);
        if (dropData == null)
        {
            Debug.LogWarning($"[DynamicObjectFactory] 掉落物数据解析失败: guid={data.guid}");
            return null;
        }
        
        // 使用 WorldSpawnService 重建
        var position = data.GetPosition();
        var pickup = WorldSpawnService.Instance.SpawnById(
            dropData.itemId, 
            dropData.quality, 
            dropData.amount, 
            position
        );
        
        if (pickup == null)
        {
            Debug.LogWarning($"[DynamicObjectFactory] 掉落物生成失败: itemId={dropData.itemId}");
            return null;
        }
        
        // 获取 IPersistentObject 组件并设置 GUID
        var persistentObj = pickup.GetComponent<IPersistentObject>();
        if (persistentObj != null)
        {
            SetPersistentId(persistentObj, data.guid);
        }
        
        return persistentObj;
    }
    
    /// <summary>
    /// 重建树木
    /// 🛡️ 封印二：在执行 Instantiate 之前，必须先校验数据有效性
    /// </summary>
    private static IPersistentObject TryReconstructTree(WorldObjectSaveData data)
    {
        // 解析 TreeSaveData 进行数据验证
        TreeSaveData treeData = null;
        if (!string.IsNullOrEmpty(data.genericData))
        {
            treeData = JsonUtility.FromJson<TreeSaveData>(data.genericData);
        }
        
        // === 数据有效性检查（防腐层）===
        if (treeData != null)
        {
            // 检查是否是已销毁但非树桩的无效数据
            if (treeData.currentHealth <= 0 && !treeData.isStump)
            {
                Debug.LogWarning($"[DynamicObjectFactory] 跳过无效的树木数据（已销毁但非树桩）: guid={data.guid}");
                return null;
            }
            
            // 检查生长阶段有效性
            if (treeData.growthStageIndex < 0)
            {
                Debug.LogWarning($"[DynamicObjectFactory] 跳过无效的树木数据（生长阶段无效）: guid={data.guid}");
                return null;
            }
        }
        
        // === Legacy Fallback：旧存档 prefabId 为空 ===
        string prefabId = data.prefabId;
        if (string.IsNullOrEmpty(prefabId))
        {
            prefabId = "M1";  // 默认使用 M1 预制体
            Debug.LogWarning($"[DynamicObjectFactory] 旧存档兼容：使用默认预制体 M1, guid={data.guid}");
        }
        
        if (_registry == null)
        {
            Debug.LogError("[DynamicObjectFactory] PrefabRegistry 未初始化");
            return null;
        }
        
        // 查找预制体
        var prefab = _registry.GetPrefab(prefabId);
        if (prefab == null)
        {
            Debug.LogWarning($"[DynamicObjectFactory] 找不到预制体: {prefabId}");
            return null;
        }
        
        // 实例化（先禁用，避免闪烁）
        var position = data.GetPosition();
        var instance = Object.Instantiate(prefab, position, Quaternion.identity);
        instance.SetActive(false);  // 🛡️ 封印三：防闪烁
        
        // 获取 IPersistentObject 组件
        var persistentObj = instance.GetComponentInChildren<IPersistentObject>();
        if (persistentObj == null)
        {
            Debug.LogError($"[DynamicObjectFactory] 预制体 {prefabId} 没有 IPersistentObject 组件");
            Object.Destroy(instance);
            return null;
        }
        
        // 强制设置 GUID（关键！）
        SetPersistentId(persistentObj, data.guid);
        
        // 注册到 Registry
        PersistentObjectRegistry.Instance.Register(persistentObj);
        
        return persistentObj;
    }
    
    /// <summary>
    /// 强制设置对象的 PersistentId
    /// </summary>
    private static void SetPersistentId(IPersistentObject obj, string guid)
    {
        if (obj is TreeController tree)
        {
            tree.SetPersistentIdForLoad(guid);
        }
        else if (obj is WorldItemPickup pickup)
        {
            pickup.SetPersistentIdForLoad(guid);
        }
        // 其他类型...
    }
}
```

### 4. TreeController 修改

#### 4.1 Save() 方法增加 prefabId

```csharp
public WorldObjectSaveData Save()
{
    var data = new WorldObjectSaveData
    {
        guid = PersistentId,
        objectType = ObjectType,
        prefabId = GetPrefabId(),  // 新增
        sceneName = gameObject.scene.name,
        isActive = gameObject.activeSelf
    };
    
    // ... 其他代码 ...
}

/// <summary>
/// 获取预制体 ID
/// </summary>
private string GetPrefabId()
{
    // 方案 A: 使用 TreeData 的 ID
    if (treeData != null)
    {
        return $"Tree_{treeData.treeId}";
    }
    
    // 方案 B: 使用 GameObject 名称（去除 (Clone) 后缀）
    string name = gameObject.name.Replace("(Clone)", "").Trim();
    return $"Tree_{name}";
}
```

#### 4.2 新增 SetPersistentIdForLoad 方法

```csharp
/// <summary>
/// 为存档加载设置 PersistentId（仅供 DynamicObjectFactory 调用）
/// </summary>
public void SetPersistentIdForLoad(string guid)
{
    persistentId = guid;
}
```

#### 4.3 TreeSaveData 扩展字段

```csharp
var treeData = new TreeSaveData
{
    growthStageIndex = currentStageIndex,
    currentHealth = this.currentHealth,
    maxHealth = CurrentStageConfig?.health ?? 0,
    daysGrown = daysInCurrentStage,
    state = (int)currentState,
    // 新增字段
    season = (int)currentSeason,
    isStump = currentState == TreeState.Stump,
    stumpHealth = currentStumpHealth
};
```

### 5. PersistentObjectRegistry.RestoreAllFromSaveData 修改

```csharp
public void RestoreAllFromSaveData(List<WorldObjectSaveData> dataList)
{
    if (dataList == null) return;
    
    // ... 现有的匹配率统计 ...
    
    // Step 4: 恢复 (Restoring) - 遍历存档数据进行 Load()
    int restored = 0;
    int notFound = 0;
    int reconstructed = 0;  // 新增：重建计数
    
    foreach (var data in dataList)
    {
        var obj = FindByGuid(data.guid);
        
        if (obj != null)
        {
            // 找到对象，直接恢复
            try
            {
                obj.Load(data);
                restored++;
            }
            catch (Exception e)
            {
                Debug.LogError($"[PersistentObjectRegistry] 恢复对象失败: {data.objectType}, GUID: {data.guid}, 错误: {e.Message}");
            }
        }
        else
        {
            // 🔥 新增：尝试重建动态对象
            var reconstructedObj = DynamicObjectFactory.TryReconstruct(data);
            if (reconstructedObj != null)
            {
                try
                {
                    reconstructedObj.Load(data);
                    reconstructed++;
                    
                    if (showDebugInfo)
                        Debug.Log($"[PersistentObjectRegistry] 重建对象成功: {data.objectType}, GUID: {data.guid}");
                }
                catch (Exception e)
                {
                    Debug.LogError($"[PersistentObjectRegistry] 重建对象后恢复失败: {data.objectType}, GUID: {data.guid}, 错误: {e.Message}");
                }
            }
            else
            {
                notFound++;
                if (showDebugInfo)
                    Debug.LogWarning($"[PersistentObjectRegistry] 找不到对象且无法重建: {data.objectType}, GUID: {data.guid}");
            }
        }
    }
    
    if (showDebugInfo)
        Debug.Log($"[PersistentObjectRegistry] 恢复完成: 成功 {restored}, 重建 {reconstructed}, 未找到 {notFound}, 修剪 {pruned}");
}
```

## 预制体 ID 命名规范

| 对象类型 | prefabId 格式 | 示例 |
|---------|--------------|------|
| 树木 | `M{n}` | `M1`, `M2`, `M3` |
| 石头 | `Stone_{stoneId}` | `Stone_M1`, `Stone_M2` |
| 箱子 | `Chest_{chestType}` | `Chest_Wood`, `Chest_Iron` |
| 掉落物 | 不需要 prefabId | 使用 WorldSpawnService.SpawnById() |

## 初始化流程

```csharp
// GameManager 或 Bootstrap 中
void Start()
{
    // 加载 PrefabRegistry
    var registry = Resources.Load<PrefabRegistry>("PrefabRegistry");
    DynamicObjectFactory.Initialize(registry);
}
```

## 正确性属性

### P1: GUID 唯一性

重建的对象必须使用存档中的 GUID，不能生成新 GUID。

**验证方式**: 保存 → 重启 → 加载 → 再保存，检查 GUID 是否一致。

### P2: 位置一致性

重建的对象位置必须与存档中的位置一致。

**验证方式**: 比较重建后对象的 transform.position 与 saveData.GetPosition()。

### P3: 状态完整性

重建的对象状态必须与保存时一致。

**验证方式**: 比较重建后对象的各项属性与 saveData 中的值。

## 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 预制体被删除/重命名 | 无法重建对象 | 使用稳定的 prefabId，不依赖文件名 |
| 预制体结构变化 | Load() 失败 | 版本兼容性处理，默认值填充 |
| 大量动态对象 | 加载时间长 | 分批加载，进度条显示 |

---

## 🛡️ 质量封印检查清单（执行指令001）

### 封印一：序列化的完整性 (The Serialization Seal)
- [ ] `DropDataDTO` 必须加上 `[Serializable]` 特性
- [ ] 警示：如果漏了 `[Serializable]`，`JsonUtility` 会生成空字符串

### 封印二：回退逻辑的防腐层 (The Corruption Seal)
- [ ] 在执行 `Instantiate` 之前，必须先校验数据有效性
- [ ] `health <= 0 && !isStump` → 坏死数据，直接丢弃
- [ ] `growthStageIndex < 0` → 无效数据，直接丢弃

### 封印三：视觉同步的原子性 (The Visual Seal)
- [ ] `UpdateVisuals()` 必须是 `TreeController.Load()` 的最后一行
- [ ] 实例化时先 `SetActive(false)`，Load 后再 `SetActive(true)`
