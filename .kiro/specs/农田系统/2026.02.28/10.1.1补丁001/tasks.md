# 10.1.1 补丁001 - 任务列表

> 基于全面交互漏洞分析报告 + 用户 Q1-Q4 确认 + design.md
> 创建时间：2026-02-18 会话1续3

---

## 任务 1：方案 A — CacheFarmInput 预览锁定 + 1+8 刷新

解决漏洞 A-1（1+8 预览时序）、A-2（连续点击预览不更新）、A-3（需要 ItemData）、A-4（需要 layerIndex/cellPos）

- [x] 1.1 修改 CacheFarmInput 方法（GameInputManager.cs）
  - 获取 ItemData：`database.GetItemByID(itemId)`
  - 调用 `farmPreview.UnlockPosition()`（先解锁）
  - 调用 `ForceUpdatePreviewToPosition(worldPos, itemData)`（刷新完整预览含 1+8）
  - 计算 alignedPos/layerIndex/cellPos
  - 调用 `farmPreview.LockPosition(alignedPos, cellPos, layerIndex)`（锁定）
  - 保留原有缓存字段赋值

## 任务 2：方案 B — OnActionComplete 长按分支区分农田工具和通用工具

解决漏洞 B（锄头长按原地重复/远距离卡死）、C-1（浇水长按 RequestAction 被忽略）、C-2（浇水长按远距离卡死）

- [x] 2.1 将 IsHoldingFarmTool 改为 public（GameInputManager.cs）
  - 当前是 private，需要改为 public 供 PlayerInteraction 调用
  - 通过 GameInputManager.Instance.IsHoldingFarmTool() 访问

- [x] 2.2 修改 OnActionComplete 长按分支（PlayerInteraction.cs）
  - 在 shouldContinue 分支内，获取 GameInputManager.Instance
  - 调用 gim.IsHoldingFarmTool() 判断当前手持是否农田工具
  - 农田工具分支：
    - StopAnimationTracking()
    - lockManager.EndAction(true)（清空 hotbar 缓存，保持锁定）
    - isPerformingAction = false
    - 有缓存 → ConsumePendingFarmInput()
    - 无缓存且长按中 → ProcessFarmInputAt(当前鼠标位置)
  - 通用工具分支：保持原有 StartAction(actionToRepeat, true) + 后续逻辑不变

## 任务 3：方案 D — ConsumePendingFarmInput 先 UnlockPosition

解决漏洞 D-1（锁定状态下 ForceUpdatePreviewToPosition 无法渲染 1+8）

- [x] 3.1 修改 ConsumePendingFarmInput 方法（GameInputManager.cs）
  - 在 ProcessFarmInputAt 之前增加 farmPreview.UnlockPosition()
  - 获取 FarmToolPreview.Instance，null 检查后调用

## 任务 4：方案 E — WASD/导航取消时清除缓存

解决漏洞 E-1（WASD 不清除输入缓存）

- [x] 4.1 修改 CancelFarmingNavigation 方法（GameInputManager.cs）
  - 在方法末尾（状态重置之后）增加 `_hasPendingFarmInput = false`

- [x] 4.2 修改 HandleMovement 方法（GameInputManager.cs）
  - 在 autoNavigator.IsActive 分支的 CancelFarmingNavigation() 之后增加 `_hasPendingFarmInput = false`
  - 同时增加 farmPreview.UnlockPosition()（防止 CancelFarmingNavigation 因状态检查跳过解锁）

## 任务 5：方案 Q4 — 动画期间切换工具栏中断处理

解决 Q4（动画期间切换工具栏后缓存消费时序问题）

- [x] 5.1 修改 OnActionComplete 松开分支（PlayerInteraction.cs）
  - 在 ConsumePendingFarmInput 之前检查 lockManager.HasCachedHotbarInput
  - 如果有 hotbar 缓存：跳过 ConsumePendingFarmInput，直接丢弃农田缓存
  - 通过 GameInputManager.Instance.ClearPendingFarmInput() 清除

- [x] 5.2 新增 ClearPendingFarmInput 公共方法（GameInputManager.cs）
  - 公共方法，设置 _hasPendingFarmInput = false
  - 同时调用 farmPreview.UnlockPosition()（清除缓存时解锁预览）

## 任务 6：编译验证与回归检查

- [x] 6.1 编译验证
  - 使用 mcp_mcp_unity_recompile_scripts 验证所有修改无编译错误
  - 检查 getDiagnostics 确认无语法/类型错误

- [x] 6.2 更新分析报告 ✅
  - 在全面交互漏洞分析报告.md 末尾追加"八、Q1-Q4 确认结果与实施记录"章节
  - 记录用户确认的行为规范和实际修改内容

## 任务 7：更新工作区记忆

- [x] 7.1 更新子 memory（10.1.1补丁001/memory.md）
  - 追加会话1续3记录（由 Hook 自动完成 + 续4补充）
  - 记录所有修改的文件和完成的任务

- [x] 7.2 更新主 memory（农田系统/memory.md）
  - 追加 10.1.1补丁001 的摘要记录 + 子工作区索引表更新

- [x] 7.3 创建验收指南
  - 在 10.1.1补丁001/ 下创建验收指南.md
  - 包含功能验收清单和测试步骤
