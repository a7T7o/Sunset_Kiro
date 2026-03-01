# 10.0.2 农作物视觉层 Prefab 驱动重构 — 需求文档

## 背景

10.0.1 完成了农作物系统的核心业务逻辑（状态机、收获、枯萎、保质期、存档、16个 PBT 测试）。
当前 CropController 的视觉表现（Sprite）从 SeedData SO 读取，限制了未来扩展性。
用户明确要求学习 TreeController 的 Prefab 驱动模式，将表现配置从 SO 移到 Prefab Inspector 上。

## 核心原则

1. SO 做减法，Prefab 做加法
2. 业务逻辑一行不动（状态机/收获/枯萎/保质期/存档全部保留）
3. 只改视觉层的数据来源（从 SeedData 改为 CropController 自身的 CropStageConfig）
4. 新增并兼容，不破坏现有功能

---

## 用户故事

### US-1：作物阶段配置 Prefab 化

作为开发者，我希望每种作物的生长阶段 Sprite 配置在 Prefab Inspector 上完成，
这样我可以直接在 Prefab 上预览和调整每个阶段的外观，无需修改 SO。

验收标准：
- AC-1.1：CropController 新增 `CropStageConfig[]` 序列化数组，每个元素包含 normalSprite 和 witheredSprite
- AC-1.2：CropController 的 `UpdateVisuals()` 从自身 `stages[]` 读取 Sprite，不再依赖 `seedData.growthStageSprites`
- AC-1.3：CropController 的 `GetCurrentSprite()` 从自身 `stages[]` 读取
- AC-1.4：所有依赖 `seedData.growthStageSprites.Length` 的阶段计算改为使用 `stages.Length`
- AC-1.5：枯萎 Sprite 从 `stages[index].witheredSprite` 读取，不再依赖 `seedData.witheredStageSprites`

### US-2：SeedData 精简

作为开发者，我希望 SeedData 只保留数值配置，不再持有 Sprite 数组，
这样 SO 更精简，批量工具改动更小。

验收标准：
- AC-2.1：SeedData 删除 `growthStageSprites` 字段
- AC-2.2：SeedData 删除 `witheredStageSprites` 字段
- AC-2.3：SeedData 新增 `GameObject cropPrefab` 字段（引用作物预制体）
- AC-2.4：SeedData 的 `OnValidate()` 移除对 growthStageSprites 的验证，新增对 cropPrefab 的验证
- AC-2.5：SeedData 保留所有数值字段不变（growthDays/season/harvestCropID/保质期等）

### US-3：旧版兼容代码同步更新

作为开发者，我希望所有引用 `seedData.growthStageSprites` 的旧代码都同步更新，
确保编译通过且逻辑正确。

验收标准：
- AC-3.1：CropManager.TryHarvest 中的 `seedData.growthStageSprites.Length` 改为从 controller 获取
- AC-3.2：CropInstance（已标记 Obsolete）中的 Sprite 引用同步更新或标记废弃
- AC-3.3：编译零错误

### US-4：批量 SO 生成工具同步

作为开发者，我希望 `Tool_BatchItemSOGenerator` 适配 SeedData 的新字段结构，
确保批量生成种子 SO 时不再设置已删除的 Sprite 数组字段。

验收标准：
- AC-4.1：`DrawSeedSettings()` 移除 Sprite 数组相关 UI（如果有）
- AC-4.2：`DrawSeedSettings()` 新增 cropPrefab 字段绘制
- AC-4.3：`CreateSeedData()` 不再设置 growthStageSprites/witheredStageSprites
- AC-4.4：`CreateSeedData()` 支持设置 cropPrefab

### US-5：DynamicObjectFactory 支持作物重建

作为开发者，我希望存档系统能正确重建作物对象，
使用 SeedData.cropPrefab 实例化作物预制体。

验收标准：
- AC-5.1：DynamicObjectFactory 新增 `TryReconstructCrop` 方法
- AC-5.2：从存档数据中获取 seedId，通过 ItemDatabase 找到 SeedData，使用 cropPrefab 实例化
- AC-5.3：实例化后调用 CropController.Load() 恢复状态
- AC-5.4：TryReconstruct 的 switch 分支新增 "Crop" 类型处理

### US-6：现有测试适配

作为开发者，我希望现有的 16 个 PBT 测试在重构后仍然全部通过，
因为业务逻辑没有变化。

验收标准：
- AC-6.1：CropSystemTests.cs 中所有测试编译通过
- AC-6.2：所有 16 个测试运行通过
- AC-6.3：如果测试中有引用 growthStageSprites 的逻辑，同步更新

---

## 技术约束

1. Unity 6 API：使用 `FindFirstObjectByType<T>()` 替代 `FindObjectOfType<T>()`
2. 存档兼容：存档存的是 seedID + stage + grownDays 等数值，不存 Sprite 引用，改动不影响存档格式
3. CropController.Save() 中的 `prefabId` 字段需要更新为有意义的值（用于 DynamicObjectFactory 重建）
4. 编码规范：遵循项目 Region 分组、命名规范、中文注释

## 不做的事情

1. 不重写 CropController 的业务逻辑
2. 不删除 CropData/WitheredCropData SO
3. 不改存档 DTO 结构
4. 不做 ItemData 继承链大手术（equipmentType 下沉是后续优化项）
