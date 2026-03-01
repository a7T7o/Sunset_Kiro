# 10.1.1 补丁001 - 农田系统交互漏洞修补设计文档

> 创建时间：2026-02-18 会话1续3
> 基于全面交互漏洞分析报告 + 用户 Q1-Q4 确认

---

## 一、用户确认的行为规范

### Q1 锄头长按行为（已确认）
- 长按 = 快速连续输入，只有第一个位置生效并锁定
- 长按期间预览框和耕地预览（1+8 GhostTilemap）都不锁定，跟随鼠标
- 动画完成瞬间锁定当前鼠标位置作为下一个目标
- 不需要缓存状态，只需保持预览跟随
- 耕完后如果仍在长按，获取此时鼠标位置并锁定执行

### Q2 浇水长按行为（已确认）
- 与锄头完全一致的逻辑（含长按），共用逻辑
- 手持判断和动画类型分开

### Q3 种子长按行为（已确认）
- 使用与锄头/浇水一致的逻辑，支持缓存和长按
- 没有动画播放步骤，只需创建（Instantiate）
- 连续点击也支持缓存

### Q4 动画期间切换工具栏（已确认）
- 动画期间绝对禁止任何形式的切换
- 切换必须中断当前动画（最后防线）
- WASD 不能中断动画（lockManager.IsLocked 已阻止）

### 补充约束：所有耕地创建必须播放动画
- 近距离缓存位置不能跳过动画直接变为耕地
- ConsumePendingFarmInput 路径中的近距离分支也必须播放动画
- 当前代码 TryTillSoil 近距离分支已有 RequestAction(Crush)，符合要求

---

## 二、修改方案总览

### 方案 A：CacheFarmInput 预览锁定 + 1+8 刷新
- 目标：缓存输入时锁定预览到缓存位置，正确渲染 1+8 GhostTilemap
- 修改文件：GameInputManager.cs（CacheFarmInput 方法）
- 修改内容：
  1. 获取 ItemData（通过 database.GetItemByID）
  2. UnlockPosition()（先解锁，让 ForceUpdate 能渲染 1+8）
  3. ForceUpdatePreviewToPosition(worldPos, itemData)（刷新完整预览）
  4. 计算 alignedPos/layerIndex/cellPos
  5. LockPosition(alignedPos, cellPos, layerIndex)（锁定）
  6. 缓存字段赋值

### 方案 B：OnActionComplete 长按分支区分农田工具和通用工具
- 目标：农田工具长按不先播动画，让 TryXxx 内部处理；通用工具保持原有行为
- 修改文件：PlayerInteraction.cs（OnActionComplete 方法）
- 修改内容：
  1. 在 shouldContinue 分支中，通过 GameInputManager.IsHoldingFarmTool() 判断
  2. 农田工具分支：StopAnimationTracking → EndAction(true) → isPerformingAction=false → ProcessFarmInputAt(鼠标位置) 或 ConsumePendingFarmInput
  3. 通用工具分支：保持原有 StartAction(actionToRepeat, true) 行为
- 需要 GameInputManager 暴露 IsHoldingFarmTool 为 public

### 方案 D：ConsumePendingFarmInput 先 UnlockPosition
- 目标：消费缓存时先解锁，让 ForceUpdatePreviewToPosition 能正常渲染
- 修改文件：GameInputManager.cs（ConsumePendingFarmInput 方法）
- 修改内容：在 ProcessFarmInputAt 之前调用 farmPreview.UnlockPosition()

### 方案 E：WASD/导航取消时清除缓存
- 目标：WASD 移动和导航取消时清除过时的输入缓存
- 修改文件：GameInputManager.cs（CancelFarmingNavigation + HandleMovement）
- 修改内容：
  1. CancelFarmingNavigation 末尾增加 _hasPendingFarmInput = false
  2. HandleMovement 的 autoNavigator.IsActive 分支增加 _hasPendingFarmInput = false

### 方案 Q4：动画期间切换工具栏中断动画
- 目标：动画期间 hotbar 切换时中断当前动画，丢弃农田缓存
- 修改文件：PlayerInteraction.cs（OnActionComplete 方法）
- 修改内容：在松开分支中，ConsumePendingFarmInput 之前检查 HasCachedHotbarInput，有则丢弃农田缓存
- 备选方案：在 HandleHotbarSelection 中检测到锁定状态下的切换时，设置中断标记

---

## 三、关键时序验证

### 3.1 缓存输入时序（方案 A 修正后）
```
帧 N：用户点击（动画中）
  → HandleUseCurrentTool → _isExecutingFarming 或 isPerformingAction 阻断
  → CacheFarmInput(itemId)
    → UnlockPosition()（_isLocked = false）
    → ForceUpdatePreviewToPosition(worldPos, itemData)
      → UpdateHoePreview(_isLocked=false)
        → UpdateRealtimeData ✅
        → ClearGhostTilemap ✅
        → GetPreviewTiles → SetTile ✅（1+8 渲染到缓存位置）
        → UpdateCursor ✅
    → LockPosition(alignedPos, cellPos, layerIndex)（_isLocked = true）

帧 N+1：UpdatePreviews → UpdateHoePreview
  → UpdateRealtimeData（更新到鼠标位置）
  → if (_isLocked) return;（跳过视觉更新，1+8 保持在缓存位置）✅
```

### 3.2 长按继续时序（方案 B 修正后 — 农田工具）
```
动画结束 → OnActionComplete
  → shouldContinue = true（长按中）
  → IsHoldingFarmTool() = true（锄头/浇水壶/种子）
  → StopAnimationTracking
  → EndAction(true)（清空 hotbar 缓存，保持锁定）
  → isPerformingAction = false
  → 有缓存? ConsumePendingFarmInput() : ProcessFarmInputAt(鼠标位置)
    → ForceUpdatePreviewToPosition → TryTillSoil/TryWaterTile/TryPlantSeed
      → LockPosition → RequestAction → Execute → UnlockPosition（近距离）
      → LockPosition → StartFarmingNavigation（远距离）
```

### 3.3 消费缓存时序（方案 D 修正后）
```
ConsumePendingFarmInput
  → _hasPendingFarmInput = false
  → 验证物品一致性
  → UnlockPosition()（_isLocked = false）← 新增
  → ProcessFarmInputAt(_pendingFarmWorldPos)
    → ForceUpdatePreviewToPosition（_isLocked=false，能正常渲染 1+8）✅
    → TryTillSoil/TryWaterTile/TryPlantSeed
```

---

## 四、不修改的部分（保护清单）

| 组件 | 保护原因 |
|------|---------|
| IsToolAction | 保留 Crush，镐子需要长按连续挖矿 |
| 通用工具长按分支 | 镐子/斧头等保持原有 StartAction 重播行为 |
| UpdateRealtimeData 无条件更新 | 当前行为合理，不影响功能 |
| TryTillSoil/TryWaterTile 近距离分支的 RequestAction | 已有动画播放，符合"所有耕地必须播放动画"约束 |
| lockManager.IsLocked 阻止 WASD | 动画期间 WASD 不能中断，当前行为正确 |
