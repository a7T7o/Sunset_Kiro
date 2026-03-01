# Phase 5.0 修复与完善方案

**创建日期**: 2026-02-12
**基线文档**: 审查报告 v2（2026-02-12）
**目标**: 修复审查报告 v2 中发现的全部缺陷，补充缺失的规则文件，更新过时内容

---

## 一、问题总览

| # | 问题 | 来源 | 优先级 | 类型 |
|---|------|------|--------|------|
| 1 | 创建存档系统规则文件 `save-system.md` | 审查报告 v2 缺口 #1 | P0 | 新建 |
| 2 | 创建箱子/交互系统规则文件 `chest-interaction.md` | 审查报告 v2 缺口 #2 | P0 | 新建 |
| 3 | README.md 索引信息过时 | 审查报告 v2 缺陷 #1 | P0 | 修复 |
| 4 | communication.md 与 rules.md 冗余 | 审查报告 v2 缺陷 #3 | P1 | 标注 |
| 5 | Hook 领域映射与规则文件同步风险 | 审查报告 v2 缺陷 #4 | P1 | 补充 |
| 6 | maintenance-guidelines.md 体积膨胀 | 审查报告 v2 缺陷 #2 | P2 | 观察 |
| 7 | archive/farming.md 内容过时 | 审查报告 v2 缺陷 #5 | P1 | 更新 |
| 8 | rules.md 沟通规范整合质量审核 | 用户要求 | P1 | 审核 |

---

## 二、问题 #1：创建存档系统规则文件

### 2.1 必要性分析

存档系统是项目中最容易出错的模块之一，经历了 3.7.2→3.7.6 共 5 轮 bug 清扫。核心易错点包括：

- GUID 生命周期（静态 vs 动态对象）
- 执行顺序陷阱（Load() 可能先于 Start()）
- 动态对象重建流程
- 反向修剪机制
- Unity 命名陷阱（自动后缀清洗）

没有规则文件意味着每次涉及存档的开发都需要重新从 memory 中爬取这些知识。

### 2.2 规则文件内容规划

**文件**: `.kiro/steering/save-system.md`
**优先级**: P1（手动加载）
**触发关键词**: 存档、保存、加载、GUID、持久化、PersistentObject

**章节结构**:

#### 一、核心架构
- PersistentObjectRegistry：Dictionary 存储，反向修剪（PruneStaleRecords 而非 Clear）
- DynamicObjectFactory：动态对象重建（树木/石头/箱子/掉落物）
- PrefabDatabase：自动扫描 + ID 别名映射（替代旧 PrefabRegistry）
- SaveManager：存档管理器，协调全局存档/读档流程

#### 二、GUID 生命周期

| 对象类型 | GUID 来源 | 生成时机 | 稳定性 |
|---------|----------|---------|--------|
| 静态对象（场景预制体） | PersistentIdAutomator 自动分配 | 编辑器保存场景时 | ✅ 绝对稳定 |
| 动态对象（运行时生成） | 放置/生成时 `System.Guid.NewGuid()` | 运行时 | ✅ 存档后稳定 |
| 重建对象（读档恢复） | 从存档数据中读取 | DynamicObjectFactory.TryReconstruct | ✅ 与原始一致 |

**🔴 禁止事项**:
- 禁止使用 `GetInstanceID()` 作为持久化标识（每次运行都不同）
- 禁止在 `Awake()/Start()` 中生成新 GUID 覆盖已有值

#### 三、执行顺序陷阱（最易出错）

**核心问题**: 动态重建的对象，`Load()` 在 `Instantiate` 后立即调用，但 `Start()` 要等到下一帧。

```
DynamicObjectFactory.TryReconstruct()
  → Instantiate(prefab)
  → obj.Load(data)          ← 此时 Start() 还没执行
  → SetActive(true)
  → [下一帧] Start() 执行   ← 如果 Start() 中有初始化，会覆盖 Load() 的数据！
```

**防御措施**:
- `Initialize()` 方法必须检查数据是否已被 `Load()` 填充（null 检查）
- 箱子系统已踩过此坑：`ChestController.Initialize()` 中 `_inventory` 的 null 检查

#### 四、动态对象重建流程

```
SaveManager.LoadGame()
  → RestoreGameTimeData()           // 1. 先恢复时间（TimeManager）
  → RestoreAllFromSaveData()        // 2. 再恢复世界对象
    → 遍历 WorldObjectSaveData 列表
    → 已注册对象 → 直接调用 Load()
    → 未注册对象 → DynamicObjectFactory.TryReconstruct()
      → 根据 objectType 分发：Tree/Stone/Chest/Drop
      → Instantiate → Load → SetActive → Register
```

**🔴 加载顺序保证**: TimeManager 必须先于所有世界对象恢复，否则季节渐变等依赖时间的系统会读到默认值。

#### 五、反向修剪机制

- `PruneStaleRecords()` 而非 `Clear()`
- 只移除"存档中没有但注册表中有"的对象
- 保留"存档中有且注册表中也有"的对象
- 石头使用假死机制：`SetActive(false)` 而非 `Destroy()`

#### 六、自动化工具

| 工具 | 文件 | 用途 |
|------|------|------|
| PersistentIdAutomator | `Editor/PersistentIdAutomator.cs` | 场景保存时自动修复空/重复 GUID |
| PersistentIdValidator | `Editor/PersistentIdValidator.cs` | 手动验证工具，检查 GUID 完整性 |
| PrefabDatabaseAutoScanner | Editor 脚本 | 预制体自动扫描，维护 PrefabDatabase |

#### 七、Unity 命名陷阱

- `Instantiate()` 会自动添加 `(Clone)` 后缀
- 场景中复制对象会添加 `(1)`、`(2)` 等后缀
- `DynamicObjectFactory` 中使用正则清洗：`Regex.Replace(name, @"\s*\(.*\)$", "")`

#### 八、渲染层级存档

- `WorldObjectSaveData` 包含 `sortingLayerName` 和 `sortingOrder`
- 存档时通过 `SetSortingLayer(SpriteRenderer)` 记录
- 读档时通过 `RestoreSortingLayer(SpriteRenderer)` 恢复

### 2.3 信息来源

| 来源 | 关键内容 |
|------|---------|
| 3.7.2 memory | 动态对象重建系统、PrefabRegistry、DynamicObjectFactory |
| 3.7.3 memory | GUID 漂移修复、PersistentIdAutomator/Validator |
| 3.7.4 memory | 石头假死、箱子 IsEmpty、反向修剪、掉落物关联 |
| 3.7.5 memory | PrefabDatabase 自动化、ID 别名映射、渲染层级存档 |
| 3.7.6 memory | 季节渐变种子稳定性、加载顺序、箱子 Initialize 覆盖问题 |
| SaveDataDTOs.cs | 数据结构定义 |
| PersistentObjectRegistry.cs | 注册中心实现 |
| DynamicObjectFactory.cs | 重建逻辑 |

---

## 三、问题 #2：创建箱子/交互系统规则文件

### 3.1 必要性分析

箱子系统经历了 22 个会话的开发，涉及复杂的层级结构、UI 互斥、存档集成。核心易错点：

- 箱子层级结构（父物体 → Box 子物体，ChestController 在子物体上）
- UI 互斥逻辑（BoxPanel 与 PackagePanel）
- 存档时的 Initialize/Load 顺序问题
- 交互系统（距离检测、导航集成、Sprite Bounds 检测）

### 3.2 规则文件内容规划

**文件**: `.kiro/steering/chest-interaction.md`
**优先级**: P1（手动加载）
**触发关键词**: 箱子、交互、ChestController、BoxPanel、锁、钥匙

**章节结构**:

#### 一、箱子层级结构

箱子采用父子层级结构：

```
箱子父物体（PlaceableObject + PersistentObject）
  └── Box 子物体（ChestController + SpriteRenderer + Collider）
```

- `PlaceableObject`：放置系统管理（位置、旋转、Sorting Layer）
- `PersistentObject`：存档系统注册（GUID、注册/注销）
- `ChestController`：箱子核心逻辑（在 Box 子物体上，不在父物体上）

**🔴 易错点**: 获取 ChestController 时不能在父物体上 `GetComponent`，必须用 `GetComponentInChildren` 或直接引用子物体。

#### 二、4 种箱子类型

| 类型 | 列数 | 行数 | 容量 |
|------|------|------|------|
| 木质小箱 | 12 | 1 | 12 |
| 木质大箱 | 12 | 2 | 24 |
| 铁质小箱 | 12 | 1 | 12 |
| 铁质大箱 | 12 | 2 | 24 |

- 所有箱子固定 12 列布局（与背包一致）
- 大箱 = 2 行，小箱 = 1 行

#### 三、UI 互斥机制

BoxPanel（箱子面板）与 PackagePanel（背包面板）的互斥关系：

| 区域 | 打开箱子时 | 关闭箱子时 |
|------|-----------|-----------|
| Main 区域 | BoxPanel 显示 | BoxPanel 隐藏 |
| Top 区域 | PackagePanel 切换到 Top 模式 | PackagePanel 恢复 Main 模式 |

- 打开箱子 → PackagePanel 从 Main 移到 Top，BoxPanel 占据 Main
- 关闭箱子 → BoxPanel 隐藏，PackagePanel 回到 Main

#### 四、存档集成

**核心陷阱**: `Initialize()` 与 `Load()` 的执行顺序

```
放置新箱子：Initialize() → 创建空 inventory → 正常
读档恢复：DynamicObjectFactory.TryReconstruct() → Load(data) → [下一帧] Start()/Initialize()
```

- `Initialize()` 中必须检查 `_inventory != null`，避免覆盖 `Load()` 已恢复的数据
- 这是箱子系统实际踩过的坑（3.7.6 会话中修复）

**存档数据结构**:
- `ChestSaveData`：箱子类型、容量、物品列表
- 嵌套在 `WorldObjectSaveData` 中通过 `objectType = "Chest"` 标识

#### 五、交互系统

**距离检测**:
- 使用 Collider 底部中心（不是 bounds.center）作为交互点
- 玩家位置使用 `playerCollider.bounds.center`（遵守最高优先级规则）
- Sprite Bounds 用于可视化检测（高亮、鼠标悬停）

**导航集成**:
- 点击箱子 → 自动导航到交互距离 → 到达后自动打开
- 导航目标点 = Collider 底部中心偏移

#### 六、锁系统

| 规则 | 说明 |
|------|------|
| 锁/钥匙配对 | 每把锁对应特定钥匙 |
| 归属转移 | 箱子可以转移归属权 |
| 玩家箱子 | 即使上锁也可直接打开（不需要钥匙） |
| 其他玩家箱子 | 必须有对应钥匙才能打开 |

#### 七、已知未修复问题

1. **Up 区域未接入 InventoryInteractionManager** — 拖拽操作在 Up 区域不生效
2. **拖拽与修饰键两套系统不互通** — Shift+点击和拖拽是独立的交互路径

### 3.3 信息来源

| 来源 | 关键内容 |
|------|---------|
| 箱子系统 memory（22 个会话） | 完整开发历史、所有踩坑记录 |
| 箱子系统 8_战后清扫 | 最终 bug 修复和反省 |
| ChestController.cs | 箱子核心逻辑实现 |
| 3.7.6 memory | Initialize/Load 顺序问题修复 |

---

## 四、问题 #3：README.md 索引信息过时

### 4.1 问题描述

Phase 4.0 将 `layers.md`、`communication.md`、`workspace-memory.md` 从 always 降级为 manual，但 README.md 中：
- P0 表格仍列出这三个文件为"默认加载（always）"
- 目录结构图仍显示它们在 P0 默认加载分组中

### 4.2 修复方案

1. 更新 P0 表格：移除这三个文件，或标注为"已降级为 manual"
2. 更新目录结构图：将这三个文件移到 P1 manual 分组
3. 更新加载机制说明：反映当前实际的 2 文件默认加载 + Hook 按需加载模式
4. 更新"更新日志"章节：记录 Phase 4.0/4.1 的变更

### 4.3 优先级

P0 — 索引文件不准确会误导所有后续维护工作。

---

## 五、问题 #4：communication.md 与 rules.md 冗余

### 5.1 问题描述

Phase 4.1 会话 12 中，已将 communication.md 的核心沟通规范整合进 rules.md。但 communication.md 仍保留完整内容，存在冗余。

### 5.2 当前冗余对照

| 内容 | rules.md | communication.md | 冗余？ |
|------|---------|-----------------|--------|
| 核心原则（直接执行、简洁明了） | ✅ | ✅ | 是 |
| 方案讨论流程（5步） | ✅ | ✅ | 是 |
| 何时展示代码 | ✅ | ✅ | 是 |
| 一条龙完整流程（5步详细） | 触发入口 | ✅ 完整内容 | 否（互补） |
| 回复风格示例对比 | ❌ | ✅ | 否（独有） |
| 特殊情况处理 | ❌ | ✅ | 否（独有） |

### 5.3 处理方案

**不删除 communication.md**（遵守"禁止删除文档"规则），而是：
1. 在 communication.md 冗余章节顶部添加标注：`> ⚠️ 此章节已整合至 rules.md，以 rules.md 为准`
2. 保留 communication.md 的独有内容（一条龙完整流程、回复风格示例、特殊情况处理）
3. 这样 communication.md 被手动加载时，用户能清楚知道哪些是最新的、哪些是冗余的

### 5.4 优先级

P1 — 不影响功能，但影响维护清晰度。


---

## 六、问题 #5：Hook 领域映射与规则文件同步风险

### 6.1 问题描述

`smart-assistant.kiro.hook` 中硬编码了领域→文件的映射关系。当新增或重命名规则文件时，Hook 不会自动更新，导致映射失效。

例如：如果本次 Phase 5.0 新建了 `save-system.md` 和 `chest-interaction.md`，Hook 中没有对应的映射条目，这两个文件永远不会被自动加载。

### 6.2 修复方案

1. **立即修复**: 在 `smart-assistant.kiro.hook` 的映射中添加新文件的条目
   - 存档/保存/加载/GUID/持久化 → `save-system.md`
   - 箱子/交互/ChestController/BoxPanel/锁/钥匙 → `chest-interaction.md`

2. **长期防护**: 在 `maintenance-guidelines.md` 中添加检查项：
   > 每次新增/重命名/删除 steering 规则文件时，必须同步更新：
   > - `smart-assistant.kiro.hook` 的领域映射
   > - `README.md` 的文件索引表
   > - `maintenance-guidelines.md` 的关键词章节

### 6.3 优先级

P1 — 不修复则新规则文件形同虚设（永远不会被自动加载）。

---

## 七、问题 #6：maintenance-guidelines.md 体积膨胀观察

### 7.1 问题描述

`maintenance-guidelines.md` 作为 manual 加载文件，当前体积尚可。但随着规则文件增多、检查项增加，有膨胀风险。

### 7.2 观察策略

- **当前状态**: 无需操作，体积在合理范围内
- **触发阈值**: 当文件超过 200 行时，考虑拆分为：
  - `maintenance-guidelines.md`（核心检查清单）
  - `maintenance-details.md`（详细说明和示例）
- **监控方式**: 每次修改 maintenance-guidelines.md 时关注行数

### 7.3 优先级

P2 — 预防性观察，当前无需行动。

---

## 八、问题 #7：archive/farming.md 内容过时

### 8.1 问题描述

`archive/farming.md` 记录的是农田系统早期设计。但从 9.0.2 到 10.0.2，农田系统经历了大量开发：
- 9.0.2 放置农田
- 9.0.3 预览农田
- 9.0.4 智能交互升级
- 9.0.5 智能交互 bug 修复
- 10.0.1 农作物设计与完善
- 10.0.2 农 BUG 农 SO

当前 farming.md 的内容与实际代码严重脱节。

### 8.2 更新方案

**方案**: 基于最新代码和 memory 记录，更新 farming.md 的核心章节：

1. **农田放置系统**: 更新为 PlacementManager + FarmTileManager + PlacementGridCalculator 的实际架构
2. **农田预览系统**: 补充 FarmToolPreview 的实际实现
3. **交互系统**: 补充 PlacementNavigator + PlayerAutoNavigator 的智能交互
4. **作物系统**: 补充 CropController + CropData + SeedData 的实际数据结构
5. **已知问题**: 从各工作区 memory 中汇总未修复的问题

**注意**: archive 文件是参考性质，不需要像 P0 规则那样精确。重点是让新 AI 读到时不会被过时信息误导。

### 8.3 优先级

P1 — 过时的参考文档比没有文档更危险（会误导决策）。

---

## 九、问题 #8：rules.md 沟通规范整合质量审核

### 9.1 审核结论

Phase 4.1 会话 12 中整合的沟通规范质量**良好**，覆盖了核心需求：

| 整合内容 | 完整度 | 评价 |
|---------|--------|------|
| 核心原则（直接执行、简洁明了） | ✅ 完整 | 准确反映用户偏好 |
| 文档语言规范（中文强制） | ✅ 完整 | 覆盖 specs/对话/术语 |
| 与中文用户对话规范 | ✅ 完整 | 4 条具体规则 |
| 方案讨论流程 | ✅ 完整 | 5 步流程 |
| 何时展示代码 | ✅ 完整 | 4 种触发条件 |
| 技术决策 | ✅ 完整 | 稳妥方案 + 错误处理 |
| 一条龙模式 | ✅ 触发入口 | 完整流程在 communication.md |

### 9.2 改进建议

无重大改进需求。唯一的小优化：
- "理解用户意图后直接执行，不要反复确认"这条规则非常关键，当前位置在核心原则第二条，足够醒目
- 如果未来发现 AI 仍然反复确认，可以考虑将其提升到"最高优先级规则"章节

---

## 十、执行计划

| 序号 | 操作 | 涉及文件 | 优先级 |
|------|------|---------|--------|
| 1 | 创建存档系统规则文件 | `save-system.md`（新建） | P0 |
| 2 | 创建箱子/交互系统规则文件 | `chest-interaction.md`（新建） | P0 |
| 3 | 修复 README.md 索引 | `README.md`（修改） | P0 |
| 4 | 标注 communication.md 冗余 | `communication.md`（修改） | P1 |
| 5 | 更新 Hook 映射 + 维护指南 | `smart-assistant.kiro.hook` + `maintenance-guidelines.md`（修改） | P1 |
| 6 | 更新 farming.md | `archive/farming.md`（修改） | P1 |
| 7 | 观察 maintenance-guidelines.md 体积 | 无需操作 | P2 |
| 8 | rules.md 沟通规范 | 无需操作（已通过审核） | — |

**预计总工作量**: 操作 1-6 约需 2-3 个会话完成。建议按优先级顺序执行：先 P0（1-3），再 P1（4-6）。

---

## 十一、附录：fsAppend 防护规则（已实施）

**问题**: 会话 14 中 fsAppend 漏掉 text 参数，导致连续 7 次 aborted 错误形成死循环。

**已实施的修复**（写入 rules.md "文档写入规范"章节）:
- fsAppend 必须提供 text 参数——调用前先确认 text 内容非空
- 连续失败 2 次 → 立即停止重试，改用 fsWrite 重写整个文件
- 禁止对同一文件连续调用 3 次以上失败的写入操作
- 失败后的恢复策略：readFile → 确认断点 → fsWrite 完整内容或 fsAppend 确认 text 非空后重试

**未采用 preToolUse Hook 的原因**: 会话 8 讨论过，中间 Hook 每次 write 触发，一次对话 5-20 次额外消耗 2,000-8,000 tokens，成本不划算。规则约束是零成本方案。


---

## 十二、全面了解后的补充发现

基于全面阅读 SO设计系统（10个会话）、箱子系统（22个会话）、农田系统（9.0.1-10.0.2 全部迭代）后，补充以下发现：

### 12.1 农作物存档集成待验证（影响问题 #1）

10.0.1 农作物设计与完善中，`CropSaveData` 已扩展（含 cropDataId、growthStage、currentGrowthTime、isWatered、isWithered 等字段），但：
- CropController 的 `Save()`/`Load()` 实现尚未完整验证
- 农作物的动态重建流程（DynamicObjectFactory 中是否有 Crop 分支）未确认
- 这意味着 `save-system.md` 的"动态对象重建流程"章节需要预留农作物扩展位

**建议**: 在 save-system.md 的动态对象重建章节中，标注"农作物重建流程待 10.0.x 完成后补充"。

### 12.2 SO 字段优化重构搁置中（不影响存档，影响数据一致性）

SO设计系统的 `0_字段优化重构` 子工作区处于搁置状态：
- equipmentType/consumableType 计划从基类 ItemData 移到子类
- 这不影响存档系统（存档只关心 GUID 和状态数据，不关心 SO 字段结构）
- 但影响 `so-design.md` 和 `items.md` 的准确性

**建议**: 无需在 Phase 5.0 中处理，但在 so-design.md 中标注"字段优化重构搁置中"。

### 12.3 箱子系统已知未修复问题（影响问题 #2）

箱子系统有两个已知未修复问题：
1. Up 区域未接入 InventoryInteractionManager（拖拽不生效）
2. 拖拽与修饰键两套系统不互通

**建议**: 已在问题 #2 的"七、已知未修复问题"中记录，无需额外处理。

### 12.4 10.0.2 重构锐评审核中

农田系统 10.0.2 的重构锐评处于"路径C（重大异议）"状态，建议方案A务实路线。这不影响 Phase 5.0 的规则文件创建，但意味着 `archive/farming.md` 的更新（问题 #7）需要等 10.0.2 方向确定后再执行，避免二次修改。

**建议**: 问题 #7 的执行时机调整为"10.0.2 方向确定后"。

### 12.5 执行计划调整

| 原序号 | 操作 | 调整 |
|--------|------|------|
| 1 | 创建 save-system.md | 无调整，预留农作物扩展位 |
| 2 | 创建 chest-interaction.md | 无调整 |
| 3 | 修复 README.md 索引 | 无调整 |
| 4 | 标注 communication.md 冗余 | 无调整 |
| 5 | 更新 Hook 映射 | 无调整，本次已同步执行 |
| 6 | 更新 farming.md | ⚠️ 推迟到 10.0.2 方向确定后 |
| 7 | 观察 maintenance 体积 | 无调整 |
| 8 | rules.md 沟通规范 | 无调整 |
