# 存档系统严重问题修复 - 设计文档

## 问题根因分析

### 🔥 P0-3: Registry 生命周期问题（致命！这是真正的元凶）

**代码分析**：
```csharp
// PersistentObjectRegistry.cs
public class PersistentObjectRegistry : MonoBehaviour, IPersistentObjectRegistry
{
    private void Awake()
    {
        // 单例检查
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        
        _instance = this;
        DontDestroyOnLoad(gameObject);  // 🔥 问题根源！
    }
}
```

**问题根因**：
1. `PersistentObjectRegistry` 是单例且 `DontDestroyOnLoad`
2. 当重新加载场景（读取存档）时，旧场景销毁了，但 Registry 里还存着旧对象的引用
3. 新场景加载时，对象执行 `Start()` 注册，发现 ID 已存在（残留的 key），触发自愈生成新 ID
4. SaveManager 读档时拿着存档里的旧 ID 去找对象，找不到！

**灾难流程图**：
```
场景 A 运行中
├── Tree_01 注册到 Registry (ID: Tree_01)
├── 玩家砍树，保存存档 (存档记录: Tree_01 血量=50)
└── 玩家读档

读档触发场景重载
├── 场景 A 销毁（但 Registry 是 DontDestroyOnLoad，不清空！）
├── 场景 A 重新加载
├── Tree_01 执行 Start() → TryRegister()
│   └── Registry 说："Tree_01 已存在！"（残留的 key）
│   └── Tree_01 触发自愈，生成新 ID: Tree_New_Random
├── SaveManager.RestoreAllFromSaveData()
│   └── 查找 Tree_01 → 找不到！
│   └── 树木状态恢复失败！
```

**现状分析**：
- ✅ `PersistentObjectRegistry.Clear()` 方法**已存在**（第 175-183 行）
- ✅ Clear() 方法已正确清空 `_registry` 和 `_byType` 两个字典
- ❌ `SaveManager.LoadGame()` **没有调用** Clear()！这是问题所在！

**修复方案**：
```csharp
// SaveManager.LoadGame() 开头添加（第一行！）
public bool LoadGame(string slotName)
{
    // 🔥 关键修复：清空 Registry，避免 ID 残留
    // Clear() 方法已存在，只需在这里调用
    if (PersistentObjectRegistry.Instance != null)
    {
        PersistentObjectRegistry.Instance.Clear();
        if (showDebugInfo)
            Debug.Log("[SaveManager] 已清空 PersistentObjectRegistry");
    }
    
    // ... 后续加载逻辑
}
```

---

### P0-1: 玩家位置不同步

**代码分析**：
```csharp
// SaveManager.RestorePlayerData()
private void RestorePlayerData(PlayerSaveData data)
{
    if (data == null) return;
    
    var player = GameObject.FindGameObjectWithTag("Player");
    if (player != null)
    {
        player.transform.position = new Vector3(data.positionX, data.positionY, 0);
    }
}
```

**🔥 辅助文档补充分析**：
- 如果 Player 身上有 `Rigidbody2D`，直接修改 `transform.position` 可能被物理引擎覆盖
- Tool 是玩家的子物体，如果它"留在原地"，说明父子关系可能在加载瞬间断开了

**修复方案**：
```csharp
private void RestorePlayerData(PlayerSaveData data)
{
    if (data == null) return;
    
    var player = GameObject.FindGameObjectWithTag("Player");
    if (player != null)
    {
        // 🔥 修复：暂时禁用物理组件
        var rb = player.GetComponent<Rigidbody2D>();
        bool wasKinematic = false;
        if (rb != null)
        {
            wasKinematic = rb.isKinematic;
            rb.isKinematic = true;
        }
        
        // 设置位置
        player.transform.position = new Vector3(data.positionX, data.positionY, 0);
        
        // 🔥 修复：强制重置 Tool 子物体的 localPosition
        var tool = player.transform.Find("Tool");
        if (tool != null)
        {
            tool.localPosition = Vector3.zero;
        }
        
        // 恢复物理组件状态
        if (rb != null)
        {
            rb.isKinematic = wasKinematic;
        }
    }
}
```

---

### P0-2: 箱子 IsEmpty 状态问题

**代码分析**：

```csharp
// ChestController.IsEmpty 属性
public bool IsEmpty => (_inventoryV2 == null || _inventoryV2.IsEmpty) && (_inventory == null || _inventory.IsEmpty);
```

**问题根因**：
1. `ChestController` 有两个库存系统：`_inventory` (ChestInventory) 和 `_inventoryV2` (ChestInventoryV2)
2. UI 操作修改的是 `_inventory`
3. 存档保存/加载使用的是 `_inventoryV2`
4. `Load()` 方法调用了 `SyncV2ToInventory()`，但可能没有正确执行

**🔥 辅助文档补充分析**：
- 死锁场景：读档后 `_inventoryV2` 恢复了数据，但 `_inventory` 可能还有脏数据
- 需要在 Load 完毕后强制检查状态

**现状分析**：
- ✅ `ChestController.Load()` 已有完善实现（第 268-310 行）
- ✅ 已确保 `_inventory` 和 `_inventoryV2` 初始化
- ✅ 已调用 `SyncV2ToInventory()` 同步数据
- ✅ 已调用 `UpdateSprite()` 更新视觉状态
- ⚠️ 缺少：处理空箱子情况（存档中没有槽位数据时清空两个库存）
- ⚠️ 缺少：强制状态检查确保 `IsEmpty` 返回正确值

**修复方案**：
```csharp
public void Load(WorldObjectSaveData data)
{
    if (data == null) return;
    
    // 恢复位置
    transform.position = data.GetPosition();
    transform.rotation = Quaternion.Euler(0, 0, data.rotationZ);
    
    // 解析箱子特有数据
    if (!string.IsNullOrEmpty(data.genericData))
    {
        var chestData = JsonUtility.FromJson<ChestSaveData>(data.genericData);
        if (chestData != null)
        {
            isLocked = chestData.isLocked;
            
            // 🔥 确保 _inventory 和 _inventoryV2 已初始化（已实现）
            int capacity = chestData.capacity > 0 ? chestData.capacity : (storageData?.storageCapacity ?? 20);
            if (_inventory == null)
            {
                _inventory = new ChestInventory(capacity);
            }
            if (_inventoryV2 == null)
            {
                _inventoryV2 = new ChestInventoryV2(capacity);
            }
            
            // 恢复库存数据
            if (chestData.slots != null && chestData.slots.Count > 0)
            {
                _inventoryV2.LoadFromSaveData(chestData.slots);
                SyncV2ToInventory();
            }
            else
            {
                // 🔥 修复：如果存档中没有槽位数据，说明箱子是空的
                // 必须清空两个库存系统，避免残留数据
                _inventoryV2.Clear();
                _inventory.Clear();
            }
        }
    }
    
    // 🔥 修复：强制状态检查
    // 确保视觉状态与数据一致
    UpdateSprite();
    
    // 🔥 锐评007 要求：验证 IsEmpty 属性返回正确值
    // 如果 _inventoryV2 和 _inventory 都为空，IsEmpty 应该返回 true
    if (showDebugInfo)
    {
        Debug.Log($"[ChestController] Load 完成: GUID={PersistentId}, isLocked={isLocked}");
        Debug.Log($"[ChestController] 状态检查: IsEmpty={IsEmpty}, _inventoryV2.IsEmpty={_inventoryV2?.IsEmpty}, _inventory.IsEmpty={_inventory?.IsEmpty}");
    }
}
```

---

### P1-1: TreeController ID 冲突警告

**问题**：
- 每次启动游戏，如果有 Ctrl+D 复制的树木，都会触发警告
- 🔥 **更深层原因**：Registry 跨场景残留导致的假冲突（见 P0-3）

**修复方案**：
1. 首先修复 P0-3（Registry 清空），这会解决大部分假冲突
2. 将警告级别降低为 `Debug.Log`（仅在 `showDebugInfo` 时输出）

```csharp
private void RegisterToPersistentRegistry()
{
    if (PersistentObjectRegistry.Instance == null) return;
    
    if (!PersistentObjectRegistry.Instance.TryRegister(this))
    {
        // ID 冲突（可能是 Ctrl+D 复制的克隆体）
        if (showDebugInfo)
            Debug.Log($"[TreeController] {gameObject.name} ID 冲突检测 (ID: {persistentId})，正在重新生成...");
        persistentId = System.Guid.NewGuid().ToString();
        PersistentObjectRegistry.Instance.Register(this);
    }
}
```

---

### P1-2: StoneController 存档支持

**修改文件**: `Assets/YYY_Scripts/Controller/StoneController.cs`

**修改内容**：
1. 添加 `IPersistentObject` 接口实现
2. 添加 `persistentId` 字段
3. 实现 `Save()` 和 `Load()` 方法
4. 在 `Start()` 中注册到 `PersistentObjectRegistry`

---

### 🔥 P2-1: 反向修剪逻辑（Pruning）

**问题**：被砍掉的树又回来了！

**根因**：
- 当前 Save/Load 是"存在即保存，找到即恢复"
- 被砍掉的树在存档里没有数据
- 读档时场景加载默认树，SaveManager 找不到数据就跳过
- 结果：已删除的物体"复活"了

**🔥 锐评007 关键指令**：
1. 使用 `SetActive(false)` 而不是 `Destroy()`（考虑对象池化策略）
2. 必须先获取 `_registry.Keys` 的**副本**，避免遍历时修改集合
3. 执行顺序：先修剪，再恢复

**修复方案**：

```csharp
// PersistentObjectRegistry.RestoreAllFromSaveData() 修改
public void RestoreAllFromSaveData(List<WorldObjectSaveData> dataList)
{
    if (dataList == null) return;
    
    // 🔥 Step 1: 构建存档快照 - 收集存档中的所有 GUID
    var savedGuids = new HashSet<string>(dataList.Select(d => d.guid));
    
    // 🔥 Step 2: 快照当前场景 - 获取 _registry.Keys 的副本（避免遍历时修改集合）
    var currentRegistryKeys = new List<string>(_registry.Keys);
    
    // 🔥 Step 3: 修剪 (Pruning) - 场景中有但存档中没有 = 已删除
    int pruned = 0;
    foreach (var guid in currentRegistryKeys)
    {
        if (!savedGuids.Contains(guid))
        {
            // 存档中没有这个对象 → 说明玩家把它删了（砍树/挖箱子）
            if (_registry.TryGetValue(guid, out var obj) && obj != null)
            {
                if (obj is MonoBehaviour mb && mb != null)
                {
                    if (showDebugInfo)
                        Debug.Log($"[PersistentObjectRegistry] 反向修剪: 禁用 {obj.ObjectType}, GUID: {obj.PersistentId}");
                    
                    // 🔥 使用 SetActive(false) 而不是 Destroy()
                    // 原因：对于树木等对象，可能需要调用特定的 Hide() 方法
                    // 使用 Disable 比 Destroy 更安全，避免影响对象池
                    mb.gameObject.SetActive(false);
                    pruned++;
                }
            }
        }
    }
    
    // 🔥 Step 4: 恢复 (Restoring) - 遍历存档数据进行 Load()
    int restored = 0;
    int notFound = 0;
    
    foreach (var data in dataList)
    {
        var obj = FindByGuid(data.guid);
        if (obj != null)
        {
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
            notFound++;
            if (showDebugInfo)
                Debug.LogWarning($"[PersistentObjectRegistry] 找不到对象: {data.objectType}, GUID: {data.guid}");
        }
    }
    
    if (showDebugInfo)
        Debug.Log($"[PersistentObjectRegistry] 恢复完成: 成功 {restored}, 未找到 {notFound}, 修剪 {pruned}");
}
```

---

## 修复方案详细设计

### 修复 1: 🔥 Registry 生命周期（最高优先级！）

**修改文件**: `Assets/YYY_Scripts/Data/Core/SaveManager.cs`

**现状**：
- `PersistentObjectRegistry.Clear()` 方法**已存在**，无需添加
- `SaveManager.LoadGame()` **没有调用** Clear()，这是需要修复的

**修改内容**：
```csharp
public bool LoadGame(string slotName)
{
    // 🔥 关键修复：清空 Registry，避免 ID 残留
    // 必须在任何恢复逻辑之前执行！
    if (PersistentObjectRegistry.Instance != null)
    {
        PersistentObjectRegistry.Instance.Clear();
        if (showDebugInfo)
            Debug.Log("[SaveManager] 已清空 PersistentObjectRegistry");
    }
    
    // ... 后续加载逻辑（保持不变）
}
```

### 修复 2: 玩家位置 + Tool 重置

**修改文件**: `Assets/YYY_Scripts/Data/Core/SaveManager.cs`

### 修复 3: 箱子 IsEmpty 状态

**修改文件**: `Assets/YYY_Scripts/World/Placeable/ChestController.cs`

### 修复 4: TreeController ID 冲突警告

**修改文件**: `Assets/YYY_Scripts/Controller/TreeController.cs`

### 修复 5: StoneController 存档支持

**修改文件**: `Assets/YYY_Scripts/Controller/StoneController.cs`

### 修复 6: 反向修剪逻辑

**修改文件**: `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs`

---

## 🔥 实施顺序（重新排序！）

1. **P0-3: Registry 生命周期** - **这是真正的元凶！必须首先修复！**
2. **P0-1: 玩家位置 + Tool 重置** - 物理引擎覆盖问题
3. **P0-2: 箱子 IsEmpty 状态** - 双库存同步问题
4. **P1-1: TreeController ID 冲突警告** - 修复 P0-3 后会大幅减少
5. **P1-2: StoneController 存档支持** - 需要较多代码
6. **P2-1: 反向修剪逻辑** - 解决"已删除物体复活"问题

---

## 测试验证

### 🔥 Registry 测试（最重要！）
1. 创建新存档，砍伐部分树木
2. 保存游戏
3. 加载存档
4. 验证：控制台没有 ID 冲突警告
5. 验证：树木状态正确恢复

### 玩家位置测试
1. 移动玩家到特定位置
2. 保存游戏
3. 移动玩家到其他位置
4. 加载存档
5. 验证：玩家回到保存时的位置
6. 验证：Tool 子物体 localPosition 为 (0,0,0)

### 箱子测试
1. 创建新存档，放置箱子并放入物品
2. 保存游戏
3. 清空箱子内容
4. 加载存档
5. 验证：箱子内容恢复，`IsEmpty` 返回正确值
6. 验证：空箱子可以被镐子挖取

### 反向修剪测试
1. 创建新存档，砍掉一棵树
2. 保存游戏
3. 加载存档
4. 验证：被砍掉的树没有"复活"

### 石头测试
1. 创建新存档，挖掘部分石头
2. 保存游戏
3. 加载存档
4. 验证：石头状态恢复到保存时的阶段和血量
