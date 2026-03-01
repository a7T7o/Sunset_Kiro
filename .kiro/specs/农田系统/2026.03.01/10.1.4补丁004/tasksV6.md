# 补丁004 任务清单 V6 — 农田预览系统最终修复

> 来源：`designV6.md`（9个模块，其中Q/V/N'/O''已实施）
> 只包含尚未实施的改动
> 执行顺序：S执行层 → T浇水随机 → U浇水b层 → P'水渍 → P''播种b层 → 验收

---

## 阶段一：模块S执行层补全

- [x] 1. ExecuteFarmAction 新增 RemoveCrop 分支
  - [x] 1.1 在 `GameInputManager.ExecuteFarmAction` 的 switch 中新增 `case FarmActionType.RemoveCrop:`，使用 Crush 动画 + `_pendingTileUpdate` 延迟执行
  - [x] 1.2 在 `GameInputManager.Update` 的 `_pendingTileUpdate` 监听 switch 中新增 `case FarmActionType.RemoveCrop:` 调用 `ExecuteRemoveCrop`
  - [x] 1.3 在 `GameInputManager.OnFarmActionAnimationComplete` 的兜底 switch 中新增 `case FarmActionType.RemoveCrop:` 调用 `ExecuteRemoveCrop`
  - [x] 1.4 新增 `private bool ExecuteRemoveCrop(int layerIndex, Vector3Int cellPos)` 方法：获取 tileData.cropController 并调用 RemoveCrop()
  - [x] 1.5 验证：getDiagnostics 编译通过

- [x] 2. TryEnqueueFarmTool 入队类型判断
  - [x] 2.1 修改 `TryEnqueueFarmTool` 中 Hoe 的入队类型：`farmPreview.HasCrop ? FarmActionType.RemoveCrop : FarmActionType.Till`
  - [x] 2.2 验证：getDiagnostics 编译通过

## 阶段二：模块T浇水随机重写

- [x] 3. UpdateWateringPreview 随机逻辑重写
  - [x] 3.1 在 `FarmToolPreview` 新增字段：`bool _needsNewPuddleVariant`、`bool _wateringModeInitialized`
  - [x] 3.2 在 `UpdateWateringPreview` 开头新增切换检测：`!_wateringModeInitialized` 时随机默认样式并设为 true
  - [x] 3.3 移除"进入新格子就随机"逻辑（删除 `if (cellPos != _lastWateringCellPos)` 随机块）
  - [x] 3.4 新增"浇水后移出才随机"逻辑：`_needsNewPuddleVariant && cellPos != _lastWateringCellPos` 时触发随机并重置标志
  - [x] 3.5 在 `UpdateHoePreview` 和 `UpdateSeedPreview` 开头添加 `_wateringModeInitialized = false;`
  - [x] 3.6 在 `GameInputManager.ExecuteWaterTile` 成功后设置 `FarmToolPreview.Instance` 的 `_needsNewPuddleVariant = true` 和更新 `_lastWateringCellPos`（需新增 public 方法或属性）
  - [x] 3.7 验证：getDiagnostics 编译通过

## 阶段三：模块U浇水b层 + 模块P'水渍移出isValid

- [x] 4. UpdateWateringPreview b层拦截 + 水渍移出isValid
  - [x] 4.1 在 `UpdateWateringPreview` 的 canWater 计算后新增：`if (canWater && queuePreviewPositions.Contains(cellPos)) canWater = false;`
  - [x] 4.2 将水渍tile放置逻辑从 `if (isValid)` 内部移出，改为无论 isValid 都放置水渍tile（作为shader载体）
  - [x] 4.3 随机逻辑（模块T已改造）也移出 isValid 保护圈
  - [x] 4.4 验证：getDiagnostics 编译通过

## 阶段四：模块P''播种b层拦截

- [x] 5. UpdateSeedPreview b层拦截
  - [x] 5.1 在 `UpdateSeedPreview` 的 canPlant 计算后新增：`if (canPlant && queuePreviewPositions.Contains(cellPos)) canPlant = false;`
  - [x] 5.2 验证：getDiagnostics 编译通过

## 阶段五：验收

- [x] 6. 编译验证 + 正确性属性审查
  - [x] 6.1 对所有修改文件执行 getDiagnostics，确认 0 错误 0 警告
  - [x] 6.2 逐条检查 CP-S1~S6、CP-T1~T5、CP-U1、CP-P1~P3 在代码中的对应实现
  - [x] 6.3 创建验收指南V6
  - [x] 6.4 更新 memory + 主 memory
