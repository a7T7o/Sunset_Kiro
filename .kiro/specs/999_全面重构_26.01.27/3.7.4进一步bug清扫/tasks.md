# 存档系统 Bug 修复 - 任务列表

**创建日期**: 2026-02-03  
**版本**: 1.2（补充 Load() 激活逻辑任务）  
**状态**: 执行中

---

## P0 - 紧急修复

### 1. 石头假死机制

- [x] 1.1 修改 `StoneController.DestroyStone()` 方法
  - [x] 1.1.1 将 `Destroy(gameObject)` 改为 `SetActive(false)`
  - [x] 1.1.2 移除从 `PersistentObjectRegistry` 注销的代码（如果有）
  - [x] 1.1.3 保留从 `ResourceNodeRegistry` 注销的代码
  - [x] 1.1.4 添加调试日志

- [x] 1.2 验证 `ShouldSave` 属性
  - [x] 1.2.1 确认 `isDepleted = true` 时返回 `false`
  - [x] 1.2.2 确认 `gameObject.activeInHierarchy = false` 时返回 `false`

- [x] 1.3 🔴 修改 `StoneController.Load()` 方法（致命漏洞修复）
  - [x] 1.3.1 添加 `isDepleted = false` 重置
  - [x] 1.3.2 添加 `SetActive(true)` 激活对象
  - [x] 1.3.3 添加 `ResourceNodeRegistry.Register()` 重新注册
  - [x] 1.3.4 更新调试日志

- [x] 1.4 处理 `StoneDebris` 清理逻辑
  - [x] 1.4.1 在场景加载时清理所有 StoneDebris（临时碎片效果）

### 2. 箱子 IsEmpty 修复

- [x] 2.1 修改 `ChestController.IsEmpty` 属性
  - [x] 2.1.1 只检查 `_inventory`，移除对 `_inventoryV2` 的检查
  - [x] 2.1.2 添加注释说明修改原因

### 3. 掉落物反向修剪修复

- [x] 3.1 修改 `WorldItemPickup` 添加注册逻辑
  - [x] 3.1.1 在 `Start()` 中注册到 `PersistentObjectRegistry`
  - [x] 3.1.2 在 `OnDestroy()` 中从 `PersistentObjectRegistry` 注销

- [x] 3.2 修改 `PersistentObjectRegistry.RestoreAllFromSaveData()` 反向修剪逻辑
  - [x] 3.2.1 区分动态对象（掉落物）和静态对象（石头、树木）
  - [x] 3.2.2 动态对象使用 `Destroy()` 销毁
  - [x] 3.2.3 静态对象使用 `SetActive(false)` 禁用

---

## P1 - 重要修复

### 4. 箱子实时同步

- [x] 4.1 在 `ChestController` 中订阅 `ChestInventory.OnInventoryChanged` 事件
  - [x] 4.1.1 在 `Initialize()` 中订阅 `_inventory.OnInventoryChanged`
  - [x] 4.1.2 事件处理器调用 `SyncInventoryToV2()`

> 注：`ChestInventory` 已有 `OnInventoryChanged` 事件，无需添加

### 5. 石头动态重建支持

- [x] 5.1 在 `DynamicObjectFactory` 中添加石头重建
  - [x] 5.1.1 在 `TryReconstruct()` 中添加 "Stone" 类型判断
  - [x] 5.1.2 实现 `TryReconstructStone()` 方法
  - [x] 5.1.3 添加 `StoneSaveData` 解析逻辑

- [x] 5.2 在 `StoneController` 中添加 GUID 设置方法
  - [x] 5.2.1 添加 `SetPersistentIdForLoad()` 方法

### 9. 箱子动态重建支持（新增）

- [x] 9.1 在 `DynamicObjectFactory` 中添加箱子重建
  - [x] 9.1.1 在 `TryReconstruct()` 中添加 "Chest" 类型判断
  - [x] 9.1.2 实现 `TryReconstructChest()` 方法
  - [x] 9.1.3 添加 `ChestSaveData` 解析逻辑

- [x] 9.2 在 `ChestController` 中添加 GUID 设置方法
  - [x] 9.2.1 添加 `SetPersistentIdForLoad()` 方法

- [x] 9.3 更新 `DynamicObjectFactory.SetPersistentId()` 辅助方法
  - [x] 9.3.1 添加 ChestController 支持

---

## P2 - 架构改进

### 6. 掉落物关联机制

- [x] 6.1 在 `WorldItemPickup` 中添加来源标记
  - [x] 6.1.1 添加 `sourceNodeGuid` 字段
  - [x] 6.1.2 添加 `SetSourceNodeGuid()` 方法
  - [x] 6.1.3 在 `Save()` 中保存 `sourceNodeGuid`
  - [x] 6.1.4 在 `Load()` 中恢复 `sourceNodeGuid`

- [x] 6.2 在资源节点生成掉落物时设置来源
  - [x] 6.2.1 修改 `StoneController.SpawnOreDrops()`
  - [x] 6.2.2 修改 `StoneController.SpawnStoneDrops()`

- [x] 6.3 在 `RestoreAllFromSaveData()` 中添加关联检查
  - [x] 6.3.1 如果 `sourceNodeGuid` 对应的资源节点存在且活跃，不恢复掉落物

---

## 验证任务

### 7. 石头修复验证

- [ ] 7.1 手动测试石头存档恢复
  - [ ] 7.1.1 保存存档（石头存在）
  - [ ] 7.1.2 挖掉石头
  - [ ] 7.1.3 加载存档
  - [ ] 7.1.4 验证石头恢复

### 8. 箱子修复验证

- [ ] 8.1 手动测试箱子空状态判断
  - [ ] 8.1.1 保存存档（箱子有物品）
  - [ ] 8.1.2 加载存档
  - [ ] 8.1.3 打开箱子，清空所有物品
  - [ ] 8.1.4 关闭箱子
  - [ ] 8.1.5 用镐子敲箱子
  - [ ] 8.1.6 验证触发挖取而非推动

---

## 任务说明

- `[ ]` - 必须完成的任务
- 任务按优先级排序：P0 > P1 > P2
- 所有任务必须在本次迭代完成

---

**文档结束**
