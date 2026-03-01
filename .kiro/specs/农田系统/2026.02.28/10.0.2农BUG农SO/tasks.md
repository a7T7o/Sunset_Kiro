# 10.0.2 农作物视觉层 Prefab 驱动重构 — 任务列表

## 任务总览

- [x] 1. 新增 CropStageConfig 结构体
  - [x] 1.1 创建 `CropStageConfig.cs`，定义 normalSprite + witheredSprite 字段
- [x] 2. 重构 CropController 视觉层
  - [x] 2.1 新增 `CropStageConfig[] stages` 序列化字段
  - [x] 2.2 修改 `GetCurrentSprite()` 从 stages 读取（含枯萎回退逻辑）
  - [x] 2.3 修改 `UpdateGrowthStage()` 使用 stages.Length
  - [x] 2.4 修改 `IsMatureStage()` 使用 stages.Length
  - [x] 2.5 修改 `HarvestMature()` 中重复收获阶段计算
  - [x] 2.6 修改 `GetTotalStages()` 使用 stages.Length
  - [x] 2.7 修改 `ResetForReHarvest()` 中阶段计算
- [x] 3. 精简 SeedData
  - [x] 3.1 删除 `growthStageSprites` 和 `witheredStageSprites` 字段
  - [x] 3.2 新增 `cropPrefab` 字段
  - [x] 3.3 更新 `OnValidate()` 验证逻辑
- [x] 4. 同步更新依赖代码
  - [x] 4.1 更新 CropManager.TryHarvest 中的阶段计算
  - [x] 4.2 处理 CropInstance（Obsolete）的编译兼容
- [x] 5. DynamicObjectFactory 支持作物重建
  - [x] 5.1 新增 `TryReconstructCrop` 方法
  - [x] 5.2 在 `TryReconstruct` 中新增 "Crop" 分支
- [x] 6. 批量 SO 工具适配
  - [x] 6.1 `DrawSeedSettings()` 新增 cropPrefab 绘制
  - [x] 6.2 `CreateSeedData()` 新增 cropPrefab 赋值
- [x] 7. 编译验证与测试
  - [x] 7.1 Unity 编译零错误验证
  - [x] 7.2 运行现有 CropSystemTests 确保全部通过
- [x] 8. 更新 farming.md 规则文档
- [x] 9. 更新 memory.md 并生成验收指南
