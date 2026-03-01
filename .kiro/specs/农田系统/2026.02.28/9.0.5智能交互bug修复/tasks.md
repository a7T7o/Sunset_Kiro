# 9.0.5 智能交互Bug修复 - 任务列表

**创建日期**: 2026-02-09
**工作区**: `.kiro/specs/农田系统/9.0.5智能交互bug修复/`

---

## Phase 1: FarmToolPreview 锁定机制 + 视觉数据分离

- [x] 1.1 FarmToolPreview 添加锁定字段和方法
  - 新增 `_isLocked`、`_lockedWorldPosition`、`_lockedCellPos`、`_lockedLayerIndex` 字段
  - 新增 `LockPosition(Vector3 worldPos, Vector3Int cellPos, int layerIndex)` 方法
  - 新增 `UnlockPosition()` 方法
  - 新增 `IsLocked` 公开属性
  - 文件：`FarmToolPreview.cs`

- [x] 1.2 UpdateXxxPreview 实现视觉与数据分离（盲人导航修复）
  - 新增 `UpdateRealtimeData()` 轻量方法：只更新 CurrentCellPos/IsValid/IsInRange/CurrentCursorPos，不触碰 GhostTilemap/CursorRenderer
  - `UpdateHoePreview()` 开头先调用 `UpdateRealtimeData()`，然后 `if (_isLocked) return;` 跳过视觉更新
  - `UpdateWateringPreview()` 同上
  - `UpdateSeedPreview()` 同上
  - 确保锁定状态下实时数据仍然跟随鼠标更新，只有视觉渲染被冻结
  - 文件：`FarmToolPreview.cs`

## Phase 2: FarmNavState 状态机扩展

- [x] 2.1 扩展 FarmNavState 枚举
  - 在现有 `Idle`、`Navigating`、`Executing` 基础上新增 `Preview` 和 `Locked` 状态
  - 最终枚举：`Idle`、`Preview`、`Locked`、`Navigating`、`Executing`
  - 文件：`GameInputManager.cs`

- [x] 2.2 UpdatePreviews 修改为不阻断实时数据更新
  - 移除 `UpdatePreviews()` 中对 `_farmNavState` 的检查（不再在锁定/导航/执行时 return）
  - 让 `UpdatePreviews()` 始终调用 `UpdateFarmToolPreview()`，由 FarmToolPreview 内部的 `_isLocked` 控制视觉冻结
  - 面板打开时：不隐藏预览，不更新预览（面板冻结事件，状态保留）
  - 文件：`GameInputManager.cs`

## Phase 3: 点击锁定与距离分流

- [x] 3.1 TryTillSoil 添加锁定逻辑
  - IsValid 通过后，调用 `FarmToolPreview.LockPosition()` 锁定预览
  - 设置 `_farmNavState = FarmNavState.Locked`
  - 近距离分支：设置 `Executing` → 执行 → `UnlockPosition()` → 回到 `Preview`
  - 远距离分支：`StartFarmingNavigation` → 状态转为 `Navigating`
  - 文件：`GameInputManager.cs`

- [x] 3.2 TryWaterTile 添加锁定逻辑
  - 与 3.1 相同的锁定模式
  - IsValid 通过后锁定预览，距离分流
  - 近距离：`Executing` → 执行 → 解锁 → `Preview`
  - 远距离：`Navigating` → 导航
  - 文件：`GameInputManager.cs`

- [x] 3.3 TryPlantSeed 添加锁定逻辑
  - 与 3.1 相同的锁定模式
  - IsValid 通过后锁定预览，距离分流
  - 近距离：`Executing` → 执行 → 解锁 → `Preview`
  - 远距离：`Navigating` → 导航（含快照校验）
  - 文件：`GameInputManager.cs`

## Phase 4: 中断恢复与状态清理

- [x] 4.1 CancelFarmingNavigation 修改为恢复 Preview 状态 + UnlockPosition 原子性
  - 调用 `FarmToolPreview.UnlockPosition()` 解锁预览
  - 状态重置为 `FarmNavState.Preview`（而非 `Idle`），前提是仍持有农具/种子
  - 如果不持有农具/种子，则重置为 `Idle`
  - 确保无论导航成功/失败/超时/被杀，UnlockPosition 一定被调用（原子性）
  - 文件：`GameInputManager.cs`

- [x] 4.2 导航完成回调中添加解锁逻辑 + 锁定点距离校验（锐评003陷阱一）
  - `WaitForNavigationComplete` 协程中，执行回调后调用 `UnlockPosition()`
  - 状态从 `Executing` 恢复为 `Preview`（而非 `Idle`）
  - 导航失败（距离过远）时也要解锁预览
  - 使用 try/finally 确保 UnlockPosition 一定被调用
  - 🔴 锐评003陷阱一：导航回调中距离校验必须使用 `Vector3.Distance(player, preview.LockedWorldPos)` 而非 `preview.IsInRange`（因为 IsInRange 是鼠标实时距离，不是锁定点距离）
  - 新增 `LockedWorldPos` 公开属性到 FarmToolPreview，供回调中读取锁定点位置
  - 文件：`GameInputManager.cs`

- [x] 4.3 切换物品时正确处理状态
  - `HandleHotbarSelection` 中 `CancelFarmingNavigation` 已存在
  - 确保切换到非农具时状态回到 `Idle`，切换到农具时回到 `Preview`
  - 文件：`GameInputManager.cs`

- [x] 4.4 面板热键不取消导航（用户修正）
  - 移除 `HandlePanelHotkeys` 中所有 `CancelFarmingNavigation()` 调用
  - 面板打开时 `IsAnyPanelOpen()` 返回 true，输入自然被屏蔽
  - 关闭面板后导航状态自然恢复
  - ESC 特殊处理：无面板打开时 ESC 中断导航，有面板打开时 ESC 关闭面板（不取消导航）
  - 文件：`GameInputManager.cs`

- [x] 4.5 导航中重新点击处理（中断+重启）
  - 在 `HandleUseCurrentTool` 中，当 `_farmNavState == Navigating` 时处理新点击
  - 读取 FarmToolPreview 的实时数据（非锁定数据）判断新位置有效性
  - 如果新位置有效：CancelFarmingNavigation → 重新锁定新位置 → 重新开始导航
  - 这是完整的中断+重启，不是"修改目的地"优化
  - 文件：`GameInputManager.cs`

## Phase 5: 执行保护

- [x] 5.1 添加执行保护标志
  - 新增 `_isExecutingFarming` 字段
  - 在 `ExecuteTillSoil`、`ExecuteWaterTile`、`ExecutePlantSeed` 中使用 `try/finally` 保护
  - 执行中不响应新的点击事件
  - 文件：`GameInputManager.cs`

- [x] 5.2 HandleUseCurrentTool 添加执行保护检查
  - 在 `HandleUseCurrentTool` 开头检查 `_isExecutingFarming`，为 true 时直接 return
  - 确保执行中不会重复触发
  - 文件：`GameInputManager.cs`

## Phase 6: 浇水失败原因增强

- [x] 6.1 FarmToolPreview 添加浇水失败原因记录
  - 新增 `WateringFailureReason` 枚举（None/NoFarmland/NotTilled/AlreadyWatered/HasObstacle/ManagerNull）
  - 新增 `LastWateringFailure` 属性
  - 在 `UpdateWateringPreview` 中记录失败原因
  - 文件：`FarmToolPreview.cs`

- [x] 6.2 GameInputManager 浇水失败日志增强
  - `TryWaterTile` 失败时，读取 `FarmToolPreview.LastWateringFailure` 输出具体原因
  - 使用 `showDebugInfo` 开关控制日志输出
  - 文件：`GameInputManager.cs`

## Phase 7: 编译验证与测试

- [x] 7.1 编译验证
  - 确保所有修改后项目编译通过
  - 使用 getDiagnostics 检查 `FarmToolPreview.cs` 和 `GameInputManager.cs`

- [x] 7.2 正确性属性验证
  - P1：预览锁定不变性 — 锁定状态下预览位置不变
  - P2：状态机完整性 — 所有转换在状态转换表中有定义
  - P3：中断恢复一致性 — 中断后 IsLocked=false 且状态为 Idle/Preview
  - P6：预览-状态同步 — IsLocked 与 FarmNavState 始终同步

- [x] 7.3 创建验收指南
  - 包含功能验收清单
  - 测试步骤说明
  - 预期结果描述
  - 已知限制
