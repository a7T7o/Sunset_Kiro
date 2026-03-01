# 全面重构 - 开发记忆

## 模块概述

基于外部架构师（Code Reaper）的全域深度剖析，对项目进行系统性清理和重构。

## 当前状态

- **完成度**: 85%
- **最后更新**: 2026-02-06
- **状态**: 🚧 进行中

---

## 子工作区索引

| 编号 | 名称 | 状态 | 说明 |
|------|------|------|------|
| 0 | 初步工作 | ✅ 已完成 | 核心系统归一化 |
| 1 | 入门工作 | ✅ 已完成 | 基础清理 |
| 2 | 完善工作 | ✅ 已完成 | 交互系统修复 |
| 3 | 装备与工具 | ✅ 已完成 | Phase 3 军火库行动 |
| 3_微调 | 微调 | ✅ 已完成 | 细节优化 |
| 3.5 | 终极技术债务清算 | ✅ 已完成 | 数据层净化、存档补完、交互逻辑重构 |
| 3.6 | 债务清算后的清扫 | ✅ 已完成 | Registry 生命周期修复、反向修剪逻辑 |
| 3.7.1 | 玩家瞬移解决 | ✅ 已完成 | 修复 Tool 子物体错误标签导致的位置问题 |
| 3.7.2 | 存档bug大清扫 | ✅ 已完成 | 动态对象重建机制（树苗、掉落物） |
| 3.7.3 | 消失bug | ✅ 已完成 | GUID 漂移修复 + 自动化 GUID 管理系统 |
| 3.7.4 | 进一步bug清扫 | ✅ 已完成 | 石头假死机制、箱子 IsEmpty 修复、掉落物关联 |
| 3.7.5 | 自动化存档 | ✅ 已完成 | PrefabDatabase 自动化方案、渲染层级存档 |
| 3.7.6 | 再再进一步bug清扫 | ✅ 已完成 | 树木季节渐变存档、箱子内容加载顺序修复 |

---

## 重大里程碑

### Phase 3.5 - 终极技术债务清算（2026-01-31）

**三大战役**:
1. **战役 A: 数据层净化** - 创建 ItemDataEditor，保留 OnValidate 核心验证
2. **战役 B: 存档系统补完** - TreeController 实现 IPersistentObject，UI 刷新
3. **战役 C: 交互逻辑重构** - 添加农田工具距离检测（严禁删除 Raycast）

**关键决策**:
- 保留 OnValidate 核心验证（批量工具场景需要数据验证安全网）
- 严禁删除 Raycast（用于可交互物体检测，与农田工具无关）
- 添加 GUID 冲突自愈机制（防止 Ctrl+D 复制导致的 ID 冲突）

### Phase 3.6 - 债务清算后的清扫（2026-02-01）

**核心修复**:
- Registry 生命周期问题（跨场景残留导致 GUID 错位）
- 反向修剪逻辑（被删除的物体"复活"问题）
- 玩家位置不同步
- 箱子 IsEmpty 状态问题

**关键教训**:
- 💀 在 LoadGame 开头调用 Clear() 是"自杀式"操作
- 正确做法：使用 PruneStaleRecords() 只清理空引用

### Phase 3.7.1 - 玩家瞬移解决（2026-02-01）

**问题**: 读档时玩家本体不动，但 Tool 子物体瞬移到存档位置

**根因**: Tool 子物体错误设置了 "Player" 标签，FindGameObjectWithTag 返回了错误对象

**解决方案**: 新增 FindPlayerRoot() 方法，通过 PlayerMovement 组件确认真正的 Player

### Phase 3.7.2 - 存档bug大清扫（2026-02-01）

**问题**: 动态生成对象（树苗、掉落物）在重启游戏后读档消失

**解决方案**:
1. 创建 PrefabRegistry 用于 prefabId → Prefab 映射
2. 创建 DynamicObjectFactory 实现动态重建
3. TreeController/WorldItemPickup 实现完整的 Save/Load
4. 三道质量封印（DropDataDTO、数据验证、UpdateSprite 顺序）

### Phase 3.7.3 - 消失bug（2026-02-01 ~ 2026-02-03）

**问题**: prefabId 格式错误（存档中存的是 `M1 (5)` 而不是 `M1`）

**解决方案**:
1. GetPrefabId() 使用正则清洗 Unity 后缀
2. DynamicObjectFactory 添加工厂容错
3. Getter 提纯（不再自动生成 ID）
4. 创建自动化 GUID 管理系统（PersistentIdAutomator + PersistentIdValidator）

**关键教训**:
- Unity 会给重复或实例化的物体自动加后缀 `(1)`, `(Clone)`
- 必须用正则清洗名字才能作为 ID

### Phase 3.7.4 - 进一步bug清扫（2026-02-03）

**修复内容**:
1. **石头假死机制** - DestroyStone() 改为 SetActive(false)
2. **箱子 IsEmpty 修复** - 只检查 _inventory，不检查 _inventoryV2
3. **石头 Load() 激活** - 添加 SetActive(true) 和 ResourceNodeRegistry 重新注册
4. **掉落物反向修剪** - 动态对象使用 Destroy()，静态对象使用 SetActive(false)
5. **箱子实时同步** - 订阅 OnInventoryChanged 事件
6. **石头动态重建** - DynamicObjectFactory 支持石头重建
7. **掉落物关联机制** - sourceNodeGuid 字段链接来源
8. **StoneDebris 清理** - 加载存档时清理临时碎片
9. **箱子动态重建** - DynamicObjectFactory 支持箱子重建

### Phase 3.7.5 - 自动化存档（2026-02-03 ~ 2026-02-05）

**核心改进**:
1. **PrefabDatabase** - 替代 PrefabRegistry，支持自动扫描、智能回退
2. **ID 别名映射** - 解决旧存档 ID 与新系统不兼容问题
3. **渲染层级存档** - 保存 sortingLayerName 和 sortingOrder
4. **箱子销毁修复** - OnDestroyed() 删除整个箱子物体（包括父物体）

**新增文件**:
- `Assets/YYY_Scripts/Data/Core/PrefabDatabase.cs`
- `Assets/Editor/PrefabDatabaseEditor.cs`
- `Assets/Editor/PrefabDatabaseAutoScanner.cs`
- `Assets/111_Data/Database/PrefabDatabase.asset`

### Phase 3.7.6 - 再再进一步bug清扫（2026-02-05）

**修复内容**:
1. **树木季节渐变存档** - 使用 persistentId.GetHashCode() 作为种子（替代 GetInstanceID()）
2. **渐变进度更新** - SetSeason() 中添加 UpdateVegetationSeason() 调用
3. **箱子内容加载顺序** - Initialize() 添加 null 检查，避免覆盖 Load() 恢复的数据

**关键洞察**:
- 不需要存储 hasTransitioned，因为可以实时推导
- 只需修改种子计算方式即可保证渐变状态一致

---

## 技术债务清单

### 已清理 ✅

| 债务 | 解决方案 | 完成日期 |
|------|---------|---------|
| GameInputManager 违反 SRP | 添加距离检测，保留 Raycast | 2026-01-31 |
| TreeController 未实现 IPersistentObject | 完整实现 Save/Load | 2026-01-31 |
| Registry 跨场景残留 | PruneStaleRecords() 替代 Clear() | 2026-02-01 |
| Tool 子物体错误标签 | FindPlayerRoot() 方法 | 2026-02-01 |
| 动态对象重建缺失 | DynamicObjectFactory + PrefabRegistry | 2026-02-01 |
| GUID 漂移问题 | 自动化 GUID 管理系统 | 2026-02-02 |
| 石头假死机制缺失 | DestroyStone() 改为 SetActive(false) | 2026-02-03 |
| 箱子双数据源不同步 | IsEmpty 只检查 _inventory | 2026-02-03 |
| PrefabRegistry 手动配置 | PrefabDatabase 自动扫描 | 2026-02-03 |
| 渲染层级未存档 | WorldObjectSaveData 扩展 | 2026-02-05 |
| 季节渐变种子不稳定 | 使用 persistentId.GetHashCode() | 2026-02-05 |

### 待处理 ⏳

| 债务 | 优先级 | 说明 |
|------|--------|------|
| 异步存档/读档 | P2 | 用户要求"加载动画"，当前先同步实现 |
| 房子位置存档 | P3 | 用户确认暂不处理 |

---

## 关键文件索引

### 存档系统核心

| 文件 | 说明 |
|------|------|
| `Assets/YYY_Scripts/Data/Core/SaveManager.cs` | 存档管理器 |
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 持久化对象注册中心 |
| `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs` | 动态对象工厂 |
| `Assets/YYY_Scripts/Data/Core/PrefabDatabase.cs` | 预制体数据库（替代 PrefabRegistry） |
| `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` | 存档数据结构 |

### 自动化工具

| 文件 | 说明 |
|------|------|
| `Assets/Editor/PersistentIdAutomator.cs` | 场景保存时自动修复 GUID |
| `Assets/Editor/PersistentIdValidator.cs` | 手动验证工具 |
| `Assets/Editor/PrefabDatabaseAutoScanner.cs` | 预制体自动扫描器 |
| `Assets/Editor/PrefabDatabaseEditor.cs` | 预制体数据库编辑器 |

### 世界对象控制器

| 文件 | 说明 |
|------|------|
| `Assets/YYY_Scripts/Controller/TreeController.cs` | 树木控制器 |
| `Assets/YYY_Scripts/Controller/StoneController.cs` | 石头控制器 |
| `Assets/YYY_Scripts/World/Placeable/ChestController.cs` | 箱子控制器 |
| `Assets/YYY_Scripts/World/WorldItemPickup.cs` | 掉落物 |

---

## 经验教训总结

### 1. Unity 命名陷阱
- Unity 会给重复或实例化的物体自动加后缀 `(1)`, `(Clone)`
- 必须用正则清洗名字才能作为 ID

### 2. 日志可能具有误导性
- 日志显示"成功"不代表真的成功
- 当日志和实际行为不一致时，要怀疑代码操作的对象是否正确

### 3. Unity Tag 的陷阱
- FindGameObjectWithTag() 返回顺序不确定
- 子物体也可以有 Tag，可能导致意外的查找结果
- 最佳实践：通过组件类型查找对象，而不是 Tag

### 4. 原地读档 vs 换场景读档
- 原地读档模式下，Registry 是连接存档数据和场景实例的唯一桥梁
- 绝对不能 Clear()，只能 PruneStaleRecords()

### 5. 动态对象 vs 静态对象
- 静态对象：场景预设，GUID 序列化在场景文件
- 动态对象：运行时生成，需要动态重建机制

### 6. 执行顺序很重要
- Load() 可能先于 Start() 执行
- Initialize() 需要检查是否已被 Load() 初始化

---

## 会话记录摘要

详细会话记录请查看各子工作区的 memory.md 文件。

| 日期 | 工作区 | 主要内容 |
|------|--------|---------|
| 2026-01-27 | 主工作区 | 锐评审视，Phase 1-2 核心系统归一化 |
| 2026-01-30 | 3_装备与工具 | Phase 3 军火库行动，装备槽位拖拽修复 |
| 2026-01-31 | 3.5 | 终极技术债务清算，三大战役 |
| 2026-02-01 | 3.6 | Registry 生命周期修复，反向修剪逻辑 |
| 2026-02-01 | 3.7.1 | 玩家瞬移解决 |
| 2026-02-01 | 3.7.2 | 动态对象重建机制 |
| 2026-02-01~03 | 3.7.3 | GUID 漂移修复，自动化 GUID 管理 |
| 2026-02-03 | 3.7.4 | 石头/箱子/掉落物全面修复 |
| 2026-02-03~05 | 3.7.5 | PrefabDatabase 自动化方案 |
| 2026-02-05 | 3.7.6 | 季节渐变存档，箱子加载顺序修复 |



---

## 🔴 代码文件追踪汇总（2026-02-06 更新）

### 存档系统核心（跨多个子工作区修改）

| 文件 | 涉及的子工作区 | 最后修改 | 操作 |
|------|---------------|---------|------|
| `SaveManager.cs` | 3.5, 3.6, 3.7.2, 3.7.4, 3.7.5 | 2026-02-05 | 修改 |
| `PersistentObjectRegistry.cs` | 3.5, 3.6, 3.7.3 | 2026-02-03 | 修改 |
| `DynamicObjectFactory.cs` | 3.7.2, 3.7.3, 3.7.4 | 2026-02-03 | 新增 |
| `PrefabDatabase.cs` | 3.7.5 | 2026-02-05 | 新增（替代 PrefabRegistry） |
| `SaveDataDTOs.cs` | 3.7.2, 3.7.4, 3.7.5 | 2026-02-05 | 修改 |

### 自动化工具（Editor）

| 文件 | 涉及的子工作区 | 最后修改 | 操作 |
|------|---------------|---------|------|
| `PersistentIdAutomator.cs` | 3.7.3 | 2026-02-02 | 新增 |
| `PersistentIdValidator.cs` | 3.7.3 | 2026-02-02 | 新增 |
| `PrefabDatabaseAutoScanner.cs` | 3.7.5 | 2026-02-05 | 新增 |
| `PrefabDatabaseEditor.cs` | 3.7.5 | 2026-02-05 | 新增 |

### 世界对象控制器

| 文件 | 涉及的子工作区 | 最后修改 | 操作 |
|------|---------------|---------|------|
| `TreeController.cs` | 3.5, 3.7.2, 3.7.6 | 2026-02-05 | 修改 |
| `StoneController.cs` | 3.7.4 | 2026-02-03 | 修改 |
| `ChestController.cs` | 3.7.4, 3.7.6 | 2026-02-05 | 修改 |
| `WorldItemPickup.cs` | 3.7.2, 3.7.4 | 2026-02-03 | 修改 |

### 服务层

| 文件 | 涉及的子工作区 | 最后修改 | 操作 |
|------|---------------|---------|------|
| `PersistentManagers.cs` | 1（SO设计系统） | 2026-01-08 | 新增 |
| `TimeManager.cs` | 1（SO设计系统） | 2026-01-08 | 修改 |
| `SeasonManager.cs` | 1（SO设计系统）, 3.7.6 | 2026-02-05 | 修改 |
| `InventoryBootstrap.cs` | 3.5 | 2026-01-31 | 修改 |

### 输入与交互

| 文件 | 涉及的子工作区 | 最后修改 | 操作 |
|------|---------------|---------|------|
| `GameInputManager.cs` | 3.5, 农田系统 | 2026-01-27 | 修改 |

### 已删除/废弃的文件

| 文件 | 删除日期 | 原因 |
|------|---------|------|
| `PrefabRegistry.cs` | 2026-02-05 | 被 PrefabDatabase.cs 替代 |
