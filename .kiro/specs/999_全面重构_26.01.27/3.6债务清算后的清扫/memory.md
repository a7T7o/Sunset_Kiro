# 3.6 债务清算后的清扫 - 开发记忆

## 模块概述

修复存档系统的多个严重问题，包括：
- Registry 生命周期问题（跨场景残留导致 GUID 错位）
- 玩家位置不同步
- 箱子 IsEmpty 状态问题
- 树木 ID 冲突警告
- 石头存档支持缺失
- 反向修剪逻辑缺失（被删除的物体"复活"）

## 当前状态

- **完成度**: 90%（代码修复已完成，待用户测试验证）
- **最后更新**: 2026-02-01
- **状态**: ✅ 代码修复完成 - 等待用户测试验证

## 会话记录

### 会话 5 - 2026-02-01（锐评010 代码执行）

**用户需求**:
> 锐评010 审核通过，执行代码修改

**完成任务**:
1. ✅ 在 `PersistentObjectRegistry.cs` 中实现 `PruneStaleRecords()` 方法
   - 收集所有 `Value == null` 的键
   - 移除这些空引用
   - 同时清理 `_byType` 字典中的空引用
   - 添加调试日志
2. ✅ 修改 `SaveManager.LoadGame()` - 将 `Clear()` 替换为 `PruneStaleRecords()`
   - 更新注释说明原地读档模式下的正确做法
3. ✅ 在 `ChestController.Load()` 中添加 capacity 调试日志
4. ✅ 编译验证通过（4 个无关警告）

**修改文件**:
- `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` - 新增 PruneStaleRecords() 方法
- `Assets/YYY_Scripts/Data/Core/SaveManager.cs` - 将 Clear() 替换为 PruneStaleRecords()
- `Assets/YYY_Scripts/World/Placeable/ChestController.cs` - 添加 capacity 调试日志

**技术细节**:
1. **PruneStaleRecords()** 只移除 `Value == null` 的键值对，保留所有活着的对象
2. **原地读档模式**：Registry 是连接存档数据和场景实例的唯一桥梁，绝对不能 Clear()
3. **调试日志**：`[Chest] Load Capacity: {capacity}` 用于验证容量计算正确性

**编译状态**: ✅ 成功

**遗留问题**:
- [ ] 用户测试验证（功能测试任务 3.2）

---

### 会话 4 - 2026-02-01（锐评009 紧急修复）

**用户需求**:
> 锐评009 指出我犯了致命错误：在 LoadGame 开头调用 Clear() 是"自杀式"操作，导致存档系统完全失效

**问题诊断**:
- 💀 **致命错误**：我在 `SaveManager.LoadGame()` 开头调用了 `PersistentObjectRegistry.Instance.Clear()`
- **灾难流程**：
  1. 场景里的树和箱子在 `Start()` 时已注册到 Registry
  2. 玩家点击"加载"
  3. 我调用 `Clear()`，把这些活生生的对象引用全删了
  4. 数据读进来了，代码拿着 GUID 去找对象，Registry 是空的，全部找不到！
  5. **结果**：所有数据的 Restore 操作全部跳过，反向修剪逻辑也全部失效

**根因分析**:
- **原地读档**（不换场景）模式下，Registry 是连接存档数据和场景实例的唯一桥梁
- 我错误地把"换场景读档"的解决方案（Clear）应用到了"原地读档"场景
- 正确做法：只清理"已销毁但还在注册表里的空引用"（`PruneStaleRecords()`）

**完成任务**:
1. ✅ 创建 requirements01.md - 紧急修复需求文档
2. ✅ 创建 design01.md - 紧急修复设计文档
3. ✅ 创建 tasks01.md - 紧急修复任务清单
4. ✅ 检查 ChestController 硬编码 - 确认已正确实现（优先使用存档数据）

**关键发现**:
- ChestController 的 capacity 计算逻辑已正确：`chestData.capacity > 0 ? chestData.capacity : (storageData?.storageCapacity ?? 20)`
- 需要实现 `PruneStaleRecords()` 方法替代 `Clear()`

**修改文件**:
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/requirements01.md` - 新建
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/design01.md` - 新建
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/tasks01.md` - 新建
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/memory.md` - 更新

**遗留问题**:
- [ ] 实现 PruneStaleRecords() 方法
- [ ] 删除 Clear() 调用，改为 PruneStaleRecords()
- [ ] 测试验证

---

### 会话 3 - 2026-02-01（代码执行）

**用户需求**:
> 锐评002 已给出"🟢 全面通过"，开始执行代码修改

**完成任务**:
1. ✅ 任务 1.1.2：在 `SaveManager.LoadGame()` 开头调用 `PersistentObjectRegistry.Instance.Clear()`
2. ✅ 任务 1.2.1：修改 `RestoreAllFromSaveData()` 添加反向修剪逻辑
3. ✅ 任务 2.1：修改 `RestorePlayerData()` 添加物理禁用、`Physics2D.SyncTransforms()`、Tool 重置
4. ✅ 任务 3.1.2：完善 `ChestController.Load()` 处理空箱子情况
5. ✅ 任务 3.1.3：添加 IsEmpty 状态检查调试日志

**代码审查确认（无需修改）**:
- ✅ 任务 1.1.1：`Clear()` 方法已存在
- ✅ 任务 3.1.1：`_inventory` 和 `_inventoryV2` 初始化已实现
- ✅ 任务 4.1.1：TreeController 的 ID 冲突警告已经改为调试日志
- ✅ 任务 5.1 和 5.2：StoneController 已实现 IPersistentObject，StoneSaveData 已存在

**修改文件**:
- `Assets/YYY_Scripts/Data/Core/SaveManager.cs` - LoadGame() 开头调用 Clear()，RestorePlayerData() 添加物理禁用和 Tool 重置
- `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` - RestoreAllFromSaveData() 添加反向修剪逻辑
- `Assets/YYY_Scripts/World/Placeable/ChestController.cs` - Load() 添加空箱子处理和状态检查日志

**技术细节**:
1. **Registry 清空**：在 LoadGame() 第一行调用 Clear()，确保在任何恢复逻辑之前执行
2. **反向修剪**：构建存档快照 → 快照当前场景 → 修剪（SetActive(false)）→ 恢复
3. **玩家位置**：禁用 rb.simulated → 设置位置 → Physics2D.SyncTransforms() → 重置 Tool → 恢复 rb.simulated
4. **箱子空状态**：存档无槽位数据时，循环清空两个库存系统的所有槽位

**编译状态**: ✅ 成功（4 个无关警告）

**遗留问题**:
- [ ] 用户测试验证（阶段 6 的测试任务）

---

### 会话 2 - 2026-02-01（文档修缮）

**用户需求**:
> 仅仅是这样就行了吗？你的Requirement和design不需要进行修缮和优化吗？

**完成任务**:
1. 全面审视 requirements.md、design.md、tasks.md
2. 对比源代码验证文档准确性
3. 修缮 design.md - 更正 Registry Clear() 方法现状描述
4. 修缮 design.md - 完善箱子 Load() 方法的状态检查说明
5. 修缮 tasks.md - 更正任务 1.1.1 描述（Clear() 已存在）
6. 修缮 tasks.md - 完善任务 3.1 的子任务描述
7. 修缮 tasks.md - 添加修改文件清单的现状列

**关键发现**:
- `PersistentObjectRegistry.Clear()` 方法**已存在**（第 175-183 行），无需添加
- `SaveManager.LoadGame()` **没有调用** Clear()，这是需要修复的核心问题
- `ChestController.Load()` 大部分已实现，只需补充空箱子处理和状态检查日志

**修改文件**:
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/design.md` - 更正现状分析
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/tasks.md` - 更正任务描述
- `.kiro/specs/999_全面重构_26.01.27/3.6债务清算后的清扫/memory.md` - 添加本次会话记录

**遗留问题**:
- [ ] 等待用户审核文档
- [ ] 执行阶段 1-6 的任务

---

### 会话 1 - 2026-02-01（文档创建）

**用户需求**:
> 修复存档系统的四个核心问题

**完成任务**:
1. 创建 requirements.md - 需求文档
2. 创建 design.md - 设计文档
3. 创建 tasks.md - 任务清单

**关键发现**:
- 辅助文档（0辅助阶段001.md）指出了 Registry 生命周期问题是**真正的元凶**
- 单例 + DontDestroyOnLoad = 跨场景残留，导致 GUID 错位
- 反向修剪逻辑缺失导致被删除的物体"复活"

**锐评审核**:
- 锐评007 确认文档通过审核
- 指出代码处于"半成品"状态，需要按顺序填坑

**遗留问题**:
- [ ] 执行 Registry 修复
- [ ] 执行反向修剪逻辑
- [ ] 执行玩家位置修正
- [ ] 执行箱子状态修复
- [ ] 执行树木 ID 冲突修复
- [ ] 执行石头存档支持
- [ ] 验证测试

---

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| Registry 清空放在 LoadGame 第一行 | 必须在场景对象重新注册之前清空 | 2026-02-01 |
| 反向修剪使用 SetActive(false) | 避免影响对象池，比 Destroy 更安全 | 2026-02-01 |
| 玩家位置设置时禁用物理 | 防止 Rigidbody2D 覆盖位置设置 | 2026-02-01 |

## 相关文件

| 文件 | 说明 |
|------|------|
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 持久化注册表（需修改） |
| `Assets/YYY_Scripts/Data/Core/SaveManager.cs` | 存档管理器（需修改） |
| `Assets/YYY_Scripts/World/Placeable/ChestController.cs` | 箱子控制器（需完善） |
| `Assets/YYY_Scripts/Controller/TreeController.cs` | 树木控制器（需修改） |
| `Assets/YYY_Scripts/Controller/StoneController.cs` | 石头控制器（需添加接口） |
| `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` | 存档数据结构（需添加） |

## 跨工作区索引

| 相关系统 | 工作区 | 关联问题 |
|---------|--------|---------|
| 树木系统 | `.kiro/specs/树木系统/` | IPersistentObject 实现 |
| 箱子系统 | `.kiro/specs/箱子系统/` | 双库存同步问题 |

## 锐评归档

| 锐评 | 内容摘要 |
|------|---------|
| 0辅助阶段001.md | 技术顾问的深度分析，指出 Registry 生命周期是真正元凶 |
| 1准备阶段锐评001.md | 文档审核通过，指出代码紧急纠偏指令 |
