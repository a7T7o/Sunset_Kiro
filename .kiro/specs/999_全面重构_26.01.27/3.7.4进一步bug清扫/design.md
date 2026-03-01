# 存档系统 Bug 修复 - 设计文档

**创建日期**: 2026-02-03  
**版本**: 1.2（修复 Load() 激活逻辑遗漏）  
**状态**: 待实施

---

## 一、架构概述

### 1.1 当前架构问题

```
┌─────────────────────────────────────────────────────────────┐
│                    当前问题链路                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  石头问题：                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ 挖掉石头 │ -> │ Destroy │ -> │ 引用null │                 │
│  └─────────┘    └─────────┘    └─────────┘                 │
│                                      ↓                      │
│                              ┌─────────────┐                │
│                              │ 反向修剪失败 │                │
│                              └─────────────┘                │
│                                                             │
│  箱子问题：                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ UI清空  │ -> │_inventory│ -> │ 已清空  │                 │
│  └─────────┘    └─────────┘    └─────────┘                 │
│                                      ↓                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ IsEmpty │ <- │_inventoryV2│ <- │ 未同步  │               │
│  │ = false │    └─────────┘    └─────────┘                │
│  └─────────┘                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 修复后架构

```
┌─────────────────────────────────────────────────────────────┐
│                    修复后架构                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  石头修复：                                                  │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │ 挖掉石头 │ -> │SetActive(F) │ -> │ 引用保留    │         │
│  └─────────┘    └─────────────┘    └─────────────┘         │
│                                           ↓                 │
│                                   ┌─────────────┐           │
│                                   │ 反向修剪成功 │           │
│                                   └─────────────┘           │
│                                                             │
│  箱子修复：                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ IsEmpty │ <- │_inventory│ <- │ 只检查它 │                │
│  │ = true  │    └─────────┘    └─────────┘                 │
│  └─────────┘                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、组件设计

### 2.1 StoneController 修改

#### 2.1.1 DestroyStone() 方法

**修改前**：
```csharp
private void DestroyStone()
{
    isDepleted = true;
    
    if (ResourceNodeRegistry.Instance != null)
    {
        ResourceNodeRegistry.Instance.Unregister(gameObject.GetInstanceID());
    }
    
    // 🔴 问题：直接销毁
    if (transform.parent != null)
    {
        Destroy(transform.parent.gameObject);
    }
    else
    {
        Destroy(gameObject);
    }
}
```

**修改后**：
```csharp
private void DestroyStone()
{
    isDepleted = true;
    
    // 🔥 只从 ResourceNodeRegistry 注销（假死的石头不应该被攻击）
    if (ResourceNodeRegistry.Instance != null)
    {
        ResourceNodeRegistry.Instance.Unregister(gameObject.GetInstanceID());
    }
    
    // 🔥 不从 PersistentObjectRegistry 注销！
    // 这样反向修剪才能正确工作
    
    // 🔥 假死：禁用而非销毁
    if (transform.parent != null)
    {
        transform.parent.gameObject.SetActive(false);
    }
    else
    {
        gameObject.SetActive(false);
    }
    
    if (showDebugInfo)
        Debug.Log($"<color=orange>[StoneController] {gameObject.name} 被完全挖掘（假死）</color>");
}
```

#### 2.1.2 Load() 方法 - 🔴 致命漏洞修复

**问题分析**：
- 石头使用假死机制（`SetActive(false)`），但 `Load()` 只恢复数据
- **没有调用 `SetActive(true)` 激活对象**
- **没有重置 `isDepleted = false`**
- **没有重新注册到 `ResourceNodeRegistry`**
- 导致石头恢复后仍然不显示、无法被攻击

**对比树木**：
| 特性 | 树木 | 石头（当前） | 石头（修复后） |
|------|------|------------|--------------|
| 销毁方式 | `Destroy()` | `SetActive(false)` | `SetActive(false)` |
| 恢复机制 | `DynamicObjectFactory` 动态重建 | 只恢复数据 | 恢复数据 + 激活对象 |
| Load() 后状态 | 新实例，自然激活 | 仍然 Inactive | 激活 + 可攻击 |

**修改后**：
```csharp
public void Load(WorldObjectSaveData data)
{
    if (data == null || string.IsNullOrEmpty(data.genericData)) return;
    
    // 从 genericData 反序列化石头数据
    var stoneData = JsonUtility.FromJson<StoneSaveData>(data.genericData);
    if (stoneData == null) return;
    
    // 恢复石头特有数据
    currentStage = (StoneStage)stoneData.stage;
    oreType = (OreType)stoneData.oreType;
    oreIndex = stoneData.oreIndex;
    currentHealth = stoneData.currentHealth;
    
    // 更新运行时调试状态
    lastStage = currentStage;
    lastOreType = oreType;
    lastOreIndex = oreIndex;
    
    // 🔥 新增：恢复激活状态（假死机制的关键补充）
    isDepleted = false;
    
    // 激活对象（假死的石头需要被激活）
    if (transform.parent != null)
        transform.parent.gameObject.SetActive(true);
    else
        gameObject.SetActive(true);
    
    // 重新注册到 ResourceNodeRegistry（恢复后应该可以被攻击）
    if (ResourceNodeRegistry.Instance != null)
    {
        ResourceNodeRegistry.Instance.Register(this);
    }
    
    // 立即刷新视觉
    UpdateSprite();
    
    if (showDebugInfo)
        Debug.Log($"[StoneController] Load: GUID={PersistentId}, stage={currentStage}, health={currentHealth}, 已激活");
}
```

#### 2.1.3 ShouldSave 属性

**保持不变**：
```csharp
public bool ShouldSave => gameObject.activeInHierarchy && !isDepleted;
```

**设计理由**：
- 假死的石头（`isDepleted = true`）不应该被保存
- 存档中没有这个石头 → 读档时反向修剪会禁用它
- 这是正确的语义：存档保存的是"当前世界状态"

---

### 2.2 ChestController 修改

#### 2.2.1 IsEmpty 属性

**修改前**：
```csharp
public bool IsEmpty => (_inventoryV2 == null || _inventoryV2.IsEmpty) 
                    && (_inventory == null || _inventory.IsEmpty);
```

**修改后**：
```csharp
/// <summary>
/// 是否为空
/// 🔥 P4 修复：只检查 _inventory，因为它是 UI 操作的目标
/// _inventoryV2 只用于存档，不参与运行时逻辑判断
/// </summary>
public bool IsEmpty => _inventory == null || _inventory.IsEmpty;
```

**设计理由**：
- `_inventory` 是 UI 操作的目标，是运行时的"真理来源"
- `_inventoryV2` 只用于存档序列化，不应该参与运行时逻辑判断
- 这符合 Gemini 锐评的"单一真理来源"原则

---

### 2.3 反向修剪机制（已修复）

`PersistentObjectRegistry.RestoreAllFromSaveData()` 中的反向修剪逻辑：

```csharp
// Step 3: 修剪 (Pruning) - 场景中有但存档中没有 = 已删除
foreach (var guid in currentRegistryKeys)
{
    if (!savedGuids.Contains(guid))
    {
        // 存档中没有这个对象 → 说明玩家把它删了
        if (_registry.TryGetValue(guid, out var obj) && obj != null)
        {
            if (obj is MonoBehaviour mb && mb != null)
            {
                // 🔥 P0 修复：区分动态对象和静态对象
                if (obj is WorldItemPickup)
                {
                    Destroy(mb.gameObject);  // 动态掉落物：销毁
                }
                else
                {
                    mb.gameObject.SetActive(false);  // 静态石头：隐藏
                }
                pruned++;
            }
        }
    }
}
```

**关键点**：
- 反向修剪依赖对象仍在 `_registry` 中
- 假死机制确保对象不被销毁，引用保留
- 读档时，存档中没有的对象会被禁用或销毁
- **动态对象（掉落物）使用 `Destroy()` 销毁**
- **静态对象（石头、树木）使用 `SetActive(false)` 禁用**

---

### 2.4 WorldItemPickup 注册机制（新增）

掉落物需要注册到 `PersistentObjectRegistry` 才能被反向修剪正确处理：

```csharp
private void Start()
{
    EnsureInitialized();
    
    // 🔥 P0 修复：注册到持久化对象注册表
    RegisterToPersistentRegistry();
}

private void OnDestroy()
{
    // 🔥 P0 修复：从持久化对象注册表注销
    UnregisterFromPersistentRegistry();
}
```

**设计理由**：
- 掉落物是动态生成的，需要在生成时注册
- 反向修剪依赖 `_registry` 中的对象引用
- 没有注册的掉落物无法被反向修剪找到

---

## 三、数据流设计

### 3.1 石头存档恢复流程

```
┌─────────────────────────────────────────────────────────────┐
│                    石头存档恢复流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  保存存档（石头存在）：                                       │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │ 石头活跃 │ -> │ ShouldSave  │ -> │ 保存到存档  │         │
│  │ isDepleted=F│ │ = true      │    └─────────────┘         │
│  └─────────┘    └─────────────┘                             │
│                                                             │
│  挖掉石头：                                                  │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │ 挖掉石头 │ -> │SetActive(F) │ -> │ 假死状态    │         │
│  │          │    │ isDepleted=T│    │ 引用保留    │         │
│  └─────────┘    └─────────────┘    └─────────────┘         │
│                                                             │
│  加载存档：                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ 存档中有石头 │ -> │ FindByGuid  │ -> │ Load() 恢复 │     │
│  │              │    │ 找到假死石头│    │ SetActive(T)│     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 箱子空状态判断流程

```
┌─────────────────────────────────────────────────────────────┐
│                    箱子空状态判断流程                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  加载存档（箱子有物品）：                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Load() 调用 │ -> │SyncV2ToInv  │ -> │_inventory有物品│   │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  UI 清空箱子：                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ UI 操作     │ -> │_inventory   │ -> │ 已清空      │     │
│  │              │    │ 被修改      │    │              │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  检查 IsEmpty：                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ OnHit()     │ -> │ IsEmpty     │ -> │ = true      │     │
│  │ 检查 IsEmpty│    │只检查_inventory│ │ 触发挖取    │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、正确性属性

### 4.1 石头假死机制

| 属性 | 描述 |
|------|------|
| P-1.1 | 假死的石头仍在 `PersistentObjectRegistry._registry` 中 |
| P-1.2 | 假死的石头 `ShouldSave` 返回 `false` |
| P-1.3 | 假死的石头不在 `ResourceNodeRegistry` 中（不可被攻击） |
| P-1.4 | 读档时，存档中有的石头会被恢复（`SetActive(true)`） |
| P-1.5 | 读档时，存档中没有的石头会被禁用（反向修剪） |

### 4.2 箱子空状态判断

| 属性 | 描述 |
|------|------|
| P-2.1 | `IsEmpty` 只检查 `_inventory` |
| P-2.2 | UI 操作只修改 `_inventory` |
| P-2.3 | `Save()` 时 `_inventory` 同步到 `_inventoryV2` |
| P-2.4 | `Load()` 时 `_inventoryV2` 同步到 `_inventory` |

---

## 五、测试策略

### 5.1 单元测试

| 测试用例 | 描述 |
|---------|------|
| T-1.1 | 石头 `DestroyStone()` 后 `gameObject.activeInHierarchy` 为 `false` |
| T-1.2 | 石头 `DestroyStone()` 后仍在 `PersistentObjectRegistry` 中 |
| T-2.1 | 箱子 `_inventory` 清空后 `IsEmpty` 返回 `true` |
| T-2.2 | 箱子 `_inventoryV2` 有物品但 `_inventory` 为空时 `IsEmpty` 返回 `true` |

### 5.2 集成测试

| 测试用例 | 描述 |
|---------|------|
| T-3.1 | 保存存档（石头存在）→ 挖掉石头 → 加载存档 → 石头恢复 |
| T-3.2 | 加载存档（箱子有物品）→ 清空箱子 → 用镐子敲箱子 → 触发挖取 |

---

## 六、风险评估

### 6.1 已识别风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 假死对象内存占用 | 低 | 假死对象数量有限，内存影响可忽略 |
| 反向修剪性能 | 低 | 当前实现已优化，100ms 内完成 |
| 兼容性问题 | 中 | 修改不影响存档格式，兼容现有存档 |

### 6.2 锐评002 补充：幽灵对象累积问题

Code Reaper 指出假死对象可能导致内存泄漏，经审视：

| 对象类型 | 是否会无限增长 | 结论 |
|---------|---------------|------|
| 静态场景对象（预制体石头） | ❌ 不会 | 数量固定，不是内存泄漏 |
| 动态生成对象（掉落物） | ✅ 可能 | 需要在 P2 优化中处理 |

**P2 优化方案**：反向修剪时区分对象类型
```csharp
if (obj is WorldItemPickup) {
    Destroy(obj.gameObject);  // 动态掉落物：销毁
} else {
    obj.gameObject.SetActive(false);  // 静态石头：隐藏
}
```

### 6.2 未解决问题

| 问题 | 优先级 | 计划 |
|------|--------|------|
| 掉落物关联机制 | P2 | 后续迭代实现 |
| 箱子实时同步 | P1 | 本周完成 |
| 石头动态重建 | P1 | 本周完成 |

---

**文档结束**
