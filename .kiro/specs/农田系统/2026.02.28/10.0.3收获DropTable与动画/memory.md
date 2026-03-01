# 10.0.3 收获DropTable与动画 - 开发记忆

## 模块概述
将作物收获机制从直接给予物品改为 DropTable 掉落到地面（与树木统一），并集成 Collect 动画交互。

## 当前状态
- **完成度**: 0%
- **最后更新**: 2026-02-14
- **状态**: 待开始

## 待做内容（从 10.0.2 迭代需求分析中提炼）

### 核心任务
1. **收获改用 DropTable** — 成熟作物收获时通过 DropTable 掉落物品到地面，与 TreeController 的 SpawnDrops 模式统一
2. **收获交互集成 Collect 动画** — 右键点击成熟作物 → 导航到位 → 播放 Collect 动画（AnimState.Collect=4）→ 第4~5帧触发收获逻辑
3. **SeedData 字段精简** — 移除 growthDays / harvestCropID / harvestAmountRange（这些信息改由 DropTable 和 CropStageConfig 承载）
4. **枯萎收获逻辑调整** — 枯萎成熟作物的收获产出调整（可能使用不同的 DropTable 或固定产出枯萎作物）
5. **品质系统适配** — 品质影响 DropTable 产出或独立处理

### 参考实现
- TreeController 的 `SpawnDrops()` 方法
- TreeController 的 `DropTable` 配置
- GameInputManager 中的交互触发机制
- AnimState.Collect 动画状态

### 前置依赖
- 10.0.2 已完成：CropStageConfig + daysToNextStage 累加模式 + 固定4阶段

## 会话记录

（暂无）

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 从 10.0.2 拆分到独立工作区 | 收获机制改动较大，与生长阶段改动解耦 | 2026-02-14 |

## 涉及的代码文件

### 预计修改文件
| 文件 | 说明 |
|------|------|
| `CropController.cs` | 收获逻辑改用 DropTable |
| `SeedData.cs` | 移除 growthDays / harvestCropID / harvestAmountRange |
| `GameInputManager.cs` | 集成 Collect 动画交互 |
| `CropManager.cs` | TryHarvest 适配 DropTable |

### 参考文件
| 文件 | 关系 |
|------|------|
| `TreeController.cs` | DropTable + SpawnDrops 参考实现 |
| `CropStageConfig.cs` | 10.0.2 新增的阶段配置 |


### 会话 1 - 2026-02-14（需求文档创建）

**用户需求**:
> 继续迭代（Keep Iterating），从 10.0.2 进入 10.0.3 工作区

**完成任务**:
1. 读取 10.0.3 memory.md、农田系统主 memory.md、farming.md 规范确认当前状态
2. 读取 TreeController 的 SpawnDrops 实现（发现是简化模式：dropItemData + stageDropAmounts，非复杂 DropTable SO）
3. 读取 CropController 的 Harvest/HarvestMature/HarvestWitheredMature/DropItemToWorld/DetermineHarvestQuality 方法
4. 读取 SeedData 完整字段
5. 读取 WorldSpawnService 的 SpawnMultiple 接口
6. 读取 AnimState 枚举（Collect = 4）和 GameInputManager 的 TryInteract 方法
7. 创建 `requirements.md` — 5 个用户故事、5 个边界情况、3 个非功能需求

**关键代码发现**:
- TreeController 掉落是简化模式：`dropItemData`（单个 ItemData SO）+ `stageDropAmounts[]`，不是复杂 DropTable SO
- CropController 当前收获是"先加背包，满了才掉地面"
- `TryInteract` 直接调用 `OnInteract`，没有播放 Collect 动画

**关键设计决策**:
- 学习 TreeController 简化模式：`dropItemData` + `dropAmount`，不用复杂 DropTable SO
- 枯萎作物用独立 `witheredDropItemData` 字段
- 品质由 `dropQuality` 字段控制，移除随机骰子
- SeedData 旧字段标记 `[Obsolete]` 而非删除，保证存档兼容

**修改文件**:
- `requirements.md` - 新建，10.0.3 需求文档

**下一步**:
- [ ] 用户审核 requirements.md
- [ ] 创建 design.md
- [ ] 创建 tasks.md


### 会话 2 - 2026-02-14（播种 bug 全面排查）

**用户需求**:
> 手持种子袋右键无法播种，控制台无任何输出，无物体生成。预览框正常显示。怀疑是 9.0.5 遗留问题。

**完成任务**:
1. 读取 9.0.5 tasks.md — 确认 18 个任务全部完成，但标注"待验收"
2. 读取农田系统主 memory.md — 梳理 9.0.1→9.0.5→10.0.1→10.0.2 完整演进历程
3. 全面追踪播种链路：HandleUseCurrentTool → TryPlantSeed → ExecutePlantSeed → CropManager.CreateCrop
4. 读取 FarmToolPreview.UpdateSeedPreview — 确认预览验证逻辑正常
5. 读取 SeedBagHelper — 发现种子袋未初始化时 ConsumeSeed 会静默返回 -1
6. 读取 CropManager — 发现使用全局 cropPrefab 而非 seedData.cropPrefab

**发现的 3 个可疑问题**:
1. **种子袋动态属性未初始化**（最可能根因）— SeedBagHelper.ConsumeSeed 在未初始化时 GetPropertyInt(KEY_REMAINING) 返回 0，减 1 变 -1，触发 `remaining < 0` 静默失败
2. **CropManager.cropPrefab 可能未配置** — CropManager 使用自己的全局 cropPrefab 字段，而非 seedData.cropPrefab，如果 Inspector 未拖入则 CreateCrop 返回 null
3. **showDebugInfo 默认 false** — 所有失败日志被 `if (showDebugInfo)` 包裹，导致控制台无输出

**关于 9.0.5 遗留问题**:
- 9.0.5 代码完成但未经用户游戏内验收
- 播种核心逻辑在 9.0.4 之前就有，9.0.5 只加了锁定机制
- 问题更可能出在 10.0.1 新增的 SeedBagHelper 保质期系统

**待用户确认**:
- [ ] CropManager Inspector 的 cropPrefab 是否配置
- [ ] GameInputManager 的 showDebugInfo 是否勾选（请勾选后重试）
- [ ] 种子获取方式（商店购买 vs 手动添加到背包）

**修改文件**: 无（纯分析，未修改代码）


### 会话 2（续） - 2026-02-14（播种全面配置指南）

**用户需求**:
> 场景根本没有 CropManager，给我全面配置指南（直接在对话中输出，不用新建文件）

**完成任务**:
1. 继承恢复：确认上一轮排查的 3 个根因（场景缺 CropManager、种子袋未初始化、CreateCrop 用全局 cropPrefab）
2. 重新读取关键代码文件交叉验证：CropManager.cs、GameInputManager.cs（ExecutePlantSeed）、SeedBagHelper.cs、InventoryBootstrap.cs、SeedData.cs、FarmToolPreview.cs、FarmTileManager.cs
3. 在对话中输出完整播种配置指南，覆盖：
   - 问题 1：场景添加 CropManager 组件（配置 cropPrefab、defaultCropsContainer、showDebugInfo）
   - 问题 2：种子袋动态属性未初始化（InventoryBootstrap 注入种子不调用 InitializeSeedBag，ConsumeSeed 返回 -1 静默失败）— 推荐方案 A：ExecutePlantSeed 中加自动初始化
   - 问题 3：CropManager.CreateCrop 应优先使用 seedData.cropPrefab 而非全局 cropPrefab
   - 附加：GameInputManager.showDebugInfo 建议勾选
4. 列出 3 处需要修复的代码及对应文件

**关键结论**:
- 3 处代码修复属于 bug 修复性质，与 10.0.3 收获 DropTable 功能独立
- 用户确认场景中没有 CropManager（根因 #1）
- 种子通过 InventoryBootstrap 注入（根因 #2 的触发条件）
- FarmTileManager 和 FarmToolPreview 正常工作（预览绿框能显示）

**待用户决策**:
- [ ] 先修 3 处 bug 让播种跑通，还是放到 10.0.3 tasks 里一起做

**修改文件**: 无（纯分析+配置指南输出，未修改代码）


### 会话 3 - 2026-02-14（10.0.2.1 全面调查报告）

**用户需求**:
> CropManager 的设计方向错误，应该像树木系统一样不需要集中式 Manager。全盘了解一下，检查当前所有与农田相关的代码和系统然后进行一个全面的调查写一个报告。

**完成任务**:
1. 读取所有农田代码文件：CropController.cs、CropManager.cs、CropInstance.cs、CropInstanceData.cs、CropStageConfig.cs、FarmTileData.cs、FarmVisualManager.cs、FarmlandBorderManager.cs、FarmTileManager.cs、FarmToolPreview.cs、SeedBagHelper.cs、SoilMoistureState.cs
2. 读取关联外部文件：TreeController.cs（完整字段+关键方法签名）、SeedData.cs、CropData.cs、GameInputManager.cs（播种部分）、InventoryBootstrap.cs
3. 读取所有工作区需求文档：9.0.1、9.0.2、9.0.3、9.0.4、9.0.5、10.0.1、10.0.2、10.0.3、10.0.2 迭代需求 001
4. 在 `.kiro/specs/农田系统/10.0.2.1纠正/` 创建全面调查报告，包含三大区：
   - 第一区：代码统计与联动分析（12 个文件清单 + 联动关系图 + 3 个关键设计矛盾）
   - 第二区：工作区进程与设计方向对比（9 个工作区时间线 + 各自设计思路 + 共同方向 + 区别）
   - 第三区：全面总结与独立思考（健康度评估 + CropManager vs TreeController 对比 + 纠正方向建议 + 风险评估）

**关键发现**:
- CropManager 的 3 个核心矛盾：全局 cropPrefab vs SeedData.cropPrefab、角色定位过时、收获逻辑双重路径
- CropController 已具备完全自治能力（状态机+事件订阅+存档+交互），CropManager 是多余的
- CropManager.TryHarvest 可能已经是死路径（CropController.Harvest 通过 IInteractable 独立工作）
- 纠正方向：P0 修复播种链路（绕过 CropManager），P1 废弃 CropManager，P2 继续 10.0.3

**修改文件**:
- 全面调查报告.md - 新建，10.0.2.1 纠正工作区全面调查报告
- memory.md - 追加会话 3 记录


### 会话 3（续） - 2026-02-14（10.X 纠正一条龙执行，续接自会话 3）

**用户需求**:
> 10.0.2.X 和 10.0.3 一起并发工作，先全面废弃 CropManager，一条龙完成。

**完成任务**:
10.X 纠正已在 `.kiro/specs/农田系统/10.0.2.1纠正/` 工作区一条龙完成，覆盖了 10.0.3 requirements 中的大部分内容：
- US-1 成熟作物收获改用掉落模式 ✅（CropController.dropItemData + SpawnMultiple）
- US-2 枯萎成熟作物收获调整 ✅（witheredDropItemData）
- US-4 SeedData 字段精简 ✅（3 个字段标记 Obsolete）
- US-5 品质系统适配 ✅（dropQuality 字段替代随机骰子）
- US-3 Collect 动画集成 ❌（预留后续迭代）

**10.0.3 剩余待做**:
- [ ] Collect 动画集成（右键点击成熟作物 → 导航 → 播放 Collect 动画 → 关键帧触发收获）
- [ ] CropController Prefab Inspector 配置 dropItemData/witheredDropItemData（需要用户在 Unity 编辑器中操作）

**修改文件**: 见 10.0.2.1 纠正工作区 memory.md


### 会话 3（续） - 2026-02-14（requirements.md 覆盖标注）

**用户需求**:
> 给 10.0.3 requirements.md 补充标注哪些 US 已被 10.X 纠正覆盖

**完成任务**:
1. 在 requirements.md 用户故事章节顶部添加覆盖说明
2. US-1 标注 ✅ 已由 10.0.2.1 纠正完成
3. US-2 标注 ✅ 已由 10.0.2.1 纠正完成
4. US-3 标注 ⏳ 待实现（唯一剩余）
5. US-4 标注 ✅ 已由 10.0.2.1 纠正完成
6. US-5 标注 ✅ 已由 10.0.2.1 纠正完成

**修改文件**:
- `requirements.md` - 补充 US 覆盖标注
- `memory.md` - 追加会话记录


### 会话 4 - 2026-02-15（CropData SO 字段精简 — 废弃反向引用消除三向绑定）

**用户需求**:
> CropData 的 seedID、witheredCropID 和 SeedData 旧 harvestCropID 形成三向绑定，每创建新作物要在三个 SO + 一个 Prefab 上分别填写互相指向的 ID/引用。将 CropData.seedID、CropData.witheredCropID、WitheredCropData.seedID 标记 [Obsolete]，数据流改为单向。

**完成任务**:
1. CropData.cs — `seedID` 和 `witheredCropID` 添加 `[System.Obsolete]` 标记，移除 OnValidate 中对这两个字段的范围验证
2. WitheredCropData.cs — `seedID` 添加 `[System.Obsolete]` 标记，移除 OnValidate 中对应验证
3. Tool_BatchItemSOGenerator.cs — 用 `#pragma warning disable/restore 0618` 包裹 CreateCropData 和 CreateWitheredCropData 中赋值 obsolete 字段的代码（保留批量工具功能）
4. getDiagnostics 验证三个文件编译通过，无错误无警告
5. farming.md — 更新 CropData 章节说明废弃字段，新增 WitheredCropData 章节

**关键设计决策**:
- 旧字段标记 `[Obsolete]` 而非删除，保证存档兼容和批量工具向后兼容
- 批量工具用 `#pragma warning disable 0618` 消除编译警告，保留赋值功能
- 数据流变单向：`SeedData → cropPrefab → CropController → dropItemData(CropData)`

**修改文件**:
- CropData.cs - seedID/witheredCropID 标记 Obsolete，移除 OnValidate 验证
- WitheredCropData.cs - seedID 标记 Obsolete，移除 OnValidate 验证
- Tool_BatchItemSOGenerator.cs - pragma warning 包裹 obsolete 字段赋值
- farming.md - 更新 CropData 章节，新增 WitheredCropData 章节
- memory.md - 追加会话 4 记录


### 会话 4（续） - 2026-02-15（OnValidate ID 范围修正，对齐 ID 分配规范）

**用户需求**:
> Unity 控制台报错：枯萎作物（胡萝卜1156、甜菜1155 等）ID 不在 1200-1299 范围。实际 ID 分配规范是 1150-1199。

**完成任务**:
1. 确认 ID 分配规范：CropData 1100-1149，WitheredCropData 1150-1199（共享 11XX 段）
2. WitheredCropData.cs — OnValidate 中 `itemID` 验证范围从 1200-1299 改为 1150-1199，`normalCropID` 验证范围从 1100-1199 精确为 1100-1149
3. CropData.cs — OnValidate 中 `itemID` 验证范围从 1100-1199 精确为 1100-1149
4. getDiagnostics 验证编译通过

**根因**:
- WitheredCropData 的 OnValidate 验证范围是旧的 1200-1299，没跟着 ID 规范更新（规范在 2026-02-15 将枯萎作物从 1200 改为 1150）
- CropData 的验证范围 1100-1199 也不精确，1150-1199 已分配给枯萎作物

**附注**:
- 大蒜种子缺少 cropPrefab 是 Inspector 配置问题，需用户手动拖入对应作物预制体

**修改文件**:
- WitheredCropData.cs - OnValidate ID 验证范围改为 1150-1199
- CropData.cs - OnValidate ID 验证范围精确为 1100-1149
- memory.md - 追加会话记录


### 会话 4（续） - 2026-02-15（三个种植类 Data SO 全面审计 — Inspector 显示与逻辑统一）

**用户需求**:
> 对 SeedData、CropData、WitheredCropData 三个种植类 Data SO 进行全面审计，`[Obsolete]` 字段仍在 Inspector 中显示，需要全面处理。

**完成任务**:
1. 全面统计三个 SO 的专属字段状态（活跃/废弃/应显示/应隐藏）
2. ItemDataEditor.cs — `DrawRemainingProperties()` 新增反射过滤 `[System.Obsolete]` 字段逻辑，自动隐藏废弃字段
3. 新增 `_obsoleteFields` 缓存（HashSet），OnEnable 时重置，首次 DrawRemainingProperties 时构建
4. 新增 `using System.Linq` 和 `using System.Reflection` 引用
5. SeedData.cs — OnValidate 中 `itemID` 验证范围从 1000-1999 精确为 1000-1099（对齐 ID 分配规范）
6. getDiagnostics 验证编译通过
7. farming.md — 新增"十一、Inspector 显示规范"章节，更新 ID 范围描述

**方案选择**:
- 选择在 `DrawRemainingProperties()` 中用反射过滤（集中管理），而非给每个字段加 `[HideInInspector]`（分散管理）
- 优势：新增 Obsolete 字段时自动隐藏，无需额外操作

**隐藏的废弃字段清单**:
- SeedData：`growthDays`、`harvestCropID`、`harvestAmountRange`
- CropData：`seedID`、`witheredCropID`
- WitheredCropData：`seedID`

**附加发现（未修改，记录备忘）**:
- SeedData.GetTooltipText() 仍引用 obsolete 的 `growthDays` 显示生长周期，因为 CropStageConfig 在 Prefab 上而非 SeedData 上，运行时无法直接获取。后续迭代可考虑在 SeedData 上加一个 `displayGrowthDays` 字段或从 cropPrefab 动态读取
- 批量工具 UI 中 SeedData 的 `setSeedGrowth`/`setSeedHarvest` 等选项仍绘制废弃字段的配置，但这是工具内部 UI 不受 ItemDataEditor 影响，暂不处理

**修改文件**:
- ItemDataEditor.cs - DrawRemainingProperties 新增 Obsolete 字段过滤
- SeedData.cs - OnValidate ID 验证范围精确为 1000-1099
- farming.md - 新增 Inspector 显示规范章节
- memory.md - 追加会话记录


### 会话 4（续） - 2026-02-15（三个种植类 Data SO 字段精简执行）

**用户需求**:
> 继承自会话 4 审计，执行用户已确认的字段删除/废弃决策。

**完成任务**:
1. SeedData.cs — 删除 `growthDays`、`harvestCropID`、`harvestAmountRange` 三个字段声明（含所有 attribute）
2. SeedData.cs — 移除 GetTooltipText 中 `growthDays` 引用和 pragma warning
3. SeedData.cs — 移除 OnValidate 中 `harvestAmountRange` 兼容验证代码块
4. CropData.cs — `harvestExp` 添加 `[System.Obsolete("经验值统一归 SeedData.harvestingExp，保留仅为存档兼容")]`
5. WitheredCropData.cs — 删除 `normalCropID` 字段声明和 OnValidate 中的 normalCropID 验证代码
6. Tool_BatchItemSOGenerator.cs — 清理已删除字段对应的：
   - 字段声明：`setSeedGrowth`/`seedGrowthDays`/`setSeedHarvest`/`seedHarvestCropID`/`setCropExp`/`cropHarvestExp`/`setWitheredNormalCropID`/`witheredNormalCropID`
   - DrawSeedSettings：移除生长天数和收获作物 ID 绘制
   - DrawCropSettings：移除收获经验绘制
   - DrawWitheredCropSettings：移除对应正常作物 ID 绘制
   - CreateSeedData：移除 growthDays/harvestCropID 赋值
   - CreateCropData：移除 harvestExp 赋值
   - CreateWitheredCropData：移除 normalCropID 赋值
   - LoadSettings/SaveSettings：移除对应 EditorPrefs 读写
7. CropManager.cs — TryHarvest 中 seedData.harvestCropID/harvestAmountRange 引用改为注释说明（整个类已 Obsolete）
8. getDiagnostics 验证全部 5 个文件编译通过，无错误
9. farming.md — 更新 SeedData/CropData/WitheredCropData/Inspector 显示规范章节，反映字段从 Obsolete 改为直接删除

**关键决策**:
- SeedData 三个字段直接删除（非 Obsolete），因为用户明确说"直接废除"
- CropData.harvestExp 标记 Obsolete（经验值统一归 SeedData.harvestingExp）
- WitheredCropData.normalCropID 直接删除（无任何引用）
- CropManager 整体已 Obsolete，TryHarvest 中改为固定值 + 注释说明

**修改文件**:
- SeedData.cs - 删除 growthDays/harvestCropID/harvestAmountRange，清理 GetTooltipText 和 OnValidate
- CropData.cs - harvestExp 标记 Obsolete
- WitheredCropData.cs - 删除 normalCropID 及 OnValidate 验证
- Tool_BatchItemSOGenerator.cs - 清理 8 个字段声明 + UI 绘制 + 创建赋值 + 设置读写
- CropManager.cs - TryHarvest 中已删除字段引用改为注释
- farming.md - 更新三个 SO 章节和 Inspector 显示规范
- memory.md - 追加会话记录


### 会话 4（续） - 2026-02-15（全面审查 — 已删除字段残留引用检查）

**用户需求**:
> 全面检查回顾 TASK 4（三个种植类 Data SO 全面审计与字段精简）是否有遗漏或做错的方向与内容。

**完成任务**:
1. 全代码库搜索已删除字段残留引用：
   - `growthDays` — 仅 CropInstance.cs 注释中提及（该类整体已 Obsolete，不影响编译运行）
   - `harvestCropID` / `harvestAmountRange` — 仅 SeedData.cs 和 CropManager.cs 注释中提及（说明性注释，无实际引用）
   - `normalCropID` — 全代码库零匹配，清理彻底
2. 批量工具残留检查：8 个相关字段声明/UI 绘制/创建赋值/EditorPrefs 读写全部清除，零残留
3. DynamicObjectFactory.TryReconstructCrop 检查 — 存档重建只用 `seedData.cropPrefab`，不依赖任何已删除字段
4. CropSystemTests 检查 — 测试用纯逻辑模型，不引用任何已删除字段
5. getDiagnostics 验证全部 5 个修改文件编译通过，零错误零警告
6. Memory 交叉验证 — memory 记录与继承摘要一致，无遗漏

**审查结论**: 无遗漏，方向正确，TASK 4 可标记完成。

**修改文件**:
- memory.md - 追加审查会话记录


### 会话 4（续） - 2026-02-15（代办补充 — 用户原始需求决策点完善）

**用户需求**:
> 检查代办是否完整记录了用户原始需求中的关键决策点（needsWatering/plantingExp/harvestingExp 遗漏、reHarvestDays/maxHarvestCount 暂保留、经验值以 SeedData 为主、树木系统一并审视等）。

**完成任务**:
1. 读取 TD_000_农田系统.md 和 TD_10.0.2农BUG农SO.md，对照用户原始需求文本
2. 发现 #5 代办描述不够完整，缺少具体字段遗漏点和设计决策
3. 补充 #5 代办详细内容：needsWatering/plantingExp/harvestingExp Controller 遗漏、reHarvestDays/maxHarvestCount 暂保留闲置、经验值统一归 SeedData、Tooltip 生长周期数据源待解决

**修改文件**:
- TD_000_农田系统.md - 补充 #5 代办的详细决策点
- memory.md - 追加会话记录


### 会话 4（续） - 2026-02-15（子代办创建与主代办索引补全）

**用户需求**:
> 指出代办只更新了主代办，没有创建对应的子代办文件，且主代办索引表缺少 10.0.3。

**完成任务**:
1. 新建 `TD_10.0.3收获DropTable与动画.md` 子代办，包含 3 条待办：Controller 与 SO 数据流完善、Collect 动画集成、CropController Prefab Inspector 配置
2. 更新 `TD_000_农田系统.md` 子代办索引表，加入 10.0.3 条目

**修改文件**:
- TD_10.0.3收获DropTable与动画.md - 新建，10.0.3 子代办
- TD_000_农田系统.md - 子代办索引表加入 10.0.3
- memory.md - 追加会话记录
