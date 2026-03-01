# 补丁004V2 任务列表（tasksV3）

> 基于 designV3.md（模块 H/I/J/K）
> 前置条件：V1 代码修改已完成（design.md 模块 A-G）
> 创建时间：2026-02-21

---

## Phase 1：ClearActionQueue 执行状态保护（模块 H）

- [ ] 1.1 修改 `ClearActionQueue` 方法
  - 文件：`GameInputManager.cs`
  - 位置：`ClearActionQueue` 方法（约第2525行）
  - 改动：在清空执行状态前检查 `_isExecutingFarming`
    - `_isExecutingFarming = true` 时：只清空队列和队列预览，保留 `_pendingTileUpdate`、`_currentProcessingRequest`、`_isExecutingFarming`、`_currentHarvestTarget`
    - `_isExecutingFarming = false` 时：全部清空（保持原有行为）
  - 验证：CP-H1、CP-H2、CP-H3、CP-H4

## Phase 2：PromoteToExecutingPreview 时机修正（模块 I）

- [ ] 2.1 从 `ProcessNextAction` 移除 Promote 和 _isExecutingFarming
  - 文件：`GameInputManager.cs`
  - 位置：`ProcessNextAction` 方法中 `_isExecutingFarming = true;` 和 `PromoteToExecutingPreview` 调用
  - 改动：删除这两行
  - 验证：CP-I1、CP-I2

- [ ] 2.2 在 `ExecuteFarmAction` 开头添加 Promote 和 _isExecutingFarming
  - 文件：`GameInputManager.cs`
  - 位置：`ExecuteFarmAction` 方法开头（switch 之前）
  - 改动：添加 `_isExecutingFarming = true;` 和 `FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);`
  - 验证：CP-I3、CP-I4

## Phase 3：canTill=false 红色反馈（模块 J）

- [ ] 3.1 修改 `UpdateHoePreview` 的 else 分支
  - 文件：`FarmToolPreview.cs`
  - 位置：`UpdateHoePreview` 方法中 `if (canTill && ...)` 的 `else` 分支
  - 改动：在 `_currentGhostTileData?.Clear()` 之后，启用 `cursorRenderer` 并调用 `UpdateCursor`
  - 验证：CP-J1、CP-J2、CP-J3

## Phase 4：Sorting Order 确认（模块 K）

- [ ] 4.1 确认 ghostTilemap 的 Sorting Order
  - 文件：`FarmToolPreview.cs`
  - 位置：`EnsureComponents` 方法中 ghostTilemap 的创建/配置
  - 改动：读取代码确认 Sorting Order 设置，如果 ghostTilemap 的 Order 不高于 farmlandBorderTilemap，则调整
  - 验证：CP-K1

## Phase 5：编译验证与正确性审查

- [ ] 5.1 编译验证
  - 使用 getDiagnostics 检查 `FarmToolPreview.cs` 和 `GameInputManager.cs`
  - 目标：0 错误 0 警告

- [ ] 5.2 正确性属性逐项审查
  - 逐条检查 CP-H1~H4、CP-I1~I4、CP-J1~J3、CP-K1
  - 确认每条属性在代码中有对应保证

- [ ] 5.3 创建验收指南V3
  - 文件：`验收指南V3.md`
  - 内容：针对 V2 修复的 4 个问题的验收步骤

- [ ] 5.4 更新 memory
  - 追加续6/续7/续8 记录到子 memory
  - 同步主 memory
