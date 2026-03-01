# 存档系统紧急修复 - 设计文档 01

## 概述

**锐评来源**: 2修复阶段锐评001  
**修复目标**: 撤回自杀式 Clear()，实现正确的清理逻辑

---

## 问题根因分析

### 原地读档 vs 换场景读档

**关键区别**：
- **换场景读档**：旧场景销毁 → Registry 残留 → 新场景对象注册冲突 → 需要 Clear()
- **原地读档**：场景不销毁 → 对象已注册 → Registry 是连接存档数据和场景实例的唯一桥梁 → **绝对不能 Clear()**

**当前项目模式**：原地读档（不换场景）

**我的错误**：把"换场景读档"的解决方案（Clear）应用到了"原地读档"场景，导致桥梁断裂。

---

## 修复方案

### 方案 1: 删除 Clear() 调用

**修改文件**: `Assets/YYY_Scripts/Data/Core/SaveManager.cs`

**修改内容**：
```csharp
public bool LoadGame(string slotName)
{
    if (string.IsNullOrEmpty(slotName))
    {
        Debug.LogError("[SaveManager] 存档名称不能为空");
        return false;
    }
    
    // 🔥 删除这段代码！
    // if (PersistentObjectRegistry.Instance != null)
    // {
    //     PersistentObjectRegistry.Instance.Clear();
    //     if (showDebugInfo)
    //         Debug.Log("[SaveManager] 已清空 PersistentObjectRegistry，准备加载存档");
    // }
    
    // 🔥 替换为：清理空引用（已销毁的对象）
    if (PersistentObjectRegistry.Instance != null)
    {
        PersistentObjectRegistry.Instance.PruneStaleRecords();
        if (showDebugInfo)
            Debug.Log("[SaveManager] 已清理 PersistentObjectRegistry 中的空引用");
    }
    
    // ... 后续加载逻辑保持不变
}
```

---

### 方案 2: 实现 PruneStaleRecords() 方法

**修改文件**: `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs`

**新增方法**：
```csharp
/// <summary>
/// 清理空引用（已销毁的对象）
/// 🔥 锐评009 指令：只移除 Value 为 null 的键值对，不清空所有
/// </summary>
public void PruneStaleRecords()
{
    // 收集所有 Value 为 null 的键（对象已被 Destroy）
    var keysToRemove = _registry
        .Where(kvp => kvp.Value == null || kvp.Value.Equals(null))
        .Select(kvp => kvp.Key)
        .ToList();
    
    // 移除空引用
    foreach (var key in keysToRemove)
    {
        _registry.Remove(key);
        
        // 同时从 _byType 中移除
        foreach (var typeSet in _byType.Values)
        {
            typeSet.RemoveWhere(obj => obj == null || obj.Equals(null));
        }
    }
    
    if (showDebugInfo && keysToRemove.Count > 0)
        Debug.Log($"[PersistentObjectRegistry] PruneStaleRecords: 清理了 {keysToRemove.Count} 个空引用");
}
```

**设计说明**：
- 只移除 `Value == null` 的键值对（对象已被 Unity Destroy）
- 保留所有活着的对象引用
- 同时清理 `_byType` 字典中的空引用

---

### 方案 3: 检查 ChestController 硬编码

**修改文件**: `Assets/YYY_Scripts/World/Placeable/ChestController.cs`

**检查点**：
1. 搜索 `new ChestInventoryV2(20)` 或类似硬编码
2. 确保容量优先从以下来源获取：
   - 存档数据中的 `capacity`
   - `storageData?.storageCapacity`
   - 最后才是默认值 20

**修改逻辑**：
```csharp
// 在 Load() 方法中
int capacity = chestData.capacity > 0 
    ? chestData.capacity 
    : (storageData?.storageCapacity ?? 20);

if (_inventoryV2 == null)
{
    _inventoryV2 = new ChestInventoryV2(capacity);
}
```

---

### 方案 4: 验证反向修剪逻辑

**现有实现审查**：

`RestoreAllFromSaveData()` 方法的反向修剪逻辑**依赖于 Registry 有数据**：

```csharp
// Step 2: 快照当前场景 - 获取 _registry.Keys 的副本
var currentRegistryKeys = new List<string>(_registry.Keys);

// Step 3: 修剪 - 场景中有但存档中没有 = 已删除
foreach (var guid in currentRegistryKeys)
{
    if (!savedGuids.Contains(guid))
    {
        // 禁用该对象
    }
}
```

**问题**：如果 `Clear()` 在前面执行了，`currentRegistryKeys` 是空的，修剪逻辑完全失效！

**修复后**：删除 `Clear()` 后，`currentRegistryKeys` 会包含场景中所有活着的对象，修剪逻辑恢复正常。

---

## 数据流图

### 修复前（错误）

```
LoadGame() 开始
    ↓
Clear() ← 🔥 自杀！Registry 清空！
    ↓
读取 JSON
    ↓
RestoreAllFromSaveData()
    ↓
currentRegistryKeys = [] ← 空的！
    ↓
修剪逻辑跳过（没东西可修剪）
    ↓
遍历 saveDataList
    ↓
FindByGuid(guid) → null ← 找不到！Registry 是空的！
    ↓
所有 Load() 跳过
    ↓
💀 存档完全没有任何作用
```

### 修复后（正确）

```
LoadGame() 开始
    ↓
PruneStaleRecords() ← 只清理空引用
    ↓
读取 JSON
    ↓
RestoreAllFromSaveData()
    ↓
currentRegistryKeys = [Tree_01, Tree_02, Chest_01, ...] ← 场景中活着的对象
    ↓
修剪逻辑正常执行
    ↓
遍历 saveDataList
    ↓
FindByGuid(guid) → 找到对象！
    ↓
obj.Load(data) 执行
    ↓
✅ 存档正确恢复
```

---

## 测试验证

### 测试 1: 基础存档恢复

1. 创建新存档
2. 砍伐部分树木
3. 保存游戏
4. 加载存档
5. **预期**：树木状态正确恢复（被砍的树不在了）

### 测试 2: 箱子状态恢复

1. 放置箱子并放入物品
2. 保存游戏
3. 清空箱子
4. 加载存档
5. **预期**：箱子内容恢复

### 测试 3: 玩家位置恢复

1. 移动玩家到特定位置
2. 保存游戏
3. 移动玩家到其他位置
4. 加载存档
5. **预期**：玩家回到保存时的位置

---

## 风险评估

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| PruneStaleRecords 误删活对象 | 高 | 只检查 `== null`，不检查其他条件 |
| 反向修剪误删对象 | 中 | 使用 `SetActive(false)` 而非 `Destroy()` |
| 容量不一致 | 低 | 优先使用存档数据中的容量 |

