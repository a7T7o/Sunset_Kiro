# 补丁005 施工手册

> 工作区：10.1.5补丁005
> 基于：design.md
> 创建时间：2026-02-25
> 状态：✅ 已完成

---

## 执行顺序

任务A 完全独立，先行修复。任务B 按步骤顺序执行。

---

## 任务A：修复房子不标红视觉路由 bug

### A.1 修改 FarmToolPreview.cs → UpdateHoePreview

- [x] A.1.1 视觉路由分支1条件修改
- [x] A.1.2 Shader 染色条件修改
- [x] A.1.3 验证测试

---

## 任务B：种子从 FIFO 剥离并接入放置系统

### B.1 数据层 + 路由层

- [x] B.1.1 SeedData 标记 isPlaceable
- [x] B.1.2 移除 PlacementManager 的 SeedData 拦截

### B.2 验证层

- [x] B.2.1 新增 ValidateSeedPlacement 方法
- [x] B.2.2 PlacementManager 接入种子验证

### B.3 预览层

- [x] B.3.1 PlacementPreview 增加 SeedData 预览分支

### B.4 执行层

- [x] B.4.1 新增 ExecuteSeedPlacement 方法
- [x] B.4.2 ExecutePlacement 中增加种子分支路由
- [x] B.4.3 HasMoreItems 增加种子袋检测

### B.5 FIFO 清理 — GameInputManager.cs

- [x] B.5.1 删除种子入队/执行方法（TryPlantSeed、ExecutePlantSeed、TryEnqueueSeed）
- [x] B.5.2 删除种子输入路由分支（HandleUseCurrentTool、TryEnqueueFromCurrentInput）
- [x] B.5.3 删除种子 FIFO 处理分支（ProcessNextAction、ExecuteFarmAction）
- [x] B.5.4 删除种子预览中转和缓存字段（UpdateSeedPreview中转、UpdatePreviews种子路由、_cachedSeedData、CancelFarmingNavigation清理、ForceUpdatePreviewToPosition种子分支、ProcessFarmInputAt种子分支）

### B.6 FIFO 清理 — FarmToolPreview.cs

- [x] B.6.1 删除种子预览核心方法（UpdateSeedPreview、GetOrCreateSeedQueueRenderer、UpdateCursorForSeed）
- [x] B.6.2 删除种子队列预览分支（AddQueuePreview、RemoveQueuePreview、ClearAllQueuePreviews、PromoteToExecutingPreview、RemoveExecutingPreview）
- [x] B.6.3 删除种子预览字段（seedPreviewRenderer、seedGridRenderer、seedQueuePool、activeSeedQueuePreviews、executingSeedPreviews、isSeedMode、seedValidColor、seedInvalidColor）

### B.7 编译验证 + 集成测试

- [x] B.7.1 编译验证（0 错误 2 警告，均为已有警告）
- [x] B.7.2 集成测试清单（待手动验证）
