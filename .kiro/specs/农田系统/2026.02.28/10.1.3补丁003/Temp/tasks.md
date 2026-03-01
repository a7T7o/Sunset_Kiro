# 补丁003 任务清单

## 阶段一：P1/P5 作物位置修复

- [ ] 1. CropController 视觉子物体架构改造
  - [ ] 1.1 移除 `RequireComponent(typeof(SpriteRenderer))`
  - [ ] 1.2 `Awake` 中检测 SpriteRenderer 位置，如在自身则创建 Visual 子物体并迁移
  - [ ] 1.3 `spriteRenderer` 字段指向子物体的 SpriteRenderer
  - [ ] 1.4 验证 `AlignSpriteBottom` 现在只修改子物体 localPosition，不影响 GameObject 世界坐标
  - [ ] 1.5 验证存档加载（`DynamicObjectFactory.TryReconstructCrop`）兼容性
  - [ ] 1.6 验证种植流程：`ExecutePlantSeed` → `Instantiate` → `Initialize` → 作物位置 = 格子中心

## 阶段二：P3 队列逻辑修复 + 动画帧触发

- [ ] 2. 松开分支时序修复
  - [ ] 2.1 `PlayerInteraction.OnActionComplete` 松开分支：`isPerformingAction = false` 移到 `OnFarmActionAnimationComplete()` 之前
  - [ ] 2.2 验证：A动画完成后松开鼠标，B能正常播放动画

- [ ] 3. 动画帧触发 tile 更新机制
  - [ ] 3.1 `GameInputManager` 新增 `_pendingTileUpdate` 字段（记录待执行的 tile 更新请求）
  - [ ] 3.2 `GameInputManager` 新增 `tileUpdateTriggerProgress` 字段（默认0.5f，Inspector可调）
  - [ ] 3.3 `ExecuteFarmAction` Till 分支：移除 `ExecuteTillSoil` 同步调用，改为记录到 `_pendingTileUpdate`
  - [ ] 3.4 `ExecuteFarmAction` Water 分支：移除 `ExecuteWaterTile` 同步调用，改为记录到 `_pendingTileUpdate`
  - [ ] 3.5 `Update` 中新增延迟执行监听：当动画进度 >= `tileUpdateTriggerProgress` 时执行待处理的 tile 更新
  - [ ] 3.6 验证：耕地动画播放到约50%进度时 tile 才变化
  - [ ] 3.7 验证：连续耕地 A→B→C 流程正确（A第四帧触发→A完成→改朝向B→B动画→B第四帧触发）

## 阶段三：P2 长按退化修复

- [ ] 4. HandleUseCurrentTool 长按支持
  - [ ] 4.1 入口检测从 `GetMouseButtonDown(0)` 改为 `GetMouseButton(0)`
  - [ ] 4.2 增加防重复入队逻辑：当前鼠标位置已在 `_queuedPositions` 中则跳过
  - [ ] 4.3 区分首次按下和持续按住：首次按下正常入队，持续按住检查位置变化
  - [ ] 4.4 验证：长按左键能持续触发耕地/浇水

- [ ] 5. 长按连续执行（同一位置）
  - [ ] 5.1 `OnActionComplete` 长按分支：动画完成后如果队列为空且仍在长按，重新检测当前位置并入队
  - [ ] 5.2 验证：长按同一位置能连续执行（如连续浇水同一格）

## 阶段四：P4 导航入队统一

- [ ] 6. HandleUseCurrentTool 导航分支简化
  - [ ] 6.1 导航中/执行中的左键点击统一走 `TryEnqueueFromCurrentInput`
  - [ ] 6.2 移除导航中断重新开始的逻辑
  - [ ] 6.3 验证：导航期间点击新位置能正确入队

## 阶段五：P6 预览系统改造

- [ ] 7. 耕地预览改造：颜色叠加替代光标方框
  - [ ] 7.1 `UpdateHoePreview` 中去除光标方框渲染
  - [ ] 7.2 GhostTilemap 上的耕地预览 tile 增加颜色叠加（可交互=绿色半透明，不可交互=红色半透明）
  - [ ] 7.3 验证：鼠标移动时耕地预览显示正确的颜色叠加

- [ ] 8. 浇水预览改造：显示浇水图案
  - [ ] 8.1 `UpdateWateringPreview` 中获取水渍 Tile 并在 GhostTilemap 上预览
  - [ ] 8.2 可交互位置显示水渍图案 + 绿色叠加，不可交互显示红色叠加
  - [ ] 8.3 去除浇水模式的光标方框
  - [ ] 8.4 验证：浇水预览显示正确的水渍图案

- [ ] 9. 种子预览改造：显示作物sprite
  - [ ] 9.1 新增 `_seedPreviewSpriteRenderer` 字段（临时 SpriteRenderer）
  - [ ] 9.2 `UpdateSeedPreview` 中从 `SeedData.cropPrefab` 获取第一阶段 sprite 并显示
  - [ ] 9.3 可种植位置显示作物原色sprite，不可种植显示红色tint
  - [ ] 9.4 保留种子模式的光标框（用户要求）
  - [ ] 9.5 验证：种子预览显示正确的作物图案

- [ ] 10. 收获预览：去除光标
  - [ ] 10.1 收获模式不显示任何光标或预览
  - [ ] 10.2 验证：收获时无光标显示

- [ ] 11. 多位置队列预览
  - [ ] 11.1 `FarmToolPreview` 新增 `_queuedPreviewPositions` 列表
  - [ ] 11.2 新增 `AddQueuePreview(cellPos, layerIndex)` 方法：在指定位置显示无颜色叠加的半透明预览
  - [ ] 11.3 新增 `RemoveQueuePreview(cellPos)` 方法：移除指定位置的队列预览
  - [ ] 11.4 新增 `ClearAllQueuePreviews()` 方法：清空所有队列预览
  - [ ] 11.5 `GameInputManager` 入队成功时调用 `AddQueuePreview`
  - [ ] 11.6 `GameInputManager` 执行完成时调用 `RemoveQueuePreview`
  - [ ] 11.7 `GameInputManager` 清空队列时调用 `ClearAllQueuePreviews`
  - [ ] 11.8 验证：连续点击多个位置，每个位置都显示队列预览
  - [ ] 11.9 验证：执行完成后对应位置预览消失
  - [ ] 11.10 验证：WASD/切换工具/ESC 清空所有预览

## 阶段六：集成验证

- [ ] 12. 全流程集成测试
  - [ ] 12.1 验证：种植完整流程（选种子→预览→点击→作物出现在格子中心）
  - [ ] 12.2 验证：连续耕地流程（长按/连续点击，动画第四帧触发tile变化）
  - [ ] 12.3 验证：连续浇水流程（长按/连续点击，预览显示水渍图案）
  - [ ] 12.4 验证：队列中断恢复（WASD中断→队列清空→预览清除）
  - [ ] 12.5 验证：存档加载后作物位置正确

- [ ] 13. 更新 memory.md
