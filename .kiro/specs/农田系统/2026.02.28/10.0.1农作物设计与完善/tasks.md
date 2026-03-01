# 农作物控制器设计与SO迭代 - 任务列表

**创建日期**: 2026-02-11
**依据**: design.md + requirements.md + 锐评002审视报告

---

## Phase 1: SO 层重构

- [x] 1.1 CropData 继承重构
  - 将 `CropData` 从继承 `ItemData` 改为继承 `FoodData`
  - 新增字段：`seedID`、`harvestExp`、`witheredCropID`
  - 移除与 FoodData 重复的字段
  - 确保现有 CropData SO 资产不丢失数据（Unity 序列化兼容）
  - 验证：`CropData cropData = database.GetItem(id) as CropData` 仍然有效

- [x] 1.2 SeedData 新增字段
  - 新增 `seedsPerBag`（int, 默认5）
  - 新增 `shelfLifeClosed`（int, 默认7）
  - 新增 `shelfLifeOpened`（int, 默认2）
  - 新增 `iconOpened`（Sprite）
  - 新增 `witheredStageSprites`（Sprite[]）
  - 确保旧 SeedData SO 资产兼容（新字段使用默认值）

- [x] 1.3 新建 WitheredCropData
  - 创建 `WitheredCropData.cs`，继承 `FoodData`
  - 字段：`normalCropID`（int）、`seedID`（int）
  - CreateAssetMenu：`Farm/Items/Withered Crop`
  - ID 范围：12XX

- [x] 1.4 编译验证与 SO 资产检查
  - 确保所有 SO 脚本编译通过
  - 检查现有 CropData SO 在 Inspector 中字段是否正常
  - 检查现有 SeedData SO 新字段默认值是否正确


---

## Phase 2: CropController 重构

- [x] 2.1 CropState 枚举与状态机
  - 新增 `CropState` 枚举：Growing、Mature、WitheredImmature、WitheredMature
  - CropController 新增 `CropState state` 字段
  - 实现状态转换逻辑（仅允许合法路径）
  - 状态转换合法路径：
    - Growing → Mature（达到最后阶段）
    - Growing → WitheredImmature（缺水/过季）
    - Mature → WitheredMature（过熟/过季）
    - Mature → Growing（可重复收获作物收割后重置）

- [x] 2.2 IInteractable 实现
  - CropController 实现 `IInteractable` 接口
  - `InteractionPriority = 40`
  - `InteractionDistance = 1.2f`
  - `CanInteract()`: Mature 或 WitheredMature 时返回 true
  - `OnInteract()`: 调用 Harvest()
  - `GetInteractionHint()`: 返回"收获"或"收获（枯萎）"

- [x] 2.3 BoxCollider2D Trigger 配置
  - CropController 的 GameObject 添加 BoxCollider2D（IsTrigger = true）
  - 不注册到 ResourceNodeRegistry
  - 确保 HandleRightClickAutoNav 的 Physics2D 检测能找到作物

- [x] 2.4 OnDayChanged 生长与枯萎逻辑
  - Growing 状态：检查浇水 → 生长或停滞，检查缺水天数 → 可能 WitheredImmature
  - Mature 状态：daysSinceMature++，达到2天 → WitheredMature
  - 可重复收获作物（isReHarvestable）不触发过熟枯萎
  - 调用 UpdateVisuals() 更新 Sprite

- [x] 2.5 OnSeasonChanged 过季枯萎
  - 订阅 SeasonManager 季节变化事件
  - 非 AllSeason 且不匹配新季节时：
    - Mature → WitheredMature
    - Growing → WitheredImmature
  - 调用 UpdateVisuals() 更新 Sprite

- [x] 2.6 UpdateVisuals 视觉更新
  - WitheredImmature/WitheredMature：使用 `seedData.witheredStageSprites`
  - Growing/Mature：使用 `seedData.growthStageSprites`
  - Mature 状态：启用成熟闪烁效果（可选）
  - 枯萎状态：应用枯萎颜色

- [x] 2.7 收获逻辑（Harvest）
  - Mature 收获：产出 CropData 物品，数量随机，品质随机判定
  - WitheredMature 收获：产出 WitheredCropData 物品，品质固定 Normal
  - 可重复收获：重置到指定阶段，state = Growing
  - 不可重复收获：销毁 GameObject
  - 枯萎作物不可重复收获（即使原作物是可重复收获的）

- [x] 2.8 CropInstanceData 扩展
  - 新增 `daysSinceMature`（int, 默认0）
  - CropSaveData 同步新增 `daysSinceMature`
  - 存档加载时默认值为0，兼容旧存档


---

## Phase 3: 种子袋保质期系统

- [x] 3.1 种子袋动态属性初始化
  - 购买/获得种子袋时设置动态属性：
    - `bag_opened` = false
    - `bag_remaining` = seedData.seedsPerBag
    - `shelf_expire_day` = currentTotalDays + seedData.shelfLifeClosed
  - 确保 InventoryItem 的动态属性系统正确存储

- [x] 3.2 打开种子袋逻辑
  - 触发条件：背包右键 或 种植时自动打开
  - 设置 `bag_opened` = true
  - 保质期重算：`Min(当前剩余天数, shelfLifeOpened)`
  - 已打开的种子袋不可重复打开

- [x] 3.3 种植消耗逻辑
  - 种植时自动打开未打开的种子袋
  - `bag_remaining` 减 1
  - 剩余为 0 时从背包移除该物品
  - 否则更新 `bag_remaining`

- [x] 3.4 InventoryService 日结保质期检查
  - InventoryService 订阅 `TimeManager.OnDayChanged`
  - 遍历所有背包槽位，检查 `shelf_expire_day`
  - 过期物品替换为腐烂食物（ID 5999）
  - 腐烂食物生成时不设置任何动态属性（确保 properties.Count == 0，可堆叠）
  - 尝试与已有腐烂食物堆叠

- [x] 3.5 种子袋存档兼容
  - 动态属性通过 InventoryItem 的序列化系统自动持久化
  - 旧存档中的种子袋没有动态属性，加载时默认为"未打开"状态
  - 验证存档保存/加载后动态属性完整性

---

## Phase 4: 收获交互集成

- [x] 4.1 右键点击收获流程
  - HandleRightClickAutoNav 检测到 CropController（IInteractable + BoxCollider2D Trigger）
  - CanInteract() == true（Mature/WitheredMature）→ 导航 + 收获
  - CanInteract() == false（Growing/WitheredImmature）→ 走普通导航逻辑
  - 近距离直接 OnInteract()，远距离 FollowTarget() 导航后 OnInteract()

- [x] 4.2 枯萎未成熟作物清除
  - 左键使用锄头检测到 WitheredImmature 作物
  - 播放锄头动画 → 清除作物 GameObject → 耕地恢复为空
  - 不产出任何物品

- [x] 4.3 收获物品添加到背包
  - 调用 InventoryService 添加收获物品
  - Mature：添加 CropData 物品（数量随机、品质随机）
  - WitheredMature：添加 WitheredCropData 物品（数量随机、品质 Normal）
  - 背包满时物品掉落到地面（WorldItemPickup）

---

## Phase 5: 测试

- [x] 5.1 CropState 状态转换测试（PBT）
  - 验证正确性属性 P3：状态转换合法性
  - 随机生成状态转换序列，验证只有合法路径被允许
  - 验证非法路径被拒绝（WitheredImmature → Growing 等）
  - **Validates: Requirements AC-3.4, AC-3.5**

- [x] 5.2 种子袋保质期测试（PBT）
  - 验证正确性属性 P1：保质期单调递减
  - 随机生成打开/不打开序列，验证 shelf_expire_day 不增加
  - 验证过期后正确替换为腐烂食物
  - **Validates: Requirements AC-1.2, AC-1.3, AC-1.4**

- [x] 5.3 腐烂食物堆叠测试
  - 验证正确性属性 P2：腐烂食物无动态属性
  - 生成腐烂食物后 properties.Count == 0
  - 两个腐烂食物 CanStackWith() 返回 true
  - **Validates: Requirements AC-1.5, AC-1.6**

- [x] 5.4 收获产出一致性测试
  - 验证正确性属性 P4：收获产出一致性
  - Mature 收获 → CropData 物品
  - WitheredMature 收获 → WitheredCropData 物品
  - Growing/WitheredImmature → 不可收获
  - **Validates: Requirements AC-4.1, AC-4.3**

- [x] 5.5 过季枯萎覆盖测试
  - 验证正确性属性 P5：过季枯萎全覆盖
  - 季节切换时所有非 AllSeason 作物都变为枯萎状态
  - AllSeason 作物不受影响
  - **Validates: Requirements AC-3.5**
