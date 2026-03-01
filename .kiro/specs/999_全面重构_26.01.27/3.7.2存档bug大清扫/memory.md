# 存档 Bug 大清扫 - 开发记忆

## 模块概述

解决动态生成对象（树苗、掉落物等）在重启游戏后读档消失的问题。核心是实现"动态对象重建机制"。

## 当前状态

- **完成度**: 95%
- **最后更新**: 2026-02-01
- **状态**: 代码实现完成，待用户在 Unity 中配置 PrefabRegistry 并验证

## 问题分析

### 现象

1. **运行中保存 → 运行中加载**：25/25 匹配，树苗都在 ✅
2. **停止运行 → 重新运行 → 加载**：树苗消失 ❌（石头没有消失）

### 根因

| 对象类型 | 来源 | GUID 存储位置 | 重启后状态 |
|---------|------|--------------|-----------|
| 石头（静态） | 场景预设 | 序列化在场景文件 | 存在，GUID 不变 |
| 树苗（动态） | 运行时放置 | 仅在内存中 | 不存在，GUID 丢失 |

### 存档数据分析（slot1.json）

- 22 个 Tree 对象，全部 `growthStageIndex: 0`（树苗）
- 3 个 Stone 对象
- **所有对象的 `prefabId` 都是空字符串 `""`**

### 核心矛盾

当前存档系统只管"恢复数据"，不管"恢复存在"：
- `FindByGuid(guid)` 找不到对象 → 数据被丢弃
- 缺少"找不到就创建"的重建逻辑

## 锐评处理记录

### 锐评001（锐评013：动态对象的生死簿）

**核心指令**：
1. 数据结构升级：增加 prefabId、season、isStump
2. 保存逻辑：TreeController.Save() 写入 Prefab 标识
3. 加载逻辑：找不到 GUID 时重建对象
4. 掉落物持久化

**我的回应**：
- ✅ 认同核心问题诊断和解决方向
- ⚠️ 疑虑：TreeController 没有 prefabId 字段，需要新增
- ⚠️ 疑虑：预制体不在 Resources 目录，必须用 PrefabRegistry
- ⚠️ 疑虑：season 和 isStump 可以推断，只需要增加 stumpHealth

### 锐评002（锐评014：架构漏洞修补与性能警示）

**3 个盲区**：
1. 旧存档兼容性（prefabId 为空）
2. PrefabRegistry 内存问题
3. 季节与状态的视觉闪烁

**我的回应**：
- ✅ 盲区一：添加 Legacy Fallback 逻辑
- ⚠️ 盲区二：当前规模可接受，添加空值检查
- ✅ 盲区三：采用方案 B（先禁用后启用）

### 锐评003（锐评015：战略妥协与底线坚持）

**裁决**：
- ✅ 接受我对 position 的反驳
- ✅ 批准防闪烁方案 B
- ❌ 驳回掉落物延后
- ⚠️ 有条件接受季节字段省略

**我的回应**：
- ✅ 接受掉落物必须实现
- ✅ 接受季节加载顺序警告

### 锐评004-005（锐评016：战略妥协与底线防守）

**技术评估**：
- 掉落物不需要复杂的 prefabId 映射，使用 WorldSpawnService.SpawnById() 即可
- 项目中树木预制体命名是 M1/M2/M3

**我的回应**：
- ✅ 验证技术评估正确
- ✅ 简化掉落物重建逻辑
- ✅ 确认 Legacy Fallback 默认预制体为 M1

### 锐评006（锐评017：全面方案核准与隐形雷区排查）

**评审结果**：🟢 全面通过 (Approved with Distinction)

**2 个排雷补丁**：

| 雷区 | 问题描述 | 解决方案 |
|------|---------|---------|
| 雷区一 | genericData 序列化陷阱 | 必须定义 DropDataDTO 类，不用匿名对象 |
| 雷区二 | Legacy Fallback 空指针 | 回退逻辑仅限于有效数据，损坏数据直接跳过 |

**我的回应**：
- ✅ 完全接受 DropDataDTO 类定义
- ✅ 完全接受数据有效性检查
- ✅ 确认使用 M1 作为默认预制体
- ✅ 确认季节渐变状态存档设计
- ✅ 确认先同步后异步方案

## 用户补充需求

### 1. 季节渐变状态存档

**用户原话**：
> 所有树木自然渐变然后改变后不会回退，只有季节顺时针闭环变更

**我的理解**：
- 当前实现：每次根据 progress 和 treeSeedValue 实时计算
- 问题：动态重建后 treeID 可能变化，导致渐变状态不一致
- 解决方案：新增 `hasTransitionedToNextSeason` 和 `transitionVegetationSeason` 字段

### 2. 全面的场景对象存档

**用户原话**：
> 凡是出现在场景中的所有数据你都需要保存

**已实现 IPersistentObject**：
- TreeController ✅
- StoneController ✅
- ChestController ✅
- InventoryService ✅
- EquipmentService ✅

**需要新增**：
- WorldItemPickup（掉落物）- 本次必须实现

### 3. 异步存档/读档

**用户原话**：
> 可以用多线程，给一个加载动画在前面播放

**我的建议**：
- 本次先同步实现，确保功能正确
- 后续再升级为异步版本

## 本次实现范围

| 功能 | 优先级 | 状态 |
|------|--------|------|
| 树苗动态重建 | P0 | ✅ 已实现 |
| 掉落物动态重建 | P0 | ✅ 已实现 |
| 旧存档兼容 | P0 | ✅ 已实现 |
| 防闪烁方案 | P0 | ✅ 已实现 |
| 树木季节渐变状态存档 | P1 | ✅ 已实现（字段已添加） |
| 树桩血量存档 | P1 | ✅ 已实现 |

## 会话记录

### 会话 1 - 2026-02-01

**用户需求**:
> 现在还有一个特别严重的问题...重新运行游戏，然后点击加载存档，这个时候就会少了很多树苗

**完成任务**:
1. 分析问题根因：动态对象 vs 静态对象的 GUID 生命周期差异
2. 阅读锐评001，理解架构重构方案
3. 创建三件套文档

**修改文件**:
- `memory.md` - 创建
- `requirements.md` - 创建
- `design.md` - 创建
- `tasks.md` - 创建

---

### 会话 2 - 2026-02-01（续）

**用户需求**:
> 你确定审视完毕了这个锐评内容吗，你有进行你自己全面的思考吗？

**完成任务**:
1. 创建 `锐评回应与独立思考.md` 文档
2. 对锐评001进行独立审视，提出 5 个疑虑
3. 对锐评002进行独立审视，回应 3 个盲区
4. 对锐评003进行独立审视，接受掉落物必须实现
5. 分析用户补充需求：季节渐变状态、全面存档、异步存档

**修改文件**:
- `锐评回应与独立思考.md` - 创建并更新
- `memory.md` - 更新

**下一步**:
- 用户审核独立思考文档
- 确认后更新 design.md 和 tasks.md
- 开始执行任务

---

### 会话 3 - 2026-02-01（续）

**用户需求**:
> 004和005并非是重复的文件，你均需要仔细阅读...请你对这两个锐评进行详细的思考和专业的回应

**完成任务**:
1. 仔细阅读锐评004和锐评005
2. 检查 WorldItemPickup.cs 和 WorldSpawnService.cs 代码
3. 验证锐评中的技术评估（掉落物不需要复杂的 prefabId 映射）
4. 检查项目中的树木预制体命名（M1, M2, M3）
5. 更新 `锐评回应与独立思考.md`，添加锐评004-005回应
6. 创建 `全面更新方案.md`，详细说明三件套文档的更新计划

**修改文件**:
- `锐评回应与独立思考.md` - 追加锐评004-005回应
- `全面更新方案.md` - 创建

**关键发现**:
1. 锐评004的技术评估正确：掉落物不需要 PrefabRegistry，使用 WorldSpawnService.SpawnById() 即可
2. 项目中树木预制体命名是 M1/M2/M3，不是 Tree_Oak
3. Legacy Fallback 默认预制体应该是 `M1`

**下一步**:
- 用户审核全面更新方案
- 确认后按顺序更新三件套文档

---

### 会话 4 - 2026-02-01（续）

**用户需求**:
> 你好像并没有完全处理完006，请你确保你审核完毕，保证完整的审核流程

**完成任务**:
1. 完整阅读锐评006（锐评017：全面方案核准与隐形雷区排查）
2. 独立审视 2 个排雷补丁：
   - 雷区一：genericData 序列化陷阱 → 必须定义 DropDataDTO 类
   - 雷区二：Legacy Fallback 空指针 → 回退逻辑仅限于有效数据
3. 验证锐评指出的问题确实存在
4. 更新 `锐评回应与独立思考.md`，添加锐评006回应（第十三、十四章节）
5. 更新 `memory.md`，记录锐评006内容和本次会话

**修改文件**:
- `锐评回应与独立思考.md` - 追加锐评006回应
- `memory.md` - 更新锐评处理记录和会话记录

**关键发现**:
1. 雷区一是真实问题：Unity JsonUtility 需要明确类型才能反序列化
2. 雷区二是真实风险：损坏数据可能导致"幽灵树"
3. 锐评006 给出了最终绿灯，可以开始更新三件套文档

**下一步**:
- 更新 requirements.md（添加 US-4、US-5）
- 更新 design.md（添加 DropDataDTO、排雷补丁）
- 更新 tasks.md（添加阶段 7-9）
- 开始执行任务

---

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 使用 prefabId 而非 Resources 路径 | 更灵活，支持非 Resources 目录的预制体 | 2026-02-01 |
| 在 RestoreAllFromSaveData 中实现重建 | 集中管理，避免分散逻辑 | 2026-02-01 |
| 必须使用 PrefabRegistry | 预制体在 Assets/222_Prefabs/Tree/，不在 Resources | 2026-02-01 |
| 掉落物本次必须实现 | 锐评003 驳回延后，用户也明确要求 | 2026-02-01 |
| 季节渐变状态需要存档 | 用户明确要求"渐变不可逆" | 2026-02-01 |
| 异步存档后续实现 | 先确保功能正确，再优化性能 | 2026-02-01 |
| 掉落物使用 WorldSpawnService | 不需要 PrefabRegistry，简化实现 | 2026-02-01 |
| Legacy Fallback 默认预制体 M1 | 项目中树木预制体是 M1/M2/M3 | 2026-02-01 |
| 必须定义 DropDataDTO 类 | JsonUtility 需要明确类型才能反序列化 | 2026-02-01 |
| Legacy Fallback 数据验证 | 避免损坏数据导致"幽灵树" | 2026-02-01 |

## 待用户确认

~~1. **全面更新方案**：请审核 `全面更新方案.md`，确认三件套文档的更新计划~~
~~2. **Legacy Fallback 默认预制体**：使用 `M1` 作为默认预制体是否可接受？~~
~~3. **季节渐变状态**：我理解的"渐变不可逆"是否正确？~~
~~4. **异步存档**：是否接受"本次先同步，后续再异步"的方案？~~

**锐评006 已全面批准以上所有方案，可以开始更新三件套文档。**

### 会话 5 - 2026-02-01（续）

**用户需求**:
> 完整审核锐评006并开始执行代码任务

**完成任务**:
1. 任务1：扩展 TreeSaveData - 添加 season, isStump, stumpHealth, hasTransitionedToNextSeason, transitionVegetationSeason 字段
2. 任务2：新增 DropDataDTO（🛡️ 封印一）- 带 `[Serializable]` 特性
3. 任务3：创建 PrefabRegistry.cs - ScriptableObject 用于 prefabId → Prefab 映射
4. 任务4：PrefabRegistry.asset 需要用户在 Unity 中手动创建
5. 任务5：创建 DynamicObjectFactory.cs - TryReconstruct() + 数据验证（🛡️ 封印二）
6. 任务6：修改 TreeController.Save() - 添加 GetPrefabId() 方法
7. 任务7：添加 TreeController.SetPersistentIdForLoad() 方法
8. 任务8：修改 TreeController.Load()（🛡️ 封印三）- UpdateSprite() 是最后一行
9. 任务9：WorldItemPickup 实现 IPersistentObject 接口
10. 任务10：修改 PersistentObjectRegistry.RestoreAllFromSaveData() - 动态重建逻辑
11. 任务11：在 SaveManager.Awake() 中添加 InitializeDynamicObjectFactory()

**修改文件**:
- `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` - TreeSaveData 扩展 + DropDataDTO 新增
- `Assets/YYY_Scripts/Data/Core/PrefabRegistry.cs` - 新建
- `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs` - 新建
- `Assets/YYY_Scripts/Controller/TreeController.cs` - Save/Load/SetPersistentIdForLoad/GetPrefabId
- `Assets/YYY_Scripts/World/WorldItemPickup.cs` - 实现 IPersistentObject
- `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` - RestoreAllFromSaveData 重建逻辑
- `Assets/YYY_Scripts/Data/Core/SaveManager.cs` - InitializeDynamicObjectFactory

**编译状态**: ✅ 成功（只有4个无关警告）

**3道质量封印验证**:
- ✅ 封印一：DropDataDTO 有 `[Serializable]` 特性
- ✅ 封印二：DynamicObjectFactory 在 Instantiate 前验证数据有效性
- ✅ 封印三：TreeController.Load() 最后一行是 UpdateSprite()

**待用户操作**:
1. 在 Unity 中创建 `Assets/111_Data/Database/PrefabRegistry.asset`
2. 配置 PrefabRegistry：添加 M1, M2, M3 预制体映射
3. 执行验证测试

---

### 会话 6 - 2026-02-01（续）

**用户需求**:
> 执行修补锐评001.md中的修正指令

**问题背景**:
用户配置完 PrefabRegistry 后测试失败：
- 不仅种下的树苗消失
- 连原本场景中预设的树木也都没加载出来
- 日志显示 `[DynamicObjectFactory] 跳过无效的树木数据（已销毁但非树桩）: guid=..., health=0`
- 日志显示 `[Registry] 存档匹配率: 7/28`

**根因分析**（锐评019）:
1. **封印二过于严苛，误杀了树苗**：树苗 (Stage 0) 的设计血量就是 0，原逻辑 `if (health <= 0 && !isStump)` 把所有树苗当成"已死亡的树"处理了
2. **GUID 漂移**：静态树木的 GUID 在重启后变了（7/28 匹配率），导致它们也走进了"重建流程"，然后被封印二拦截

**完成任务**:
1. ✅ 修正封印二逻辑 - `DynamicObjectFactory.cs`
   - 修改为：只有 Stage > 0 且不是树桩且血量<=0 时才判定为死树
   - 树苗 (Stage 0) 的血量为0是合法的
2. ✅ 日志降噪 - `InventoryBootstrap.cs`
   - 移除逐条物品添加日志，只保留汇总信息
3. ✅ 日志降噪 - `StoneController.cs`
   - 移除带星号的刷屏日志，改为 showDebugInfo 控制
4. ✅ 添加 GUID 重生成工具 - `TreeController.cs`
   - 添加 `[ContextMenu("Force Regenerate GUID")]` 方法

**修改文件**:
- `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs` - 修正封印二逻辑
- `Assets/YYY_Scripts/Service/Inventory/InventoryBootstrap.cs` - 日志降噪
- `Assets/YYY_Scripts/Controller/StoneController.cs` - 日志降噪
- `Assets/YYY_Scripts/Controller/TreeController.cs` - 添加 ForceRegenerateGUID

**编译状态**: ✅ 成功（只有4个无关警告）

**待用户操作**:
1. **重新保存场景**：在 Unity 编辑器中打开场景，按 Ctrl+S 保存（固化静态树的 GUID）
2. **重新开始游戏**：不是加载旧存档，而是开始新游戏
3. **验证测试**：种植树苗 → 保存 → 重启 → 加载 → 树苗应该恢复

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `锐评回应与独立思考.md` | 对锐评001-005的独立审视和回应 |
| `全面更新方案.md` | 三件套文档的更新计划 |
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 核心存档恢复逻辑 |
| `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs` | 存档数据结构 |
| `Assets/YYY_Scripts/Controller/TreeController.cs` | 树苗控制器 |
| `Assets/YYY_Scripts/World/WorldItemPickup.cs` | 掉落物（需要实现 IPersistentObject） |
| `Assets/YYY_Scripts/World/WorldSpawnService.cs` | 掉落物生成服务 |
| `Assets/YYY_Scripts/Service/Placement/PlacementManager.cs` | 放置系统 |
| `Assets/222_Prefabs/Tree/` | 树木预制体目录（M1, M2, M3） |
| `Assets/Saves/slot1.json` | 当前存档文件 |
