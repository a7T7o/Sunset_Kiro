# 10.1.1 补丁002 - 开发记忆

## 模块概述

农田系统交互体验优化：长按失效修复、种子连续种植、收获改左键、1+8 预览刷新修复。

## 当前状态

- **完成度**: 10%（分析文档完成，待用户确认 Q1-Q4）
- **最后更新**: 2026-02-19
- **状态**: 🚧 进行中
- **当前焦点**: 全面分析文档已完成，等待用户回复 Q1-Q4

---

## 会话记录

### 会话 1 - 2026-02-19（全面分析与修复方案）

**来源**：用户验收 10.1.1补丁001 二次修复后的新反馈

**用户需求**：
> 1. 长按在无效位置时失效（锄头+浇水）
> 2. 种子缺少连续输入缓存 + 收获改左键 + 收获连续操作
> 3. 远距离点击切换时 1+8 预览仍不刷新

**完成任务**：
1. 全面勘察关键代码文件：
   - PlayerInteraction.cs：OnActionComplete 完整长按/松开分支
   - GameInputManager.cs：HandleUseCurrentTool、CacheFarmInput、ProcessFarmInputAt、TryTillSoil、TryPlantSeed、ExecutePlantSeed、ConsumePendingFarmInput、ForceUpdatePreviewToPosition、TryHandleFarmingTool、HandleRightClickAutoNav、StartFarmingNavigation
   - FarmToolPreview.cs：LockPosition、UnlockPosition、UpdateHoePreview
   - CropController.cs：OnInteract、Harvest（IInteractable 实现）
2. 精确定位三个问题的根因：
   - P1：OnActionComplete 长按分支 → ProcessFarmInputAt → TryTillSoil → IsValid()==false 直接返回，长按链断裂
   - P2：种子只响应 GetMouseButtonDown，无持续检测；收获在右键链路
   - P3：TryTillSoil 直接 LockPosition 只更新光标不刷新 GhostTilemap，锁定后 UpdateHoePreview 被跳过
3. 创建完整分析文档（6 大部分）

**新建文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `补丁002全面分析与修复方案.md` | 新建 | 用户原始反馈 + 3 个问题根因分析 + 修复方案 + Q1-Q4 |
| `memory.md` | 新建 | 子工作区记忆 |

**涉及代码文件（分析，未修改）**：
| 文件 | 分析内容 |
|------|---------|
| GameInputManager.cs | HandleUseCurrentTool、长按分支、缓存机制、种子种植、收获链路 |
| PlayerInteraction.cs | OnActionComplete 长按/松开分支 |
| FarmToolPreview.cs | LockPosition/UpdateHoePreview 的 _isLocked 守卫 |
| CropController.cs | IInteractable 收获实现 |

**待用户确认**：
- Q1：种子连续种植触发方式（长按自动 vs 快速连续点击）
- Q2：收获改左键后右键行为（完全移除 vs 双键都能）
- Q3：手持农田工具时左键点击成熟作物的优先级
- Q4：P3 修复方案选择（方案 A/B/C）

**状态**: 分析文档 ✅ 完成，待用户审核 Q1-Q4 后开始实现

### 会话 1（续2）- 2026-02-19（用户 Q1-Q4 确认 + 新增需求，被压缩中断）

**来源**：用户回复 Q1-Q4 确认 + 补充新需求

**用户确认结果**：
- Q1：不要长按，改为连续点击 + 队列缓存（FIFO，先进先出）
- Q2：移除右键收获，改左键收获，收获有动画，也用队列缓存连续点击
- Q3：成熟作物无论手持什么都左键收获（收获优先级最高）
- Q4：用户很生气 — LockPosition 时应该只更新锁定位置的 1+8 渲染，取消锁定后再跟随鼠标（用户之前就提过）

**用户新增需求**：
1. 作物显示对齐：底部中心对齐农田中心（学习树木/石头/箱子的加载方式，保证每个阶段底部中心对齐）
2. 收获动画：用户不确定是否已配置，要求查找 10.1.0 完成的内容给配置指南
3. 如果收获动画未配置导致没播放 → 暂时搁置收获逻辑修改

**AI 进度**：
1. ✅ 读取了 CropController.cs 的 Initialize、UpdateVisuals 方法
2. ✅ 读取了 TreeController.cs 的代码签名（发现 AlignSpriteBottom 方法）
3. ❌ 未完成：TreeController.AlignSpriteBottom 详细代码读取
4. ❌ 未完成：收获动画相关代码勘察（10.1.0 的 Collect 动画集成）
5. ❌ 未完成：更新补丁002分析文档（加入 Q1-Q4 确认结果 + 新增需求分析）
6. ❌ 未完成：作物对齐方案设计
7. ❌ 未完成：收获动画配置指南

**涉及代码文件（分析中，未修改）**：
| 文件 | 分析内容 |
|------|---------|
| CropController.cs | Initialize、UpdateVisuals — 作物显示逻辑 |
| TreeController.cs | 代码签名 — 发现 AlignSpriteBottom 方法（底部对齐） |

**状态**: 🚧 被压缩中断，需要继承恢复后继续勘察和文档更新


### 会话 1（续3）- 2026-02-19（继承恢复 + 代码勘察完成 + 三件套创建）

**来源**：继承恢复自会话1续2

**完成任务**：
1. ✅ 继承恢复：读取快照、memory、系统摘要，交叉验证无异常
2. ✅ 读取 TreeController.AlignSpriteBottom() 详细实现 — 确认底部对齐逻辑：`localPos.y = -spriteBounds.min.y`
3. ✅ 读取 CropController 完整代码：Awake、Initialize（3个重载）、UpdateVisuals、Harvest、HarvestMature、HarvestWitheredMature、字段定义
4. ✅ 读取 ExecutePlantSeed 完整实现 — 确认作物位置使用 `tilemaps.GetCellCenterWorld(cellPos)`，无底部对齐
5. ✅ 读取 FarmToolPreview.LockPosition/UpdateHoePreview — 确认 P3 根因：LockPosition 只 UpdateCursor 不刷新 GhostTilemap
6. ✅ 收获动画调查完成：
   - AnimState.Collect = 4 已定义
   - PlayerInteraction 中有测试代码（按 E 触发），但收获逻辑未集成 Collect 动画
   - 10.0.3 US-3 标记"⏳ 待实现"，farming.md 写"Collect 动画集成待后续迭代"
   - 结论：不是用户配置问题，是代码尚未实现 → 不触发搁置条件
7. ✅ 更新补丁002分析文档：追加第七~九章（Q1-Q4 确认结果 + P5/P6 新增需求 + 修正后问题清单）
8. ✅ 创建 design.md — 10 章节完整设计（P3/P1/P2a/P2b/P5 方案 + 操作队列详细设计 + 正确性属性）
9. ✅ 创建 tasks.md — 5 Phase、13 个任务

**新建/修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `补丁002全面分析与修复方案.md` | 追加 | 第七~九章：Q1-Q4 确认 + P5/P6 + 修正问题清单 |
| `design.md` | 新建 | 完整设计文档 |
| `tasks.md` | 新建 | 任务列表 |
| `memory.md` | 追加 | 本条记录 |

**涉及代码文件（分析，未修改）**：
| 文件 | 分析内容 |
|------|---------|
| TreeController.cs | AlignSpriteBottom 详细实现 |
| CropController.cs | 完整字段、Initialize、UpdateVisuals、Harvest |
| GameInputManager.cs | ExecutePlantSeed、HandleUseCurrentTool |
| FarmToolPreview.cs | LockPosition、UpdateHoePreview |
| PlayerAnimController.cs | AnimState 枚举（Collect = 4） |
| PlayerInteraction.cs | Collect 动画测试代码 |

**关键结论**：
- P6 收获动画：不是配置问题，是代码未实现 → 不搁置，在本补丁中集成
- P5 作物对齐：CropController 缺少 AlignSpriteBottom，需要新增
- P3 预览不刷新：LockPosition 只更新光标不刷新 GhostTilemap，需要在锁定时渲染

**状态**: 三件套完成，待用户审核后开始实现


### 会话 1（续4）- 2026-02-19（继承恢复 + 分析文档独立审核）

**来源**：继承恢复自会话1续3 + 用户要求独立审核分析文档

**用户需求**：
> 重新客观独立的分析并审核补丁002全面分析与修复方案.md，是否可实现、合理、符合交互习惯、是否存在漏洞

**完成任务**：
1. ✅ 继承恢复：交叉验证无异常
2. ✅ 重新读取所有关键代码文件进行独立验证：
   - GameInputManager.cs：HandleUseCurrentTool、TryTillSoil、TryPlantSeed、HandleRightClickAutoNav、CacheFarmInput、ConsumePendingFarmInput、ProcessFarmInputAt
   - FarmToolPreview.cs：LockPosition、UpdateHoePreview
   - PlayerInteraction.cs：OnActionComplete、IsToolAction
   - CropController.cs：UpdateVisuals、Harvest、CanInteract、OnInteract
3. ✅ 输出完整审核报告：
   - P1/P2/P3/P5/P6 根因分析全部验证准确
   - 交互习惯审核：连续点击队列合理、收获改左键合理、收获最高优先级合理
   - 发现 6 个潜在漏洞：
     1. CacheFarmInput 与队列的替换关系未明确（动画期间/执行保护期间的入口也需改为入队）
     2. `_isExecutingFarming` 标志与队列的交互需仔细处理
     3. OnActionComplete 中 Collect 动画回调需单独处理（IsToolAction 不含 Collect）
     4. WASD 清空队列与 HandleMovement/lockManager 的时序
     5. 打开面板不清空队列可能导致导航+执行与面板冲突
     6. 收获检测的多层级过滤问题

**涉及代码文件（分析，未修改）**：
| 文件 | 分析内容 |
|------|---------|
| GameInputManager.cs | HandleUseCurrentTool、TryTillSoil、TryPlantSeed、HandleRightClickAutoNav、CacheFarmInput、ConsumePendingFarmInput、ProcessFarmInputAt |
| FarmToolPreview.cs | LockPosition、UpdateHoePreview |
| PlayerInteraction.cs | OnActionComplete、IsToolAction |
| CropController.cs | UpdateVisuals、Harvest、CanInteract、OnInteract |

**关键结论**：
- 分析文档根因定位全部准确
- 第一~六章原始方案与第七~九章用户确认结果存在矛盾（文档演进自然结果，不影响实现）
- 漏洞 1（CacheFarmInput 替换）和漏洞 3（Collect 动画回调）是实现时最容易踩坑的地方

**状态**: 审核完成，待用户反馈后决定是否更新 design.md 补充漏洞处理方案


### 会话 1（续5）- 2026-02-19（用户纠正 + 全面更新002补丁，被压缩中断）

**来源**：用户纠正续4审核中的理解偏差 + 要求将所有漏洞和交互细节纳入002补丁

**用户核心指令**：
1. 纠正：Q1 问的是种子连续种植，不是锄头/浇水长按。用户回复是种子也改为连续点击+队列
2. 审核发现的 6 个漏洞 + 5 个交互细节全部纳入 002 补丁
3. 需要像 001 的交互矩阵一样做完整交互情况覆盖
4. 更新 design.md 和 tasks.md 达到"彻底完美"

**完成任务**：
1. ✅ 更新 `补丁002全面分析与修复方案.md` — 追加第十章（审核发现的漏洞与交互细节补充，含 10.1.1~10.1.5 交互细节 + 10.2.1~10.2.6 漏洞补充）
2. ✅ 重写 `design.md`（V2）— 已完成：
   - 第一章：设计概述（5 核心问题 + 6 漏洞补充 = 11 改进点）
   - 第二章：P3 修复方案
   - 第三章：FIFO 操作队列核心设计（数据结构、入队、出队、替换关系、防重复）
   - 第四章：完整交互矩阵（左键入队矩阵、队列执行期间点击矩阵、中断矩阵、预览状态矩阵、OnActionComplete 分支矩阵）
   - 第五章：P2b 收获改左键 + Collect 动画 + 队列（走 IInteractable 接口、Collect 专用分支、移除右键收获）
3. ❌ 未完成：design.md 剩余章节（P5 作物对齐、V4 WASD 中断、V5 面板暂停、正确性属性、涉及文件汇总）
4. ❌ 未完成：tasks.md 重写

**修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `补丁002全面分析与修复方案.md` | 追加 | 第十章：审核发现的漏洞与交互细节 |
| `design.md` | 重写（部分） | V2 版本，已完成第一~五章 |

**状态**: 🚧 被压缩中断，design.md 写到第五章，tasks.md 未开始重写


### 会话 1（续6）- 2026-02-19（继承恢复 + design.md V2 完成 + tasks.md V2 重写）

**来源**：继承恢复自会话1续5

**完成任务**：
1. ✅ 继承恢复：快照/memory/系统摘要交叉验证无异常
2. ✅ 完成 design.md V2 剩余章节：
   - 第六章：P2a 种子连续种植（连续点击+队列，种子用完边界处理）
   - 第七章：P5 作物显示底部中心对齐（AlignSpriteBottom，参考 TreeController）
   - 第八章：V4 WASD 中断优先级（HandleMovement 中 WASD 检测在 lockManager.IsLocked 之前）
   - 第九章：V5 面板暂停队列（_isQueuePaused 标志，暂停/恢复机制）
   - 第十章：V6 收获检测多层级过滤（crop.layerIndex == 玩家 layerIndex）
   - 第十一章：正确性属性（CP-1~CP-19，覆盖队列/收获/工具/中断/视觉/OnActionComplete）
   - 第十二章：测试框架（编译验证+逻辑审查+运行时验证）
   - 第十三章：涉及文件汇总（4个文件）
3. ✅ 重写 tasks.md V2：8 Phase、22 个任务，完整覆盖 11 个改进点

**修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `design.md` | 追加 | V2 第六~十三章（续5写到第五章，本轮补完） |
| `tasks.md` | 重写 | V1 → V2，8 Phase 22 任务 |
| `memory.md` | 追加 | 本条记录 |

**状态**: design.md V2 ✅ 全部完成（十三章），tasks.md V2 ✅ 全部完成（8 Phase 22 任务），待用户审核


### 会话 1（续7）- 2026-02-19（继承恢复 + design V3 + tasks V3 重写）

**来源**：继承恢复自会话1续6 + 用户要求彻底审视并重构 design 和 tasks

**用户核心指令**：
> 当前 design 和 tasks 是"照葫芦画瓢"，缺乏专业思考。需要把 002 的所有详细内容拆解开来进行全面调度和分配，做到相关内容在一个版块。先彻底审视并完善 design，然后重构 tasks。

**完成任务**：
1. ✅ 继承恢复：交叉验证无异常（memory 续6 = design V2 + tasks V2 完成，实际文件确认一致）
2. ✅ 重新读取所有关键代码文件（当前会话实际 readCode）：
   - GameInputManager.cs：HandleUseCurrentTool、CacheFarmInput、ConsumePendingFarmInput、ProcessFarmInputAt、TryTillSoil、TryWaterTile、TryPlantSeed、HandleMovement、HandleRightClickAutoNav、StartFarmingNavigation、CancelFarmingNavigation
   - PlayerInteraction.cs：OnActionComplete、IsToolAction
   - FarmToolPreview.cs：LockPosition、UnlockPosition、UpdateHoePreview
   - CropController.cs：UpdateVisuals、CanInteract、OnInteract、完整签名
3. ✅ 建立"变更点 → 文件×方法"映射表（4 个文件、9 个模块 A~I）
4. ✅ 重写 design.md V3（14 章，按修改目标组织）：
   - 第一章：全局架构概览（队列生命周期、替换关系、废弃清单）
   - 第二章：模块 A — 队列基础设施（数据结构、字段、_isExecutingFarming 语义变更）
   - 第三章：模块 B — HandleUseCurrentTool 全面改造（收获检测、工具入队、保护分支替换、完整伪代码）
   - 第四章：模块 C — 队列执行引擎（EnqueueAction、ProcessNextAction、ExecuteFarmAction、回调方法、ClearActionQueue）
   - 第五章：模块 D — HandleMovement WASD 中断
   - 第六章：模块 E — HandleRightClickAutoNav 作物过滤
   - 第七章：模块 F — 面板暂停/恢复 + 旧方法废弃
   - 第八章：模块 G — OnActionComplete 分支改造（Collect 专用分支、农田工具队列出队、通用工具不变）
   - 第九章：模块 H — LockPosition 1+8 渲染修复
   - 第十章：模块 I — CropController 作物底部对齐
   - 第十一章：完整交互矩阵（6 个矩阵表）
   - 第十二章：正确性属性与保证关系（CP-1~CP-19，每条标注保证模块）
   - 第十三章：测试框架
   - 第十四章：涉及文件汇总
5. ✅ 重写 tasks.md V3（7 Phase、20 个任务，每个任务自包含执行细节）：
   - Phase 1：独立模块（LockPosition 渲染 + 作物对齐）
   - Phase 2：队列基础设施（数据结构 + EnqueueAction + ProcessNextAction + ExecuteFarmAction + 回调方法）
   - Phase 3：入队入口改造（收获检测 + 工具/种子入队 + HandleUseCurrentTool 全面改造）
   - Phase 4：OnActionComplete 分支改造（Collect 专用分支 + 农田工具队列出队）
   - Phase 5：中断/过滤/暂停（WASD 中断 + 右键过滤 + 面板暂停 + ESC/快捷栏清空）
   - Phase 6：旧方法废弃与清理
   - Phase 7：集成验证（编译 + CP 审查 + 交互矩阵 + memory 更新）

**修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `design.md` | 重写 | V2 → V3，按修改目标重组，14 章 |
| `tasks.md` | 重写 | V2 → V3，7 Phase 20 任务，自包含执行细节 |
| `memory.md` | 追加 | 本条记录 |

**V3 与 V2 的核心区别**：
- V2 按"问题编号"线性排列（P3→P1→P2b→...），执行时需要在多个章节间跳来跳去
- V3 按"修改目标（文件×方法）"聚合，每个版块自包含（现状→改动→伪代码→属性），执行时只看对应版块
- tasks V3 每个任务包含完整的执行细节（文件、行号、具体改动、验收标准），不需要回翻 design

**状态**: design V3 ✅ + tasks V3 ✅ 完成，待用户审核


### 会话 1（续8）- 2026-02-19（继承恢复 + 主 memory 同步 + 用户问题回应）

**来源**：继承恢复自会话1续7

**完成任务**：
1. ✅ 继承恢复：读取快照（续5）、子 memory（续7）、系统摘要，交叉验证无异常
   - 快照续5 记录 design V2 写到第五章 → memory 续7 记录 design V3 + tasks V3 完成 → 无矛盾（续6/续7 在快照之后）
2. ✅ 读取 design.md V3 全文（1143行，14章完整）确认文件状态
3. ✅ 读取 tasks.md V3 全文（7 Phase、20 任务完整）确认文件状态
4. ✅ 同步主 memory：追加补丁002续7记录（从"design V2 + tasks V2 完成"更新为"design V3 + tasks V3 完成"）
5. ✅ 回应用户问题：确认 tasks 执行时会严格参照 design 详细内容，V3 的每个任务自包含执行细节

**修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `memory.md`（农田系统主） | 追加 | 补丁002续7记录同步 |
| `memory.md`（补丁002子） | 追加 | 本条记录 |

**状态**: design V3 + tasks V3 确认完整，主 memory 已同步，待用户审核后开始实现


### 会话 1（续9）- 2026-02-19（继承恢复 + 一条龙执行 Phase 1~6，被压缩中断）

**来源**：继承恢复自会话1续8 + 用户确认"来吧走起"开始一条龙执行

**用户需求**：
> design V3 + tasks V3 已审核通过，一条龙执行全部 20 个任务

**完成任务**：
1. ✅ 继承恢复：快照（续5）、memory（续8）、系统摘要交叉验证无异常
2. ✅ 加载 communication.md 获取一条龙模式指令
3. ✅ 全部 20 个叶子任务标记为 queued
4. ✅ **Phase 1 完成**（独立模块）：
   - 任务 1.1：FarmToolPreview.LockPosition 渲染 1+8（代码已存在，确认正确）
   - 任务 1.2：CropController AlignSpriteBottom（代码已存在，确认正确）
5. ✅ **Phase 2 完成**（队列基础设施）：
   - 任务 2.1：FarmActionType 枚举 + FarmActionRequest 结构体 + 队列字段（代码已存在）
   - 任务 2.2：EnqueueAction 入队方法（防重复 + Harvest 额外防重复）
   - 任务 2.3：ProcessNextAction 出队执行（暂停检查 + 二次验证 + 距离判断 + 导航回调）
   - 任务 2.4：ExecuteFarmAction 执行分发（Till/Water/PlantSeed/Harvest 四种类型）+ FaceTarget + GetCurrentSeedData
   - 任务 2.5：OnFarmActionAnimationComplete + OnCollectAnimationComplete + ClearActionQueue + HasSeedRemaining
6. ✅ **Phase 3 完成**（入队入口改造）：
   - 任务 3.1：TryDetectAndEnqueueHarvest 收获检测（层级过滤 CP-6 + IInteractable 接口）+ CropController 新增 LayerIndex/CellPos 属性
   - 任务 3.2：TryEnqueueFarmTool + TryEnqueueSeed + TryEnqueueFromCurrentInput
   - 任务 3.3：HandleUseCurrentTool 全面改造（保护分支替换 CacheFarmInput → TryEnqueueFromCurrentInput，收获优先 CP-5，农田工具入队，种子入队，其他工具不变 CP-19）
7. ✅ **Phase 4 完成**（OnActionComplete 分支改造）：
   - 任务 4.1：Collect 专用分支（shouldContinue 之前拦截，调用 OnCollectAnimationComplete）
   - 任务 4.2：农田工具长按分支改为 OnFarmActionAnimationComplete，松开分支改为 ClearActionQueue/OnFarmActionAnimationComplete
8. ✅ **Phase 5 完成**（中断/过滤/暂停）：
   - 任务 5.1：HandleMovement WASD 中断队列（lockManager.IsLocked 之前）
   - 任务 5.2：HandleRightClickAutoNav 过滤 CropController
   - 任务 5.3：面板暂停/恢复（_wasUIOpen + CancelCurrentNavigation 轻量版）
   - 任务 5.4：ESC + 快捷栏切换清空队列
9. ✅ **Phase 6 完成**（旧方法废弃）：
   - 任务 6.1：5 个方法 + 3 个字段标记 [System.Obsolete]，确认所有调用点已替换

**未完成**：
10. ❌ Phase 7 集成验证（4 个任务：7.1 编译验证、7.2 CP 审查、7.3 交互矩阵清单、7.4 memory 更新）
11. ❌ 验收指南.md 创建

**修改代码文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `FarmToolPreview.cs` | 修改 | LockPosition 新增锄头模式 1+8 GhostTilemap 渲染 |
| `CropController.cs` | 修改 | 新增 AlignSpriteBottom + UpdateVisuals 调用 + LayerIndex/CellPos 公开属性 |
| `GameInputManager.cs` | 大量修改 | 新增枚举/结构体/队列字段、EnqueueAction、ProcessNextAction、ExecuteFarmAction、回调方法、TryDetectAndEnqueueHarvest、TryEnqueueFarmTool/Seed/FromCurrentInput、HandleUseCurrentTool 改造、WASD 中断、右键过滤、面板暂停、ESC/快捷栏清空、旧方法废弃 |
| `PlayerInteraction.cs` | 修改 | OnActionComplete Collect 专用分支 + 农田工具分支改为队列出队 |

**修改文档文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `tasks.md` | 更新 | 任务 1.1~6.1 标记 [x]，7.1~7.4 标记 [~] |

**状态**: 🚧 Phase 1~6 代码实现全部完成，Phase 7 集成验证未开始，被压缩中断


### 会话 1（续10）- 2026-02-19（继承恢复 + Phase 7 集成验证完成 + 全部 20 任务完成）

**来源**：继承恢复自会话1续9

**完成任务**：
1. ✅ 继承恢复：快照（续9）、memory（续9）、系统摘要交叉验证无异常
2. ✅ **任务 7.1**：getDiagnostics 编译验证 — 4 个修改文件全部 0 错误 0 警告
   - GameInputManager.cs ✅
   - PlayerInteraction.cs ✅
   - FarmToolPreview.cs ✅
   - CropController.cs ✅
3. ✅ **任务 7.2**：CP-1~CP-19 正确性属性逐项代码审查 — 19/19 全部满足
4. ✅ **任务 7.3**：验收指南.md 创建 — 基于 design V3 第十一章 6 个交互矩阵生成可勾选测试清单（58 项）
5. ✅ **任务 7.4**：本条 memory 记录

**新建文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `验收指南.md` | 新建 | 运行时验收清单，7 大类 58 项可勾选测试 |

**修改文件**：
| 文件 | 操作 | 说明 |
|------|------|------|
| `tasks.md` | 更新 | 任务 7.1~7.4 标记 [x] |
| `memory.md`（补丁002子） | 追加 | 本条记录 |

**全部修改代码文件汇总（Phase 1~6，续9 完成）**：
| 文件 | 说明 |
|------|------|
| `GameInputManager.cs` | FIFO 队列系统全部实现（枚举/结构体/字段/入队/出队/执行/回调/入口改造/中断/过滤/暂停/废弃） |
| `PlayerInteraction.cs` | OnActionComplete Collect 专用分支 + 农田工具分支队列出队 |
| `FarmToolPreview.cs` | LockPosition 新增锄头模式 1+8 GhostTilemap 渲染 |
| `CropController.cs` | AlignSpriteBottom + LayerIndex/CellPos 公开属性 |

**状态**: ✅ 全部 20 个任务完成（7 Phase），补丁002 代码实现 + 集成验证全部完成


### 会话 1（续12）- 2026-02-19（002 验收反馈 + 003 补丁代码彻查开始，被压缩中断）

**来源**：用户验收 002 补丁后报告 6 个严重问题，要求进入 003 补丁

**用户反馈（6 个问题）**：
1. 🔴 作物位置完全错误 — 种在耕地 tile 底部边界处，远离实际耕地位置（截图确认）
2. 🔴 耕地/浇水长按左键退化 — 长按只触发一次，丧失之前的连续长按功能
3. 🔴 连续左键队列 bug — 动画期间点击近距离 B 地，A 动画结束瞬间 B 直接变耕地但无动画/朝向更新，队列卡住
4. 取消导航期间预览变更 — 用户确认改为连续左键直接入队
5. 🔴 种子坐标完全错误 — 与问题 1 同源，种子种下位置贴到耕地 tile 底部
6. 连续点击队列需全面优化 — 预览与执行分离，点击即显示预览+入队

**AI 代码彻查进度**：
1. ✅ 读取 CropController.AlignSpriteBottom — 确认实现：`localPos.y = -spriteBounds.min.y`（与 TreeController 一致）
2. ✅ 读取 CropController.Initialize — 确认 GameObject 位置由 `tilemaps.GetCellCenterWorld(cellPos)` 设置
3. ✅ 读取 ExecutePlantSeed — 确认种植坐标：`tilemaps.GetCellCenterWorld(cellPos)`，使用 `farmlandCenterTilemap`
4. ✅ 读取 LayerTilemaps.GetCellCenterWorld — 确认优先使用 `farmlandCenterTilemap.GetCellCenterWorld()`
5. ✅ 读取 HandleUseCurrentTool — 发现长按失效根因：代码用 `GetMouseButtonDown(0)` 而非 `GetMouseButton(0)`
6. ✅ 读取 OnActionComplete — 确认队列出队逻辑（Collect 专用分支 + 农田工具分支调用 OnFarmActionAnimationComplete）
7. ✅ 读取 ProcessNextAction — 确认队列执行引擎逻辑
8. ✅ 读取 ExecuteFarmAction — 确认动画触发和回调机制

**已发现的根因**：
- **问题 1/5 根因**：`farmlandCenterTilemap.GetCellCenterWorld()` 返回的坐标可能与实际耕地显示位置不一致（Tilemap 偏移或 Grid 配置问题），加上 `AlignSpriteBottom` 的 `-spriteBounds.min.y` 偏移，导致作物显示在耕地下方
- **问题 2 根因**：HandleUseCurrentTool 第 703 行 `bool isFirstPress = Input.GetMouseButtonDown(0)` + `if (!isFirstPress) return;` — 只检测按下瞬间，完全不检测持续按住，长按功能完全失效
- **问题 3 根因**：待进一步分析（可能与 ExecuteFarmAction 中 Till 的执行时序有关 — 先 RequestAction 播动画再 ExecuteTillSoil 执行逻辑，但回调链可能有问题）

**未完成**：
- ❌ 问题 1/5 的完整根因确认（需要检查 farmlandCenterTilemap 的实际 Grid 配置和偏移）
- ❌ 问题 3 的完整根因分析
- ❌ 问题 6 的方案设计
- ❌ 003 补丁工作区创建
- ❌ 003 补丁全面分析文档
- ❌ 003 补丁三件套（requirements/design/tasks）

**涉及代码文件（分析，未修改）**：
| 文件 | 分析内容 |
|------|---------|
| CropController.cs | AlignSpriteBottom、Initialize、UpdateVisuals |
| GameInputManager.cs | HandleUseCurrentTool、ExecutePlantSeed、ProcessNextAction、ExecuteFarmAction |
| PlayerInteraction.cs | OnActionComplete 队列出队逻辑 |
| LayerTilemaps.cs | GetCellCenterWorld 完整实现 |

**状态**: 🚧 代码彻查进行中，被压缩中断
