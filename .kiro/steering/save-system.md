---
inclusion: manual
priority: P1
keywords: [存档, 保存, 加载, GUID, 持久化, PersistentObject, Save, Load]
lastUpdated: 2026-02-12
---

# 存档系统规则

> 本文件基于 3.7.2→3.7.6 共 5 轮 bug 清扫的实战经验编写，所有规则均经过代码验证。

---

## 一、核心架构

| 组件 | 职责 | 文件 |
|------|------|------|
| PersistentObjectRegistry | Dictionary 存储所有持久化对象，反向修剪 | `PersistentObjectRegistry.cs` |
| DynamicObjectFactory | 动态对象重建（树木/石头/箱子/掉落物） | `DynamicObjectFactory.cs` |
| PrefabDatabase | 自动扫描预制体 + ID 别名映射 | ScriptableObject |
| SaveManager | 协调全局存档/读档流程 | `SaveManager.cs` |
| PersistentManagers | 管理器级持久化（TimeManager 等） | `PersistentManagers.cs` |

---

## 二、GUID 生命周期

| 对象类型 | GUID 来源 | 生成时机 | 稳定性 |
|---------|----------|---------|--------|
| 静态对象（场景预制体） | PersistentIdAutomator 自动分配 | 编辑器保存场景时 | ✅ 绝对稳定 |
| 动态对象（运行时生成） | `System.Guid.NewGuid()` | 放置/生成时 | ✅ 存档后稳定 |
| 重建对象（读档恢复） | 从存档数据中读取 | DynamicObjectFactory.TryReconstruct | ✅ 与原始一致 |

### 🔴 禁止事项
- 禁止使用 `GetInstanceID()` 作为持久化标识（每次运行都不同）
- 禁止在 `Awake()/Start()` 中生成新 GUID 覆盖已有值

---

## 三、执行顺序陷阱（最易出错）

动态重建的对象，`Load()` 在 `Instantiate` 后立即调用，但 `Start()` 要等到下一帧：

```
DynamicObjectFactory.TryReconstruct()
  → Instantiate(prefab)
  → obj.Load(data)          ← 此时 Start() 还没执行
  → SetActive(true)
  → [下一帧] Start() 执行   ← 如果 Start() 中有初始化，会覆盖 Load() 的数据
```

### 防御措施
- `Initialize()` 方法必须检查数据是否已被 `Load()` 填充（null 检查）
- 箱子系统已踩过此坑：`ChestController.Initialize()` 中 `_inventory` 的 null 检查

---

## 四、动态对象重建流程

```
SaveManager.LoadGame()
  → RestoreGameTimeData()           // 1. 先恢复时间
  → RestoreAllFromSaveData()        // 2. 再恢复世界对象
    → 遍历 WorldObjectSaveData 列表
    → 已注册对象 → 直接调用 Load()
    → 未注册对象 → DynamicObjectFactory.TryReconstruct()
      → 根据 objectType 分发：Tree/Stone/Chest/Drop
      → Instantiate → Load → SetActive → Register
```

### 🔴 加载顺序保证
TimeManager 必须先于所有世界对象恢复，否则季节渐变等依赖时间的系统会读到默认值。

### ⚠️ 农作物重建流程
农作物（Crop）的动态重建流程待 10.0.x 完成后补充。当前 CropController 已实现 `Save()/Load()` 接口，使用 `genericData` + JSON 序列化模式。

---

## 五、反向修剪机制

- `PruneStaleRecords()` 而非 `Clear()`
- 只移除"存档中没有但注册表中有"的对象
- 保留"存档中有且注册表中也有"的对象
- 石头使用假死机制：`SetActive(false)` 而非 `Destroy()`

---

## 六、特殊对象数据封装（genericData 模式）

特殊对象（如农作物）的私有数据通过 `WorldObjectSaveData.genericData` 字段封装：

```csharp
// 存档时：将特殊数据 JSON 序列化到 genericData
saveData.genericData = JsonUtility.ToJson(cropSaveData);

// 读档时：从 genericData 反序列化
var cropData = JsonUtility.FromJson<CropSaveData>(data.genericData);
```

### 标准模式
1. 定义 `[Serializable]` 的数据类（如 `CropSaveData`）放在 `SaveDataDTOs.cs`
2. `Save()` 中创建数据类实例，JSON 序列化到 `genericData`
3. `Load()` 中从 `genericData` 反序列化，恢复所有字段

### 恢复后同步模式
CropController.Load() 中恢复数据后，通过 `tileData.SetCropData()` 同步给 FarmTileManager。这是正常的读档恢复行为（存档只存在 Controller 侧，恢复后同步给 Tile 保证数据一致性），不是双重真理。

---

## 七、自动化工具

| 工具 | 文件 | 用途 |
|------|------|------|
| PersistentIdAutomator | `Editor/PersistentIdAutomator.cs` | 场景保存时自动修复空/重复 GUID |
| PersistentIdValidator | `Editor/PersistentIdValidator.cs` | 手动验证工具，检查 GUID 完整性 |
| PrefabDatabaseAutoScanner | Editor 脚本 | 预制体自动扫描，维护 PrefabDatabase |

---

## 八、Unity 命名陷阱

- `Instantiate()` 会自动添加 `(Clone)` 后缀
- 场景中复制对象会添加 `(1)`、`(2)` 等后缀
- `DynamicObjectFactory` 中使用正则清洗：`Regex.Replace(name, @"\s*\(.*\)$", "")`

---

## 九、渲染层级存档

- `WorldObjectSaveData` 包含 `sortingLayerName` 和 `sortingOrder`
- 存档时通过 `SetSortingLayer(SpriteRenderer)` 记录
- 读档时通过 `RestoreSortingLayer(SpriteRenderer)` 恢复

---

## 十、IPersistentObject 接口

项目使用 `Save()/Load()` 模式（不是 `SaveState()/RestoreState()`）：

```csharp
public interface IPersistentObject
{
    WorldObjectSaveData Save();
    void Load(WorldObjectSaveData data);
}
```

所有持久化对象（TreeController、StoneController、ChestController、CropController）统一遵循此接口。
