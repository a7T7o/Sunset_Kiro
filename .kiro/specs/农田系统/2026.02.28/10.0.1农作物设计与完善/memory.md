# 农作物设计与完善 - 开发记忆

## 模块概述

农作物控制器的全面设计，包含种子袋保质期系统、作物7样式生长系统、CropController重构、SO迭代方案。

## 当前状态

- **完成度**: 35%（设计+任务列表完成，待执行）
- **最后更新**: 2026-02-11
- **状态**: 📋 design.md + tasks.md 已创建，待用户确认后执行

---

## 会话记录

### 会话 6 - 2026-02-11（执行锐评001审视）

**用户需求**:
> 对 1执行锐评001.md 进行事实核查和审视

**完成任务**:
1. 读取锐评文件和所有相关代码文件
2. 对锐评6个核心声明进行事实核查 — 全部正确
3. 创建 `锐评001审视报告.md`
4. 发现2个额外问题：SeasonManager事件签名不一致、TryHarvestCropAtMouse与IInteractable链路冲突

**事实核查结果**:
| # | 声明 | 结论 |
|---|------|------|
| 1 | 腐烂食物移除动态属性确保可堆叠 | ✅ 正确 |
| 2 | ResourceNodeRegistry 是 O(N) 线性遍历 | ✅ 正确 |
| 3 | BoxCollider2D + Physics2D 利用 QuadTree | ✅ 正确 |
| 4 | CropData 继承 FoodData 获得食用逻辑 | ✅ 正确 |
| 5 | 保质期取最小值堵死延寿漏洞 | ✅ 正确 |
| 6 | 右键交互/左键工具的分工 | ✅ 正确 |

**额外发现**:
1. SeasonManager.OnSeasonChanged 是 `Action<Season>`，TimeManager.OnSeasonChanged 是 `Action<Season, int>` — 需确认订阅哪个
2. TryHarvestCropAtMouse 已存在于 GameInputManager — 需决定与 IInteractable 链路的关系

**采纳决策**: 完全采纳，可以开始执行 tasks.md

**产出文件**:
- `锐评001审视报告.md` - 新建
- `memory.md` - 更新（本文件）

**下一步**:
- [ ] 用户确认后按 Phase 顺序执行 tasks.md

---

### 会话 3 - 2026-02-11（全面审视）

**用户需求**:
> 请你认真思考帮助我进行全面的审视

**完成任务**:
1. 读取现有代码架构（SeedData, CropData, FoodData, InventoryItem, CropController）
2. 创建全面审视报告（8 章节，10,000+ 字）
3. 发现 10 个关键问题
4. 提出 9 项架构重构建议
5. 评估风险和工作量

**关键发现**:

**架构现状**:
- ✅ `InventoryItem` 已有完善的动态属性系统
- ✅ `CropController` 已实现自治模式
- ✅ 存档系统已完善
- ❌ `CropData` 当前不继承 `FoodData`（需要重构）
- ❌ 没有 `WitheredCropData`（需要新建）
- ❌ 没有 `RottenFoodData`（需要新建）

**关键问题**:
1. Q1：种子袋与种子的关系？（容器 vs 一次性物品）
2. Q2：种子袋内种子数量可变吗？
3. Q3：腐烂食物的 ID 是多少？（建议 5999）
4. Q6：WitheredCropData 的 ID 范围？（建议 1200-1299）
5. Q9：可重复收获作物是否会过熟枯萎？（建议不会）
6. Q10：全季节作物是否会过季枯萎？（建议不会）

**架构重构清单**:
1. R1：`CropData` 继承 `FoodData`（P0，中工作量）
2. R2：新建 `WitheredCropData : FoodData`（P0，低工作量）
3. R3：新建 `RottenFoodData : FoodData`（P0，低工作量）
4. R4：`SeedData` 新增字段（P0，低工作量）
5. R5：`CropInstanceData` 新增字段（P0，低工作量）
6. R6：`CropController` 新增枯萎逻辑（P0，中工作量）
7. R7：`CropController` 订阅季节事件（P0，低工作量）
8. R8：`InventoryService` 新增保质期检查（P0，中工作量）
9. R9：`CropController` 收割逻辑重构（P0，中工作量）

**用户体验优化建议**:
1. 增加"种子箱"系统（缓解背包压力）
2. 增加保质期倒计时 UI
3. 增加枯萎预警图标

**风险评估**:
- 高风险：`CropData` 继承重构可能导致现有代码崩溃
- 中风险：种子袋不可堆叠导致背包空间不足
- 缓解措施：全面测试、逐步迁移、增加种子箱系统

**产出文件**:
- `需求审视报告.md` - 新建（8 章节，10,000+ 字）
- `memory.md` - 更新（本文件）

**下一步**:
- 等待用户确认 Q1-Q10 的问题
- 确认架构重构方案
- 进入设计阶段（design.md）

---

### 会话 2 - 2026-02-11（需求补充与完善）

**用户需求**:
> 首先对于所有状态和内容都需要考虑可持久化，与当前存档内容结合起来，然后还有就是种子袋都是不可堆叠的，一个种子袋里面会有一定数量的种子，每个种子袋都是独立的，所以保质期也是独立的，这样不会有任何冲突，请你注意，成熟后超过两天就是比如当前是春季14天，作物刚好成熟了，如果今天没有收割，到了第二天也就是春季15天，则累计加一，到了春季16天还没收割，则累计加2，此时直接枯萎，然后还有对于过季也是全部枯萎，枯萎作物是独立的SO，就是作物可以有多种，一个是正常的，一个是枯萎的，然后对于控制器应该是需要索引两种SO，随时进行切换，然后对于腐烂的食物是可以堆叠的，这个物品是属于食物，可以作为其他配方的内容，也可以独立食用，只是会有其他buff，后续我们设计buff的时候会对所有可食用内容都进行完善，然后还有就是对于收获的作物其实也是可食用的，也是食物，但是作物就是更加纯粹的来源，食物可以是加工过的，但是后续我们对食用内容可能会进行全面完善

**完成任务**:
1. 更新 requirements.md - 补充8个关键需求点
2. 更新 memory.md - 记录本次会话

**关键需求补充**:
1. **可持久化要求** - 所有状态必须与存档系统结合
2. **种子袋不可堆叠** - stackable = false，每个种子袋独立
3. **成熟后枯萎计时精确定义** - daysSinceMature累计逻辑
4. **过季枯萎** - 季节切换时所有作物枯萎
5. **枯萎作物独立SO** - WitheredCropData，Controller索引两种SO
6. **腐烂食物可堆叠** - RottenFoodData继承FoodData，可作为配方材料
7. **作物可食用** - CropData继承FoodData
8. **食物系统一致性** - 所有可食用内容都是FoodData

**需求文档更新内容**:
- US-1 新增 AC-1.7/1.8/1.9（种子袋不可堆叠+可持久化）
- US-2 新增 AC-2.6/2.7/2.8（枯萎作物独立SO+可持久化）
- US-3 重构为"作物生长控制与枯萎机制"，新增 AC-3.4/3.5/3.6（枯萎计时+过季枯萎）
- US-4 新增 AC-4.5/4.6（作物可食用+腐烂食物特性）
- C-1 更新为三个SO类型（SeedData/CropData/WitheredCropData）
- C-4 新增"食物系统一致性"约束
- E-1 更新为"种子袋不可堆叠"
- E-3 更新为"过季全部枯萎"
- E-5 更新为"腐烂食物可堆叠+可作为配方材料"
- E-6 新增"枯萎作物收割"边界情况

**产出文件**:
- `requirements.md` - 更新
- `memory.md` - 更新（本文件）

**下一步**:
- 等待用户确认需求
- 确认后进入设计阶段（design.md）

---

### 会话 1 - 2026-02-10（全面分析 + 需求记录）

**用户需求**:
> 对农作物的控制器进行设计，包含对SO的迭代以及对农作物控制器的规划和设计：种子袋保质期系统（未打开7天/打开2天/过期变腐烂食物）、作物7样式生长系统（4生长+收获+枯萎+枯萎收获）、Controller控制生长间隔、SO只有种子和作物两个

**完成任务**:
1. 创建全面分析文档 — 10章节深度分析
2. 创建需求文档 — 5个用户故事、5个边界情况、3个设计约束

**关键决策**:
- 保质期方案：利用 InventoryItem 动态属性系统（不修改SO结构存储运行时状态）
- SeedData 新增：保质期配置（2个字段）+ 枯萎样式（1个字段）+ 种子袋数量（1个字段）
- CropData 新增：收获样式（1个字段），继承自 FoodData
- WitheredCropData 新增：独立SO，继承自 FoodData
- RottenFoodData 新增：独立SO，继承自 FoodData，可堆叠
- CropInstanceData 新增：daysSinceMature + isOverripe + currentSeason（用于检测过季）
- 新增"腐烂的食物"独立ItemData SO（FoodData）
- CropController 新增 CropState 状态机 + 双SO索引（正常+枯萎）
- **种子袋不可堆叠**（stackable = false）

**待用户确认的关键问题**:
- ~~Q1：打开后种子的堆叠规则~~ → 已确认：种子袋不可堆叠
- ~~Q2：枯萎作物的物品处理方式~~ → 已确认：独立SO（WitheredCropData）
- Q3：腐烂食物的ID分配 → 待确认
- Q4：种子袋的视觉表现 → 待确认
- ~~Q5：成熟后枯萎天数的精确定义~~ → 已确认：daysSinceMature累计逻辑

**产出文件**:
- `全面分析文档.md` - 新建
- `requirements.md` - 新建
- `memory.md` - 新建（本文件）

**涉及的代码文件（分析，未修改）**:

### 核心文件（分析）
| 文件 | 关系 |
|------|------|
| `SeedData.cs` | 需要新增保质期配置+枯萎样式字段 |
| `CropData.cs` | 需要新增收获样式+枯萎收获样式字段 |
| `CropController.cs` | 需要重构：状态机+枯萎逻辑+收割逻辑 |
| `CropInstanceData.cs` | 需要新增 daysSinceMature + isOverripe |
| `InventoryItem.cs` | 已有动态属性系统，无需修改 |
| `InventoryService.cs` | 需要新增保质期检查逻辑 |

### 参考文件
| 文件 | 关系 |
|------|------|
| `TreeController.cs` | Sprite切换模式参考 |
| `TreeSpriteData.cs` | Sprite配置结构参考 |
| `ItemData.cs` | 物品基类，了解字段结构 |
| `FoodData.cs` | 食物SO，了解食品类结构 |
| `ItemEnums.cs` | 枚举定义，了解ID范围和分类 |

---

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 保质期用动态属性而非SO字段 | 保质期是运行时状态，SO是共享的静态定义 | 2026-02-10 |
| 种子袋不可堆叠 | 每个种子袋保质期独立，避免冲突 | 2026-02-11 |
| 枯萎作物独立SO | 枯萎作物与正常作物是不同物品，需要独立配置 | 2026-02-11 |
| 作物继承FoodData | 作物可食用，统一食物系统 | 2026-02-11 |
| 腐烂食物可堆叠 | 腐烂食物是通用材料，需要可堆叠 | 2026-02-11 |
| 过季全部枯萎 | 增加游戏策略性，避免跨季种植 | 2026-02-11 |

### 会话 4 - 2026-02-11（全面分析文档V2 + Gemini设计协助消化）

**用户需求**:
> 消化 Gemini 的设计协助文档（0设计协助001.md），结合代码事实进行独立思考，输出全面分析文档V2

**完成任务**:
1. 读取所有关键代码文件（SeedData, CropData, FoodData, InventoryItem, CropController, GameInputManager, PlayerAutoNavigator, CropManager, CropInstanceData, ItemData, IInteractable）
2. 创建 `全面分析文档V2.md` — 8个核心设计决策 + 与Gemini方案对比 + 5阶段实施路线

**8个核心设计决策**:

| # | 决策 | 说明 |
|---|------|------|
| 1 | SeedData本身就是种子袋 | 不需要单独SeedBagData SO |
| 2 | 已打开/未打开图标存同一个SeedData | icon + iconOpened 双字段 |
| 3 | 右键背包打开 + 种植时自动打开 | 两种打开方式 |
| 4 | CropController实现IInteractable | 右键收获集成导航 |
| 5 | 暂用collect动画收获 | 无镰刀，后续可扩展 |
| 6 | 4个正常Sprite + 1个枯萎Sprite | 可扩展为4+4 |
| 7 | CropData改为继承FoodData | 作物可食用 |
| 8 | InventoryService订阅OnDayChanged检查保质期 | 每日检查过期种子袋 |

**与Gemini方案对比**:
- Gemini建议SeedBagData独立SO → 我认为不需要，SeedData本身就是种子袋
- Gemini建议7个Sprite → 我简化为4+1（可扩展为4+4）
- Gemini建议CropData继承FoodData → 完全认同
- Gemini建议IInteractable收获 → 完全认同

**5个待用户确认问题**:
- Q1：种子袋打开后图标是否变化？（建议：是）
- Q2：种子袋打开后能否重新关闭？（建议：不能）
- Q3：收获动画用collect还是新建？（建议：暂用collect）
- Q4：枯萎Sprite是1个还是4个？（建议：先1个，可扩展）
- Q5：保质期检查时机？（建议：OnDayChanged）

**产出文件**:
- `全面分析文档V2.md` - 新建（核心交付物）

**涉及的代码文件**:

### 核心文件（分析，未修改）
| 文件 | 关系 |
|------|------|
| `SeedData.cs` | 种子袋SO，需新增 iconOpened/seedCount/shelfLifeUnopened/shelfLifeOpened |
| `CropData.cs` | 作物SO，需改为继承 FoodData，新增 harvestSprite |
| `FoodData.cs` | 食物基类，CropData 将继承此类 |
| `InventoryItem.cs` | 动态属性系统，保质期用 SetProperty/GetProperty |
| `CropController.cs` | 作物控制器，需实现 IInteractable，重构收获逻辑 |
| `CropManager.cs` | 作物工厂，TryHarvest 是当前收获入口 |
| `CropInstanceData.cs` | 作物实例数据，需新增 daysSinceMature/currentSeason |
| `GameInputManager.cs` | 输入管理，HandleRightClickAutoNav 已有 IInteractable 链路 |
| `PlayerAutoNavigator.cs` | 导航器，FollowTarget 已支持回调 |
| `ItemData.cs` | 物品基类 |
| `IInteractable.cs` | 交互接口 |

### 参考文件
| 文件 | 关系 |
|------|------|
| `TreeController.cs` | IInteractable 实现参考 |
| `ChestController.cs` | IInteractable 实现参考 |

**遗留问题**:
- [ ] 用户审核全面分析文档V2中的5个待确认问题
- [ ] 审核通过后创建 design.md
- [ ] 然后创建 tasks.md



---

### 会话 5 - 2026-02-11（锐评002审视 + design.md + tasks.md）

**用户需求**:
> 锐评请你开始审核，包含对你v2的锐评和审核，请你客观阅读并分析，如果有反驳请你给出如果没有反驳请你总结然后完成后续任务

**完成任务**:
1. 审视锐评002 — 5个观点全部事实正确，全部采纳，无反驳
2. 创建 `锐评002审视报告.md` — 事实核查 + 采纳决策
3. 创建 `design.md` — 8章节完整设计文档
4. 创建 `tasks.md` — 5个Phase、22个任务

**锐评002事实核查结果**:
| # | 锐评声明 | 核查结论 |
|---|---------|---------|
| 1 | ResourceNodeRegistry.GetNodesInRange 是 O(N) 线性遍历 | ✅ 正确 |
| 2 | InventoryItem.CanStackWith 中 properties.Count > 0 锁死堆叠 | ✅ 正确 |
| 3 | TreeController 有 Shadow/Shake/Fall/IResourceNode，作物不需要 | ✅ 正确 |
| 4 | CropData 当前继承 ItemData 需改为 FoodData | ✅ 正确 |
| 5 | SeedData 直接加字段不需要 SeedBagData | ✅ 正确 |

**design.md 核心内容**:
- SO继承体系：CropData:FoodData, WitheredCropData:FoodData, SeedData:ItemData
- CropController：CropState 4状态枚举 + IInteractable + BoxCollider2D Trigger
- 种子袋保质期：动态属性 bag_opened/bag_remaining/shelf_expire_day
- 收获交互：右键 IInteractable 导航收获
- 正确性属性：P1-P5（保质期单调递减、腐烂食物无属性、状态转换合法、收获一致、过季覆盖）

**tasks.md 结构**:
- Phase 1: SO层重构（4任务）
- Phase 2: CropController重构（8任务）
- Phase 3: 种子袋保质期系统（5任务）
- Phase 4: 收获交互集成（3任务）
- Phase 5: 测试（5任务）

**产出文件**:
- `锐评002审视报告.md` - 新建
- `design.md` - 新建
- `tasks.md` - 新建
- `memory.md` - 更新（本文件）

**涉及的代码文件**:

### 核心文件（设计涉及，未修改）
| 文件 | 操作 | 说明 |
|------|------|------|
| `CropData.cs` | 待修改 | 改为继承 FoodData |
| `SeedData.cs` | 待修改 | 新增 seedsPerBag/shelfLife/iconOpened/witheredStageSprites |
| `CropController.cs` | 待修改 | CropState 枚举 + IInteractable + 收获逻辑 |
| `WitheredCropData.cs` | 待新增 | 枯萎作物SO |
| `CropInstanceData.cs` | 待修改 | 新增 daysSinceMature |
| `InventoryService.cs` | 待修改 | 日结保质期检查 |

### 相关文件（引用/依赖）
| 文件 | 关系 |
|------|------|
| `FoodData.cs` | CropData 将继承此类 |
| `ItemData.cs` | 物品基类 |
| `InventoryItem.cs` | 动态属性系统 |
| `GameInputManager.cs` | 右键交互入口 |
| `PlayerAutoNavigator.cs` | 导航到作物 |
| `ResourceNodeRegistry.cs` | 作物不注册到此（锐评确认） |

**下一步**:
- [ ] 等待用户确认 tasks.md
- [ ] 按 Phase 顺序执行任务


---

### 会话 7 - 2026-02-11（一条龙执行全部任务）

**用户需求**:
> 一条龙完成 tasks.md 中所有任务

**完成任务**:

**Phase 1: SO层重构** ✅
1. `CropData.cs` — 改为继承 `FoodData`，新增 `seedID`、`harvestExp`、`witheredCropID`
2. `SeedData.cs` — 新增 `seedsPerBag`、`shelfLifeClosed`、`shelfLifeOpened`、`iconOpened`、`witheredStageSprites`
3. `WitheredCropData.cs` — 新建，继承 `FoodData`，字段 `normalCropID`、`seedID`
4. 编译验证通过

**Phase 2: CropController重构** ✅
1. `CropState` 枚举（Growing/Mature/WitheredImmature/WitheredMature）
2. `IInteractable` 实现（Priority=40, Distance=1.2f）
3. `BoxCollider2D` Trigger 配置（不注册 ResourceNodeRegistry）
4. `OnDayChanged` 生长与枯萎逻辑
5. `OnSeasonChanged` 过季枯萎
6. `UpdateVisuals` 视觉更新
7. 收获逻辑（Mature→CropData, WitheredMature→WitheredCropData）
8. `CropInstanceData` 扩展（daysSinceMature）

**Phase 3: 种子袋保质期系统** ✅
1. `SeedBagHelper.cs` — 新建，种子袋动态属性初始化/打开/消耗
2. 打开种子袋逻辑（bag_opened/shelf_expire_day 重算）
3. 种植消耗逻辑（bag_remaining 减1，为0时移除）
4. `InventoryService.cs` — 日结保质期检查（过期→腐烂食物ID 5999，无动态属性确保可堆叠）
5. 存档兼容（旧种子袋无动态属性→默认未打开）

**Phase 4: 收获交互集成** ✅
1. 右键点击收获流程（IInteractable + BoxCollider2D Trigger）
2. 枯萎未成熟作物清除（锄头左键 → ClearWitheredImmature）
3. 收获物品添加到背包（满时 WorldSpawnService 掉落）

**Phase 5: 测试** ✅
1. `CropSystemTests.cs` — 新建，16个测试全部通过
   - 5.1: CropState 状态转换测试（合法路径/非法路径/随机序列/枯萎终态）
   - 5.2: 种子袋保质期测试（单调递减/打开上限/过期检测/双开幂等）
   - 5.3: 腐烂食物堆叠测试（无属性可堆叠/有属性不可堆叠）
   - 5.4: 收获产出一致性测试（Mature→CropItem/WitheredMature→WitheredCropItem/其他→None/数量范围）
   - 5.5: 过季枯萎覆盖测试（非全季枯萎/全季不受影响/同季不枯萎/已枯萎不变）

**修改文件**:

### 核心文件（修改/新增）
| 文件 | 操作 | 说明 |
|------|------|------|
| `CropData.cs` | 修改 | 继承 FoodData，新增 seedID/harvestExp/witheredCropID |
| `SeedData.cs` | 修改 | 新增 seedsPerBag/shelfLifeClosed/shelfLifeOpened/iconOpened/witheredStageSprites |
| `WitheredCropData.cs` | 新增 | 枯萎作物SO，继承 FoodData |
| `CropController.cs` | 修改 | CropState枚举+IInteractable+收获逻辑+ClearWitheredImmature+DropItemToWorld |
| `SeedBagHelper.cs` | 新增 | 种子袋动态属性管理 |
| `SaveDataDTOs.cs` | 修改 | CropSaveData 新增 daysSinceMature |
| `InventoryService.cs` | 修改 | 日结保质期检查 |
| `GameInputManager.cs` | 修改 | ExecuteTillSoil 添加 WitheredImmature 检测 |
| `FarmToolPreview.cs` | 修改 | UpdateHoePreview 添加 canClearWithered 检查 |
| `CropSystemTests.cs` | 新增 | 16个PBT测试 |

### 相关文件（引用/依赖）
| 文件 | 关系 |
|------|------|
| `FoodData.cs` | CropData/WitheredCropData 基类 |
| `ItemData.cs` | SeedData 基类 |
| `InventoryItem.cs` | 动态属性系统 |
| `CropManager.cs` | 作物工厂 |
| `FarmTileManager.cs` | 耕地状态管理 |
| `WorldSpawnService.cs` | 物品掉落 |
| `TimeManager.cs` | 时间事件源 |

**编译状态**: ✅ 成功
**测试状态**: ✅ 16/16 通过

**遗留问题**:
- [ ] 用户游戏内验收（SO资产配置、种植流程、收获流程、保质期流程）
- [ ] WitheredCropData SO 资产需要手动创建（ID 12XX）
- [ ] 腐烂食物 SO 资产需要确认（ID 5999）
- [ ] SeedData SO 需要在 Inspector 中配置新字段默认值



---

### 会话 8 - 2026-02-11（2bug锐评001 修复 + 批量SO工具同步）

**用户需求**:
> 处理 2bug锐评001.md 中的 bug 修复，同步批量SO生成工具

**锐评来源**: `.kiro/specs/农田系统/10.0.1农作物设计与完善/2bug锐评001.md`

**锐评事实核查**:
| # | 问题 | 核查结论 | 处理 |
|---|------|---------|------|
| 1 | 保质期作弊后门 | ✅ 确认存在 | 已修复 |
| 2 | 枯萎清除无反馈 | ✅ 确认存在 | 已添加占位 |
| 3 | 数组越界风险 | ❌ 事实错误（Mathf.Clamp已保护） | 跳过 |
| 4 | UI绑定缺失 | 🔍 推测性声明 | 仅验证 |

**路径判断**: 路径 B（部分正确，直接执行）

**完成任务**:

1. **Fix #1（关键）: 保质期作弊后门** ✅
   - `GameInputManager.ExecutePlantSeed` 中原来使用 `inventory.RemoveItem(seedData.itemID, -1, 1)` 绕过了种子袋系统
   - 修复为：通过 `inventory.GetInventoryItem(consumedSlotIndex)` 获取实际 InventoryItem
   - 调用 `SeedBagHelper.IsExpired()` 检查过期
   - 调用 `SeedBagHelper.ConsumeSeed()` 正确消耗（自动打开未开封种子袋）
   - 当 remaining <= 0 时调用 `inventory.ClearSlot(consumedSlotIndex)` 清除
   - 修复回滚逻辑：种植失败时恢复种子（bag_remaining +1）而非 AddItem

2. **Fix #2: 枯萎清除反馈** ✅
   - `CropController.ClearWitheredImmature` 中添加 TODO 注释和 Debug.Log 占位
   - 后续添加 VFX/SFX 时替换

3. **Fix #3 跳过**: 数组越界已有 Mathf.Clamp 保护 ✅

4. **批量SO工具同步** ✅
   - `Tool_BatchItemSOGenerator.cs` 全面同步：
     - 新增 `WitheredCropData = 12` 到 ItemSOType 枚举
     - 新增到 CategoryToSubTypes、SubTypeNames、SubTypeStartIDs(1200)、SubTypeOutputFolders
     - Seed 新增字段：seedsPerBag、shelfLifeClosed/Opened、isReHarvestable/reHarvestDays
     - Crop 新增字段：witheredCropID
     - 新增 DrawWitheredCropSettings() 方法
     - 新增 CreateWitheredCropData() 方法
     - 更新 GetFilePrefix（WitheredCropData → "WCrop"）
     - 更新 LoadSettings/SaveSettings 所有新 EditorPrefs

**编译状态**: ✅ Unity 编译通过，0 警告

**修改文件**:

### 核心文件（修改）
| 文件 | 操作 | 说明 |
|------|------|------|
| `GameInputManager.cs` | 修改 | ExecutePlantSeed 修复保质期作弊后门 |
| `CropController.cs` | 修改 | ClearWitheredImmature 添加反馈占位 |
| `Tool_BatchItemSOGenerator.cs` | 修改 | 全面同步 WitheredCropData/Seed/Crop 新字段 |

### 相关文件（引用/依赖）
| 文件 | 关系 |
|------|------|
| `SeedBagHelper.cs` | Fix #1 调用的种子袋管理器 |
| `InventoryService.cs` | Fix #1 调用的背包服务 |

**遗留问题**:
- [ ] 枯萎清除的 VFX/SFX 后续实现
- [ ] UI 绑定（种子袋图标切换）待后续 UI 系统完善时处理
