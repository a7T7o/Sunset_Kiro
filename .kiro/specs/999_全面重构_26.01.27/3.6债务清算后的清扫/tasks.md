# 存档系统严重问题修复 - 任务清单

## 概述

根据 requirements.md 和 design.md 的分析，按照锐评007的执行路径，严格按顺序执行修复。

---

## 任务列表

### 阶段 1: 核心基础设施修复（地基）

- [x] 1.1 Registry 生命周期修复
  - [x] 1.1.1 验证 `PersistentObjectRegistry.Clear()` 方法
    - ✅ Clear() 方法已存在（第 175-183 行）
    - ✅ 已正确清空 `_registry` 字典
    - ✅ 已正确清空 `_byType` 字典（按类型索引）
    - 任务：代码审查确认，无需修改
  - [x] 1.1.2 在 `SaveManager.LoadGame()` 的**第一行**调用 `PersistentObjectRegistry.Instance.Clear()`
    - ✅ 已修复！在 LoadGame() 开头添加了 Clear() 调用
    - ✅ 确保在任何恢复逻辑之前执行
    - ✅ 添加了调试日志

- [x] 1.2 反向修剪逻辑实现
  - [x] 1.2.1 修改 `PersistentObjectRegistry.RestoreAllFromSaveData()` 方法
    - ✅ 构建存档快照：将 saveDataList 的 GUID 存入 `HashSet<string> savedGuids`
    - ✅ 快照当前场景：获取当前 `_registry.Keys` 的**副本**（避免遍历时修改集合）
    - ✅ 修剪逻辑：遍历场景对象，如果存档中没有则使用 `SetActive(false)` 禁用
    - ✅ 恢复逻辑：遍历 saveDataList 进行 Load()
    - ✅ 添加修剪统计日志

---

### 阶段 2: 玩家位置修复

- [x] 2.1 修改 `SaveManager.RestorePlayerData()` 方法
  - [x] 2.1.1 暂时禁用物理组件
    - ✅ 获取 Player 的 Rigidbody2D
    - ✅ 记录原始 simulated 状态
    - ✅ 设置 simulated = false
  - [x] 2.1.2 设置玩家位置
    - ✅ 使用存档中的位置数据
  - [x] 2.1.3 强制重置 Tool 子物体
    - ✅ 查找 Tool 子物体
    - ✅ 设置 localPosition = Vector3.zero
  - [x] 2.1.4 恢复物理组件状态
    - ✅ 恢复原始 simulated 状态
  - [x] 2.1.5 强制物理同步（锐评002 锦囊1）
    - ✅ 调用 Physics2D.SyncTransforms()

---

### 阶段 3: 箱子状态修复

- [x] 3.1 完善 `ChestController.Load()` 方法
  - [x] 3.1.1 确保 `_inventory` 和 `_inventoryV2` 已初始化
    - ✅ 已实现（第 280-290 行）
    - 任务：代码审查确认
  - [x] 3.1.2 处理空箱子情况
    - ✅ 已修复！如果存档中没有槽位数据，循环清空两个库存系统的所有槽位
  - [x] 3.1.3 在 Load() 最后强制调用状态检查
    - ✅ 已调用 `UpdateSprite()` 确保视觉状态一致
    - ✅ 已添加：验证 `IsEmpty` 属性返回正确值的调试日志

---

### 阶段 4: 树木 ID 冲突修复

- [x] 4.1 修改 `TreeController.RegisterToPersistentRegistry()` 方法
  - [x] 4.1.1 将 ID 冲突警告改为调试日志
    - ✅ 代码审查确认：已经是 `Debug.Log` 且只在 `showDebugInfo` 为 true 时输出
    - 无需修改

---

### 阶段 5: 石头存档支持

- [x] 5.1 为 `StoneController` 添加 `IPersistentObject` 接口实现
  - [x] 5.1.1 添加 `_persistentId` 字段
    - ✅ 代码审查确认：已存在
  - [x] 5.1.2 实现 `PersistentId` 属性
    - ✅ 代码审查确认：已实现
  - [x] 5.1.3 实现 `ObjectType` 属性
    - ✅ 代码审查确认：返回 "Stone"
  - [x] 5.1.4 实现 `ShouldSave` 属性
    - ✅ 代码审查确认：返回 `gameObject.activeInHierarchy && !isDepleted`
  - [x] 5.1.5 实现 `Save()` 方法
    - ✅ 代码审查确认：已实现
  - [x] 5.1.6 实现 `Load()` 方法
    - ✅ 代码审查确认：已实现
  - [x] 5.1.7 在 `Start()` 中注册到 `PersistentObjectRegistry`
    - ✅ 代码审查确认：已调用 `RegisterToPersistentRegistry()`
  - [x] 5.1.8 在 `OnDestroy()` 中从 `PersistentObjectRegistry` 注销
    - ✅ 代码审查确认：已调用 `UnregisterFromPersistentRegistry()`

- [x] 5.2 创建 `StoneSaveData` 数据结构
  - [x] 5.2.1 在 `SaveDataDTOs.cs` 中添加 `StoneSaveData` 类
    - ✅ 代码审查确认：已存在（第 315 行）

---

### 阶段 6: 验证测试

- [ ] 6.1 Registry 测试
  - [ ] 6.1.1 创建新存档，砍伐部分树木
  - [ ] 6.1.2 保存游戏
  - [ ] 6.1.3 加载存档
  - [ ] 6.1.4 验证：控制台没有 ID 冲突警告
  - [ ] 6.1.5 验证：树木状态正确恢复

- [ ] 6.2 玩家位置测试
  - [ ] 6.2.1 移动玩家到特定位置
  - [ ] 6.2.2 保存游戏
  - [ ] 6.2.3 移动玩家到其他位置
  - [ ] 6.2.4 加载存档
  - [ ] 6.2.5 验证：玩家回到保存时的位置
  - [ ] 6.2.6 验证：Tool 子物体 localPosition 为 (0,0,0)

- [ ] 6.3 箱子测试
  - [ ] 6.3.1 放置箱子并放入物品
  - [ ] 6.3.2 保存游戏
  - [ ] 6.3.3 清空箱子内容
  - [ ] 6.3.4 加载存档
  - [ ] 6.3.5 验证：箱子内容恢复
  - [ ] 6.3.6 验证：空箱子可以被镐子挖取

- [ ] 6.4 反向修剪测试
  - [ ] 6.4.1 砍掉一棵树
  - [ ] 6.4.2 保存游戏
  - [ ] 6.4.3 加载存档
  - [ ] 6.4.4 验证：被砍掉的树没有"复活"

- [ ] 6.5 石头测试
  - [ ] 6.5.1 挖掘部分石头
  - [ ] 6.5.2 保存游戏
  - [ ] 6.5.3 加载存档
  - [ ] 6.5.4 验证：石头状态恢复到保存时的阶段和血量

---

## 修改文件清单

| 文件 | 修改内容 | 现状 |
|------|---------|------|
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 修改 RestoreAllFromSaveData() 添加反向修剪 | ✅ 已完成 |
| `Assets/YYY_Scripts/Data/Core/SaveManager.cs` | LoadGame() 开头调用 Clear()，修改 RestorePlayerData() | ✅ 已完成 |
| `Assets/YYY_Scripts/World/Placeable/ChestController.cs` | 完善 Load() 方法处理空箱子 | ✅ 已完成 |
| `Assets/YYY_Scripts/Controller/TreeController.cs` | 修改 ID 冲突警告级别 | ✅ 代码审查确认无需修改 |
| `Assets/YYY_Scripts/Controller/StoneController.cs` | 添加 IPersistentObject 实现 | ✅ 代码审查确认已实现 |
| `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` | 添加 StoneSaveData 类 | ✅ 代码审查确认已存在 |

---

## 执行顺序（严禁跳步！）

```
1. [Core] Registry 修复 (1.1)
   ↓
2. [Core] 反向修剪 (1.2)
   ↓
3. [Player] 位置修正 (2.1)
   ↓
4. [Chest] 状态修复 (3.1)
   ↓
5. [Tree] ID 冲突修复 (4.1)
   ↓
6. [Stone] 存档支持 (5.1, 5.2)
   ↓
7. [Test] 验证测试 (6.1 - 6.5)
```

---

## 风险提示

1. **Registry 清空时机**：必须在场景对象重新注册之前清空，否则会出现假冲突
2. **反向修剪的销毁方式**：使用 `SetActive(false)` 而非 `Destroy()`，避免影响对象池
3. **物理引擎覆盖**：设置玩家位置时必须暂时禁用 Rigidbody2D
4. **双库存同步**：箱子的 `_inventory` 和 `_inventoryV2` 必须保持一致
