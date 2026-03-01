# 自动化存档系统 - 设计文档

**创建日期**: 2026-02-03  
**版本**: 1.0

---

## 一、架构概览

```
存档系统架构
├── PrefabDatabase（新增）
│   ├── 预制体文件夹配置
│   ├── 自动扫描机制
│   ├── 智能查找（精确 → 模糊 → 回退）
│   └── 编辑器工具
│
├── DynamicObjectFactory（改进）
│   ├── 使用 PrefabDatabase 替代 PrefabRegistry
│   └── 增强的回退逻辑
│
├── 各对象 Save() 方法（改进）
│   ├── ChestController - 使用 storagePrefab.name
│   ├── TreeController - 保持不变
│   └── StoneController - 保持不变
│
└── 编辑器自动化（新增）
    ├── PrefabDatabaseEditor
    └── PrefabDatabaseAutoScanner
```

---

## 二、组件设计

### 2.1 PrefabDatabase

**文件位置**：`Assets/YYY_Scripts/Data/Core/PrefabDatabase.cs`

```csharp
namespace FarmGame.Data.Core
{
    [CreateAssetMenu(fileName = "PrefabDatabase", menuName = "FarmGame/Data/PrefabDatabase")]
    public class PrefabDatabase : ScriptableObject
    {
        #region 序列化字段
        
        [Header("预制体文件夹")]
        [Tooltip("自动扫描这些文件夹下的所有预制体")]
        [SerializeField] private string[] prefabFolders;
        
        [Header("运行时数据")]
        [SerializeField] private List<PrefabEntry> entries;
        
        [Header("ID 别名映射（旧存档兼容）")]
        [SerializeField] private List<AliasEntry> aliases;
        
        [Header("调试")]
        [SerializeField] private bool showDebugInfo;
        
        #endregion
        
        #region 内部类
        
        [Serializable]
        public class PrefabEntry
        {
            public string name;           // 预制体名称（作为 ID）
            public GameObject prefab;     // 预制体引用
            public string folderPath;     // 来源文件夹（用于分组显示）
        }
        
        [Serializable]
        public class AliasEntry
        {
            public string oldId;          // 旧存档中的 ID
            public string newPrefabName;  // 新系统中的预制体名称
            public string note;           // 备注说明
        }
        
        #endregion
        
        #region 公开方法
        
        /// <summary>
        /// 根据名称获取预制体（支持智能回退）
        /// </summary>
        public GameObject GetPrefab(string prefabName);
        
        /// <summary>
        /// 解析预制体 ID（支持别名映射）
        /// </summary>
        public string ResolvePrefabId(string saveId);
        
        /// <summary>
        /// 检查预制体是否存在
        /// </summary>
        public bool HasPrefab(string prefabName);
        
        /// <summary>
        /// 获取所有已注册的预制体名称
        /// </summary>
        public IEnumerable<string> GetAllPrefabNames();
        
        /// <summary>
        /// 获取已注册预制体数量
        /// </summary>
        public int EntryCount { get; }
        
        #endregion
        
        #region 编辑器方法
        
        /// <summary>
        /// 扫描所有配置的文件夹
        /// </summary>
        public void ScanPrefabs();
        
        /// <summary>
        /// 清空所有条目
        /// </summary>
        public void ClearEntries();
        
        #endregion
    }
}
```

### 2.2 ID 别名映射系统（锐评001 补充）

**问题背景**：旧存档中的 prefabId（如 `M1`、`Storage_1400_小木箱子_0`）与新系统自动扫描的预制体名称（如 `Tree_M1_00`、`Box_1`）不匹配，会导致加载失败。

**解决方案**：在 PrefabDatabase 中添加 ID 别名映射系统。

```csharp
#region ID 别名映射

[Header("ID 别名映射（旧存档兼容）")]
[SerializeField] private List<AliasEntry> aliases;

[Serializable]
public class AliasEntry
{
    [Tooltip("旧存档中的 ID")]
    public string oldId;
    
    [Tooltip("新系统中的预制体名称")]
    public string newPrefabName;
    
    [Tooltip("备注说明")]
    public string note;
}

/// <summary>
/// 解析预制体 ID（支持别名映射）
/// </summary>
public string ResolvePrefabId(string saveId)
{
    // 1. 如果数据库里直接有，直接返回
    if (HasPrefab(saveId)) return saveId;
    
    // 2. 查找别名映射
    var alias = aliases?.Find(a => a.oldId == saveId);
    if (alias != null && HasPrefab(alias.newPrefabName))
    {
        if (showDebugInfo)
            Debug.Log($"[PrefabDatabase] ID 别名映射: {saveId} → {alias.newPrefabName}");
        return alias.newPrefabName;
    }
    
    // 3. 前缀匹配回退（通用规则）
    string fallback = TryPrefixFallback(saveId);
    if (fallback != null && HasPrefab(fallback))
    {
        Debug.LogWarning($"[PrefabDatabase] 前缀回退: {saveId} → {fallback}");
        return fallback;
    }
    
    // 4. 返回原始 ID（让后续逻辑处理失败情况）
    return saveId;
}

/// <summary>
/// 前缀匹配回退（通用规则）
/// </summary>
private string TryPrefixFallback(string saveId)
{
    // Storage_ 开头 → 尝试 Box_1
    if (saveId.StartsWith("Storage_")) return "Box_1";
    
    // 其他规则可以在这里添加
    return null;
}

#endregion
```

**默认别名配置**（在 PrefabDatabase.asset 中配置）：

| 旧 ID | 新预制体名称 | 备注 |
|-------|-------------|------|
| `M1` | `Tree_M1_00` | 树木类型 1 |
| `M2` | `Tree_M2_00` | 树木类型 2 |
| `M3` | `Tree_M3_00` | 树木类型 3 |
| `Chest` | `Box_1` | 默认箱子 |

### 2.3 智能查找算法

```
GetPrefab(prefabName) 流程：

1. 解析 ID（通过 ResolvePrefabId）
   ├── 直接匹配 → 返回原 ID
   ├── 别名映射 → 返回映射后的 ID
   └── 前缀回退 → 返回回退 ID
   
2. 精确匹配
   └── _cache.TryGetValue(resolvedId, out prefab)
   
3. 清洗名称后匹配
   ├── 去掉 "(Clone)" 后缀
   ├── 去掉 " (1)", " (2)" 等后缀
   └── _cache.TryGetValue(cleanName, out prefab)
   
4. 返回 null（所有尝试都失败）
```

### 2.3 ChestController.Save() 改进

**修改点**：`prefabId` 使用世界预制体名称

```csharp
public WorldObjectSaveData Save()
{
    var data = new WorldObjectSaveData
    {
        guid = PersistentId,
        objectType = ObjectType,
        // 🔥 关键修改：使用世界预制体名称
        prefabId = GetWorldPrefabName(),
        // ...
    };
    return data;
}

/// <summary>
/// 获取世界预制体名称
/// </summary>
private string GetWorldPrefabName()
{
    // 1. 优先使用 StorageData 中配置的世界预制体
    if (storageData != null && storageData.storagePrefab != null)
    {
        return storageData.storagePrefab.name;
    }
    
    // 2. 回退：使用当前 GameObject 名称
    return CleanGameObjectName(gameObject.name);
}

/// <summary>
/// 清理 GameObject 名称（去掉 Clone 等后缀）
/// </summary>
private string CleanGameObjectName(string name)
{
    // 去掉 "(Clone)" 后缀
    if (name.EndsWith("(Clone)"))
        name = name.Substring(0, name.Length - 7).Trim();
    
    // 去掉 " (1)", " (2)" 等后缀
    name = Regex.Replace(name, @"\s\(\d+\)$", "");
    
    return name;
}
```

### 2.4 DynamicObjectFactory 改进

**修改点**：使用 PrefabDatabase 替代 PrefabRegistry

```csharp
public static class DynamicObjectFactory
{
    private static PrefabDatabase _database;  // 替代 _registry
    
    public static void Initialize(PrefabDatabase database)
    {
        _database = database;
        _initialized = true;
    }
    
    private static IPersistentObject TryReconstructChest(WorldObjectSaveData data)
    {
        // 使用 PrefabDatabase 的智能查找
        var prefab = _database.GetPrefab(data.prefabId);
        
        if (prefab == null)
        {
            Debug.LogWarning($"[DynamicObjectFactory] 找不到箱子预制体: {data.prefabId}");
            return null;
        }
        
        // ... 实例化逻辑
    }
}
```

---

## 三、编辑器工具设计

### 3.1 PrefabDatabaseEditor

**文件位置**：`Assets/Editor/PrefabDatabaseEditor.cs`

```csharp
[CustomEditor(typeof(PrefabDatabase))]
public class PrefabDatabaseEditor : Editor
{
    private bool _showPrefabList = true;
    private Vector2 _scrollPosition;
    
    public override void OnInspectorGUI()
    {
        var database = (PrefabDatabase)target;
        
        // 1. 绘制文件夹配置
        DrawFolderConfig();
        
        // 2. 扫描按钮
        EditorGUILayout.Space();
        if (GUILayout.Button("🔍 扫描预制体", GUILayout.Height(30)))
        {
            database.ScanPrefabs();
            EditorUtility.SetDirty(database);
        }
        
        // 3. 统计信息
        EditorGUILayout.HelpBox(
            $"已注册 {database.EntryCount} 个预制体",
            MessageType.Info
        );
        
        // 4. 预制体列表（可折叠）
        _showPrefabList = EditorGUILayout.Foldout(_showPrefabList, "预制体列表");
        if (_showPrefabList)
        {
            DrawPrefabList(database);
        }
    }
    
    private void DrawPrefabList(PrefabDatabase database)
    {
        // 按文件夹分组显示
        // 每个条目显示：名称、预制体引用、来源文件夹
    }
}
```

### 3.2 PrefabDatabaseAutoScanner

**文件位置**：`Assets/Editor/PrefabDatabaseAutoScanner.cs`

```csharp
public class PrefabDatabaseAutoScanner : AssetPostprocessor
{
    private static readonly string[] WatchedFolders = new[]
    {
        "Assets/222_Prefabs/Tree",
        "Assets/222_Prefabs/Rock",
        "Assets/222_Prefabs/Box",
        "Assets/222_Prefabs/WorldItems"
    };
    
    static void OnPostprocessAllAssets(
        string[] importedAssets,
        string[] deletedAssets,
        string[] movedAssets,
        string[] movedFromAssetPaths)
    {
        // 检查是否有预制体变化
        bool prefabChanged = CheckPrefabChanged(importedAssets, deletedAssets);
        
        if (prefabChanged)
        {
            // 延迟扫描
            EditorApplication.delayCall += TriggerScan;
        }
    }
    
    private static void TriggerScan()
    {
        var database = AssetDatabase.LoadAssetAtPath<PrefabDatabase>(
            "Assets/111_Data/Database/PrefabDatabase.asset"
        );
        
        if (database != null)
        {
            database.ScanPrefabs();
            EditorUtility.SetDirty(database);
            Debug.Log("[PrefabDatabaseAutoScanner] 自动扫描完成");
        }
    }
}
```

---

## 四、数据流

### 4.1 保存流程

```
ChestController.Save()
    │
    ├── prefabId = GetWorldPrefabName()
    │   └── 返回 "Box_1"（从 storagePrefab.name 获取）
    │
    └── 生成 WorldObjectSaveData
        └── prefabId = "Box_1"
```

### 4.2 加载流程

```
DynamicObjectFactory.TryReconstructChest(data)
    │
    ├── prefabId = data.prefabId  // "Box_1"
    │
    ├── prefab = _database.GetPrefab("Box_1")
    │   ├── 精确匹配 → 找到 Box_1 预制体
    │   └── 返回预制体
    │
    └── 实例化并恢复数据
```

### 4.3 旧存档兼容流程

```
DynamicObjectFactory.TryReconstructChest(data)
    │
    ├── prefabId = data.prefabId  // "Storage_1400_小木箱子_0"（旧存档）
    │
    ├── prefab = _database.GetPrefab("Storage_1400_小木箱子_0")
    │   ├── 精确匹配 → 失败
    │   ├── 清洗名称 → 失败
    │   ├── 类型推断 → 检测到 "Storage_" 前缀
    │   │   └── 回退到 "Box_1"
    │   └── 返回 Box_1 预制体
    │
    └── 实例化并恢复数据（输出警告日志）
```

---

## 五、正确性属性

### P1：预制体唯一性

**属性**：同一个 PrefabDatabase 中，不存在两个同名的预制体条目

**验证方式**：扫描时自动去重，保留第一个

### P2：查找确定性

**属性**：对于同一个 prefabName，`GetPrefab()` 总是返回相同的结果

**验证方式**：使用 Dictionary 缓存，O(1) 查找

### P3：向后兼容性

**属性**：旧存档中的 prefabId 能通过回退机制找到对应预制体

**验证方式**：
- `Storage_*` → 回退到 `Box_1`
- `Stone_*` → 回退到 `Rock_M1`
- `Tree_*` → 回退到 `Tree_M1_00`

---

## 六、文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `PrefabDatabase.cs` | 新增 | 预制体数据库 |
| `PrefabDatabaseEditor.cs` | 新增 | 编辑器工具 |
| `PrefabDatabaseAutoScanner.cs` | 新增 | 自动扫描器 |
| `DynamicObjectFactory.cs` | 修改 | 使用 PrefabDatabase |
| `ChestController.cs` | 修改 | Save() 使用预制体名称 |
| `PersistentManagers.cs` | 修改 | 初始化 PrefabDatabase |
| `PrefabRegistry.cs` | 标记废弃 | 保留兼容，标记 Obsolete |

---

**文档结束**
