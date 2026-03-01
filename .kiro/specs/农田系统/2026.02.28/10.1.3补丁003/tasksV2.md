# 补丁003 任务清单 V2

> 基于 designV2.md 模块 A-I 组织，按依赖关系分阶段执行。
> 所有任务都是必须完成的。

---

## 阶段一：Prefab 结构改造 + 接口暴露（模块 A + B）

> 前置条件：无。这是最基础的结构改造，后续所有模块依赖此阶段。

- [x] 1. 创建 Editor 批量工具：为所有作物 Prefab 套父物体
  - [x] 1.1 编写 Editor 脚本，遍历所有作物 Prefab
  - [x] 1.2 为每个 Prefab 创建空父物体（Crop_xxx_Root），将原有内容移到子物体
  - [x] 1.3 保存 Prefab，验证结构正确
  - [x] 1.4 运行工具，确认所有作物 Prefab 已改造（需用户在 Unity 中执行 Tools/补丁003/作物 Prefab 套父物体）
  - 验收：所有作物 Prefab 为父子结构，CropController + SpriteRenderer 在子物体上
  - 关联：CP-A1, CP-A2

- [x] 2. CropController 接口暴露（模块 B）
  - [x] 2.1 新增 `GetFirstStageSprite()` 公开方法，返回 stages[0] 的 Sprite
  - [x] 2.2 新增 `GetCellCenterPosition()` 公开方法，返回父物体（或自身 parent）的世界坐标
  - [x] 2.3 全局搜索 CropController 的 `transform.position` 引用，替换为 `GetCellCenterPosition()`
  - 验收：外部代码通过新接口获取正确位置和 Sprite
  - 关联：CP-B1, CP-B2

---

## 阶段二：队列逻辑修复 — 延迟执行 + 清理完整性（模块 C + D）

> 前置条件：无（与阶段一可并行，但建议顺序执行）

- [x] 3. ExecuteFarmAction 延迟执行机制（模块 C）
  - [x] 3.1 新增 `_pendingTileUpdate`（`FarmActionRequest?`）和 `_tileUpdateTriggered`（`bool`）字段
  - [x] 3.2 新增动画进度访问路径：PlayerInteraction 添加 `GetAnimationProgress()` 转发方法（调用 animController.GetAnimationProgress()），GameInputManager 通过 playerInteraction 访问
  - [x] 3.3 修改 ExecuteFarmAction：Till/Water 分支不立即执行 tile 更新，改为设置 `_pendingTileUpdate = request` + `_tileUpdateTriggered = false`
  - [x] 3.4 在 Update 中添加动画进度监听：`playerInteraction.GetAnimationProgress() >= 0.5f` 时执行 `_pendingTileUpdate` 中的 tile 更新，设置 `_tileUpdateTriggered = true`
  - [x] 3.5 在 OnFarmActionAnimationComplete 中添加兜底：若 `_pendingTileUpdate != null && !_tileUpdateTriggered` 则强制执行，然后清空 `_pendingTileUpdate = null` + `_tileUpdateTriggered = false`
  - [x] 3.6 每次 ExecuteFarmAction 的 Till/Water 分支调用时重置 `_pendingTileUpdate`（保证连续操作正确）
  - 验收：tile 更新在动画中期触发，不在动画开始瞬间；连续操作 A→B 流程正确
  - 关联：CP-C1, CP-C2, CP-C3

- [x] 4. ClearActionQueue 清理完整性（模块 D）
  - [x] 4.1 在 ClearActionQueue 中添加 `_pendingTileUpdate = null`（或重置标志）
  - [x] 4.2 在 ClearActionQueue 中添加 `FarmToolPreview.Instance?.ClearAllQueuePreviews()`（已注释预留，阶段六任务14实现后启用）
  - 验收：WASD/ESC/切换工具后无残留状态
  - 关联：CP-D1


---

## 阶段三：OnActionComplete 时序修复 + 长按分支（模块 E）

> 前置条件：阶段二（依赖 _pendingTileUpdate 机制和 ClearActionQueue 改动）

- [x] 5. OnActionComplete 松开分支时序修复（模块 E — P3）
  - [x] 5.1 松开分支中将 `isPerformingAction = false` 移到 `OnFarmActionAnimationComplete()` 调用之前
  - [x] 5.2 统一长按/松开分支的清理顺序为：StopTracking → isPerformingAction=false → EndAction → ClearAllCache
  - 验收：连续操作 A→B 时，A 完成后 B 能正常启动（不被 isPerformingAction 守卫拦截）
  - 关联：CP-E1, CP-E3

- [x] 6. OnActionComplete 长按分支改造（模块 E — P2）
  - [x] 6.1 GameInputManager 新增 `IsQueueEmpty()` 公开方法（`return _farmActionQueue.Count == 0`）
  - [x] 6.2 长按分支中检测队列是否为空（调用 gim.IsQueueEmpty()）
  - [x] 6.3 队列空且仍在长按时，调用 gim.TryEnqueueFromCurrentInput() 重新入队当前鼠标位置
  - [x] 6.4 队列非空时走正常 OnFarmActionAnimationComplete → ProcessNextAction 流程
  - 验收：长按锄头/浇水时能连续操作不同位置
  - 关联：CP-E2

- [x] 7. Collect 专用分支保护（模块 E — V2 补充）
  - [x] 7.1 确认 Collect 分支在 shouldContinue 判断之前拦截，不进入 IsToolAction 长按逻辑
  - [x] 7.2 确认通用工具（Slice/Pierce）else 分支完全不变
  - 验收：收获动画完成后正确走队列出队；斧头/镐子长按行为不受影响
  - 关联：CP-E4, CP-E5

---

## 阶段四：导航入队统一（模块 F）

> 前置条件：阶段二（依赖队列基础设施）

- [x] 8. HandleUseCurrentTool 导航入队统一（模块 F）
  - [x] 8.1 简化导航状态分支：导航期间的左键点击统一走 TryEnqueueFromCurrentInput
  - [x] 8.2 确认 HandleUseCurrentTool 入口保持 GetMouseButtonDown 不变（V6 修正）
  - 验收：导航期间点击新位置能正确入队；入口检测方式不变
  - 关联：CP-F1, CP-F2

---

## 阶段五：预览系统前置接口（模块 G）

> 前置条件：无（独立模块）

- [x] 9. FarmVisualManager 水渍 tile 接口暴露（模块 G）
  - [x] 9.1 新增 `GetRandomPuddleTile()` 公开方法，从 wetPuddleTiles 数组随机返回一个 tile
  - [x] 9.2 新增 `GetPuddleTiles()` 公开方法，返回完整 wetPuddleTiles 数组引用
  - 验收：FarmToolPreview 能通过公开方法获取水渍 tile
  - 关联：CP-G1

---

## 阶段六：预览系统全面改造（模块 H）

> 前置条件：阶段一（依赖 GetFirstStageSprite）、阶段五（依赖 GetRandomPuddleTile）

- [x] 10. FarmToolPreview 新增组件和字段（模块 H — 基础设施）
  - [x] 10.1 新增 queuePreviewTilemap（第二个 Tilemap，用于队列预览）
  - [x] 10.2 新增方案C覆盖层：hoeOverlayRenderer + waterOverlayRenderer（SpriteRenderer，白色方块 sprite）
  - [x] 10.3 新增种子预览组件：seedGridRenderer（程序化格子 sprite）+ seedPreviewRenderer（作物 sprite）
  - [x] 10.4 新增 SpriteRenderer 对象池：seedQueuePool + activeSeedQueuePreviews
  - [x] 10.5 新增颜色配置字段：overlayValidColor、overlayInvalidColor、queuePreviewAlpha
  - 验收：所有新组件在 EnsureComponents/Awake 中正确初始化
  - 关联：CP-H1~H6

- [x] 11. 耕地预览改造 — UpdateHoePreview（模块 H）
  - [x] 11.1 隐藏 cursorRenderer（耕地不需要方框光标）
  - [x] 11.2 启用 hoeOverlayRenderer，位置设为格子中心，颜色根据有效性切换
  - [x] 11.3 保留 GhostTilemap 1+8 预览逻辑不变
  - 验收：耕地预览显示颜色覆盖层 + 1+8 边界预览，无方框光标
  - 关联：CP-H1

- [x] 12. 浇水预览改造 — UpdateWateringPreview（模块 H）
  - [x] 12.1 隐藏 cursorRenderer
  - [x] 12.2 启用 waterOverlayRenderer，位置设为格子中心，颜色根据有效性切换
  - [x] 12.3 有效时在 ghostTilemap 上显示随机水渍 tile（调用 FarmVisualManager.GetRandomPuddleTile）
  - 验收：浇水预览显示颜色覆盖层 + 水渍 tile 预览
  - 关联：CP-H2


- [x] 13. 种子预览改造 — UpdateSeedPreview（模块 H）
  - [x] 13.1 隐藏耕地/浇水覆盖层
  - [x] 13.2 启用 seedGridRenderer（程序化格子方框），位置设为格子中心，颜色根据有效性切换（绿/红）
  - [x] 13.3 启用 seedPreviewRenderer，获取 CropController.GetFirstStageSprite() 设置 sprite，颜色根据有效性切换
  - [x] 13.4 隐藏 cursorRenderer（底部格子方框已替代）
  - 验收：种子预览复刻放置系统效果（底部格子 + 作物第一阶段 sprite + 颜色切换）
  - 关联：CP-H3

- [x] 14. 队列预览管理（模块 H）
  - [x] 14.1 实现 AddQueuePreview(cellPos, layerIndex, type)：种子用 SpriteRenderer 对象池，耕地/浇水用 queuePreviewTilemap
  - [x] 14.2 实现 RemoveQueuePreview(cellPos)：回收种子 SpriteRenderer 到对象池，或清除 queuePreviewTilemap 上的 tile
  - [x] 14.3 实现 ClearAllQueuePreviews()：清空 queuePreviewTilemap + 回收所有种子 SpriteRenderer
  - [x] 14.4 确认 ClearGhostTilemap 只清除 ghostTilemap，不影响 queuePreviewTilemap
  - 验收：队列预览正确显示/移除/清空；ClearGhostTilemap 不误清队列预览
  - 关联：CP-H4, CP-H5, CP-H6

- [x] 15. Hide 方法扩展（模块 H）
  - [x] 15.1 Hide 中隐藏所有新增覆盖层和种子预览组件
  - [x] 15.2 确认 Hide 不清空队列预览（Hide 只隐藏鼠标跟随预览）
  - 验收：Hide 后覆盖层和种子预览不可见，队列预览保留
  - 关联：CP-H1~H3

---

## 阶段七：队列预览联动（模块 I）

> 前置条件：阶段六（依赖 AddQueuePreview/RemoveQueuePreview/ClearAllQueuePreviews）

- [x] 16. GameInputManager 队列预览联动（模块 I）
  - [x] 16.1 EnqueueAction 中入队成功后调用 `FarmToolPreview.Instance?.AddQueuePreview(...)`
  - [x] 16.2 OnFarmActionAnimationComplete 中移除队列预览：`FarmToolPreview.Instance?.RemoveQueuePreview(...)`
  - [x] 16.3 OnCollectAnimationComplete 中移除队列预览：`FarmToolPreview.Instance?.RemoveQueuePreview(...)`
  - [x] 16.4 确认 ClearActionQueue 中已包含 ClearAllQueuePreviews（阶段二任务4已处理）
  - 验收：入队时显示队列预览，执行完消失，中断时全部清空
  - 关联：CP-I1, CP-I2, CP-I3

---

## 阶段八：集成验证

- [x] 17. 编译验证
  - [x] 17.1 getDiagnostics 检查所有修改文件，0 错误 0 警告
  - [x] 17.2 确认无 Unity 6 弃用 API 使用

- [x] 18. 正确性属性逐项审查
  - [x] 18.1 逐条检查 CP-A1~CP-I3（25 条属性）在代码中的实现
  - [x] 18.2 创建验收指南文档，按交互矩阵（designV2 第十章）组织测试场景

