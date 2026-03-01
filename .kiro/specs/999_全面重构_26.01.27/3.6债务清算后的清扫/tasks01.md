# 存档系统紧急修复 - 任务清单 01

## 概述

**锐评来源**: 2修复阶段锐评001  
**修复目标**: 撤回自杀式 Clear()，实现正确的清理逻辑

---

## 任务列表

### 阶段 1: 紧急复苏（CPR）

- [x] 1.1 实现 PruneStaleRecords() 方法
  - [x] 1.1.1 在 `PersistentObjectRegistry.cs` 中添加 `PruneStaleRecords()` 方法
    - 收集所有 `Value == null` 的键
    - 移除这些空引用
    - 同时清理 `_byType` 字典中的空引用
    - 添加调试日志

- [x] 1.2 修正 SaveManager.LoadGame()
  - [x] 1.2.1 删除 `PersistentObjectRegistry.Instance.Clear()` 调用
  - [x] 1.2.2 替换为 `PersistentObjectRegistry.Instance.PruneStaleRecords()` 调用
  - [x] 1.2.3 更新调试日志信息

---

### 阶段 2: 硬编码修正

- [x] 2.1 检查 ChestController 硬编码
  - [x] 2.1.1 搜索 `new ChestInventoryV2(20)` 或类似硬编码
  - [x] 2.1.2 确保容量优先从存档数据获取
  - [x] 2.1.3 如果存档数据没有，使用 `storageData?.storageCapacity`
  - [x] 2.1.4 最后才使用默认值 20
  - [x] 2.1.5 添加调试日志验证 capacity（锐评010 指令）

---

### 阶段 3: 验证测试

- [x] 3.1 编译验证
  - [x] 3.1.1 确保代码编译通过
  - [x] 3.1.2 检查 Unity 控制台无错误

- [ ] 3.2 功能测试（需用户手动验证）
  - [ ] 3.2.1 测试基础存档恢复（树木状态）
  - [ ] 3.2.2 测试箱子状态恢复
  - [ ] 3.2.3 测试玩家位置恢复
  - [ ] 3.2.4 测试反向修剪（被砍的树不复活）

---

## 修改文件清单

| 文件 | 修改内容 | 状态 |
|------|---------|------|
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 新增 PruneStaleRecords() 方法 | ✅ 已完成 |
| `Assets/YYY_Scripts/Data/Core/SaveManager.cs` | 删除 Clear()，改为 PruneStaleRecords() | ✅ 已完成 |
| `Assets/YYY_Scripts/World/Placeable/ChestController.cs` | 添加 capacity 调试日志 | ✅ 已完成 |

---

## 执行顺序

```
1. [Core] 实现 PruneStaleRecords() (1.1)
   ↓
2. [Core] 修正 SaveManager.LoadGame() (1.2)
   ↓
3. [Chest] 检查硬编码 (2.1)
   ↓
4. [Test] 编译验证 (3.1)
   ↓
5. [Test] 功能测试 (3.2)
```

---

## 关键代码片段

### PruneStaleRecords() 实现

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
    }
    
    // 同时清理 _byType 中的空引用
    foreach (var typeSet in _byType.Values)
    {
        typeSet.RemoveWhere(obj => obj == null || obj.Equals(null));
    }
    
    if (showDebugInfo && keysToRemove.Count > 0)
        Debug.Log($"[PersistentObjectRegistry] PruneStaleRecords: 清理了 {keysToRemove.Count} 个空引用");
}
```

### SaveManager.LoadGame() 修改

```csharp
public bool LoadGame(string slotName)
{
    if (string.IsNullOrEmpty(slotName))
    {
        Debug.LogError("[SaveManager] 存档名称不能为空");
        return false;
    }
    
    // 🔥 修复：清理空引用，而不是清空所有
    // 原地读档模式下，Registry 是连接存档数据和场景实例的唯一桥梁，绝对不能断！
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

## 风险提示

1. **不要再调用 Clear()**：在原地读档模式下，Clear() 会切断存档数据和场景实例的桥梁
2. **PruneStaleRecords 只清理空引用**：只移除 `Value == null` 的键值对，保留所有活着的对象
3. **反向修剪依赖 Registry 有数据**：修剪逻辑需要 `currentRegistryKeys` 不为空才能正常工作

