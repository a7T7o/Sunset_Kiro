# 10.0.3 收获 DropTable 与动画 — 需求文档

## 背景

10.0.2 完成了 CropStageConfig 的 Prefab 驱动重构和 daysToNextStage 累加模式。本工作区继续深度学习树木模式，将作物收获机制从"直接加背包"改为"掉落到地面"，并集成 Collect 动画交互。

## 前置依赖

- 10.0.2 已完成：CropStageConfig + daysToNextStage 累加模式 + 固定 4 阶段

## 设计约束

- DC-1：新增并兼容，不破坏现有功能
- DC-2：收获掉落模式学习 TreeController 的 `dropItemData` + `SpawnMultiple` 模式
- DC-3：Collect 动画（AnimState.Collect = 4）已存在，只需集成触发
- DC-4：SeedData 字段精简需保持存档兼容（旧字段标记 `[Obsolete]` 而非直接删除）

---

## 用户故事

> **⚠️ 覆盖说明**：US-1、US-2、US-4、US-5 已在 `10.0.2.1纠正` 工作区中全面完成（废弃 CropManager + 对齐树木 Prefab 驱动模式）。仅 US-3（Collect 动画集成）为本工作区剩余待实现内容。

### US-1：成熟作物收获改用掉落模式 ✅ 已由 10.0.2.1 纠正完成

作为玩家，我收获成熟作物时，产出物品应该掉落到地面（与砍树掉落一致），而不是直接进入背包。

验收标准：
- AC-1.1：成熟作物收获时，物品通过 `WorldSpawnService.SpawnMultiple()` 掉落到作物位置
- AC-1.2：掉落物品由 CropController 上配置的 `dropItemData`（ItemData SO）和 `dropAmount` 决定
- AC-1.3：掉落物散布半径与树木一致（默认 0.4f）
- AC-1.4：可重复收获的作物（如草莓）收获后重置到生长阶段，继续生长

### US-2：枯萎成熟作物收获调整 ✅ 已由 10.0.2.1 纠正完成

作为玩家，我收获枯萎的成熟作物时，应该掉落枯萎作物到地面。

验收标准：
- AC-2.1：枯萎成熟作物收获时，通过 `witheredDropItemData`（ItemData SO）掉落枯萎作物
- AC-2.2：如果未配置 `witheredDropItemData`，则不掉落任何物品，直接清除作物
- AC-2.3：枯萎作物掉落数量固定为 1（不受 dropAmount 影响）
- AC-2.4：枯萎作物不可重复收获，收获后销毁

### US-3：收获交互集成 Collect 动画 ⏳ 待实现

作为玩家，我右键点击成熟作物后，角色应该播放 Collect 动画（弯腰拔起），在动画关键帧触发收获逻辑。

验收标准：
- AC-3.1：右键点击成熟/枯萎成熟作物 → 导航到位 → 播放 Collect 动画（AnimState.Collect = 4）
- AC-3.2：收获逻辑在 Collect 动画的事件帧触发（而非动画开始时立即执行）
- AC-3.3：如果导航到位后作物状态已变化（被其他因素影响），不执行收获

### US-4：SeedData 字段精简 ✅ 已由 10.0.2.1 纠正完成

作为开发者，SeedData 中被 DropTable 和 CropStageConfig 替代的字段应该标记废弃，减少数据冗余。

验收标准：
- AC-4.1：`SeedData.growthDays` 标记 `[Obsolete]`（已被 CropStageConfig.daysToNextStage 替代）
- AC-4.2：`SeedData.harvestCropID` 标记 `[Obsolete]`（已被 CropController.dropItemData 替代）
- AC-4.3：`SeedData.harvestAmountRange` 标记 `[Obsolete]`（已被 CropController.dropAmount 替代）
- AC-4.4：保留 `isReHarvestable`、`reHarvestDays`、`maxHarvestCount`（重复收获逻辑不变）
- AC-4.5：旧字段保留在代码中不删除，确保存档兼容和渐进迁移

### US-5：品质系统适配 ✅ 已由 10.0.2.1 纠正完成

作为玩家，收获作物的品质应该由掉落配置控制，而非随机骰子。

验收标准：
- AC-5.1：掉落品质由 CropController 上配置的 `dropQuality` 字段决定（默认 0 = Normal）
- AC-5.2：移除 `DetermineHarvestQuality()` 随机品质方法（标记 `[Obsolete]`）
- AC-5.3：后续品质系统（如土壤肥力影响品质）预留扩展点，本期不实现

---

## 边界情况

- E-1：`dropItemData` 未配置时，收获不掉落任何物品，仅清除作物（不报错崩溃）
- E-2：`WorldSpawnService.Instance` 为 null 时，降级为日志警告，不崩溃
- E-3：可重复收获作物达到 `maxHarvestCount` 上限时，最后一次收获后销毁
- E-4：Collect 动画播放过程中作物被外部因素销毁（如日结枯萎），动画正常结束但不执行收获
- E-5：导航到位后玩家切换了手持物品，仍然执行收获（收获不依赖手持物品）

---

## 非功能需求

- NF-1：所有修改通过 Unity 编译，零错误
- NF-2：现有 CropSystemTests 测试不被破坏（可能需要适配）
- NF-3：存档兼容——旧存档加载后作物行为正常（旧字段标记 Obsolete 但不删除）
