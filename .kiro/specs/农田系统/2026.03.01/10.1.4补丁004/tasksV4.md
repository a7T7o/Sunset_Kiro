# 补丁004V2 任务列表（tasksV4）

> 基于 designV4.md（模块 H/I/J/K/L）
> 前置条件：V1 代码修改已完成（design.md 模块 A-G）
> 创建时间：2026-02-21

---

## Phase 1：ClearActionQueue 执行状态保护（模块 H）

- [x] 1.1 修改 `ClearActionQueue` 方法
  - 文件：`GameInputManager.cs`
  - 方法：`ClearActionQueue`（第2531行）
  - 改动：在清空执行状态前检查 `_isExecutingFarming`
    - `_isExecutingFarming == true` 时：只清空 `_farmActionQueue`、`_queuedPositions`、`_isProcessingQueue`，保留 `_pendingTileUpdate`、`_currentProcessingRequest`、`_isExecutingFarming`、`_tileUpdateTriggered`、`_currentHarvestTarget`
    - `_isExecutingFarming == false` 时：全部清空（保持原有行为）
  - 验证：CP-H1、CP-H2、CP-H3、CP-H4

## Phase 2：PromoteToExecutingPreview 时机修正（模块 I）

- [x] 2.1 从 `ProcessNextAction` 移除 Promote 和 _isExecutingFarming
  - 文件：`GameInputManager.cs`
  - 方法：`ProcessNextAction`（第2319行）
  - 改动：删除第2380行 `_isExecutingFarming = true;` 和第2383行 `FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);`
  - 验证：CP-I1、CP-I2

- [x] 2.2 在 `ExecuteFarmAction` 开头添加 Promote 和 _isExecutingFarming
  - 文件：`GameInputManager.cs`
  - 方法：`ExecuteFarmAction`（第2425行）
  - 改动：在方法开头（switch 之前）添加 `_isExecutingFarming = true;` 和 `FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);`
  - 注意：PlantSeed 分支已有 `_isExecutingFarming = false` 和 `RemoveExecutingPreview`，时序正确
  - 验证：CP-I3、CP-I4

## Phase 3：增量差集计算基础设施（模块 L — FarmlandBorderManager）

- [x] 3.1 新增 `ParseDirections` 方法
  - 文件：`FarmlandBorderManager.cs`
  - 位置：在 `SelectBorderTile` 方法之后
  - 签名：`public (bool hasU, bool hasD, bool hasL, bool hasR) ParseDirections(TileBase tile)`
  - 逻辑：逐一比较 tile 引用与 16 个边界 tile 字段，返回方向四元组
  - 验证：CP-L5

- [x] 3.2 新增 `IsBorderTile` 和 `IsShadowTile` 方法
  - 文件：`FarmlandBorderManager.cs`
  - 位置：在 `ParseDirections` 之后
  - `IsBorderTile`：调用 ParseDirections 判断是否有任何方向为 true
  - `IsShadowTile`：比较 tile 引用与 4 个阴影 tile 字段（shadowLU/shadowRU/shadowLD/shadowRD）
  - 验证：CP-L1、CP-L3

- [x] 3.3 `SelectBorderTile` 改为 public
  - 文件：`FarmlandBorderManager.cs`
  - 改动：`private TileBase SelectBorderTile(...)` → `public TileBase SelectBorderTile(...)`
  - 验证：CP-L2（增量差集需要从差集方向生成增量 tile）

## Phase 4：UpdateHoePreview 增量差集过滤改造（模块 L — FarmToolPreview）

- [x] 4.1 改造差异化过滤循环
  - 文件：`FarmToolPreview.cs`
  - 方法：`UpdateHoePreview`（第487-501行附近的 foreach 循环）
  - 改动：将 `if (kvp.Value == actualTile) continue; ghostTilemap.SetTile(kvp.Key, kvp.Value);` 替换为增量差集计算逻辑
    - actualTile == null → 直接显示预览 tile（全新位置）
    - actualTile 是阴影 tile → 直接显示预览 tile（Sorting Order 覆盖）
    - 两者都是边界 tile → ParseDirections 计算差集方向 → SelectBorderTile 生成增量 tile → 显示增量 tile
    - 其他情况 → 直接显示预览 tile
  - 注意：`_currentGhostTileData` 缓存的也应该是增量 tile（而非最终 tile），确保入队时队列预览显示增量
  - 验证：CP-L1、CP-L2、CP-L3、CP-L4、CP-L6

## Phase 5：canTill=false 红色反馈（模块 J）

- [x] 5.1 修改 `UpdateHoePreview` 的 else 分支
  - 文件：`FarmToolPreview.cs`
  - 方法：`UpdateHoePreview`（else 分支，约第503行）
  - 改动：在 `_currentGhostTileData?.Clear()` 之后，添加 `cursorRenderer.enabled = true;` 和 `UpdateCursor(layerIndex, cellPos);`
  - 逻辑：UpdateCursor 根据 currentState 设置颜色 — Invalid=红色（已有耕地），Valid=绿色（枯萎作物）
  - 验证：CP-J1、CP-J2、CP-J3

## Phase 6：编译验证与正确性审查

- [x] 6.1 编译验证
  - 使用 getDiagnostics 检查 `FarmToolPreview.cs`、`GameInputManager.cs`、`FarmlandBorderManager.cs`
  - 目标：0 错误 0 警告

- [x] 6.2 Sorting Order 确认（模块 K）
  - 读取 `EnsureComponents` 确认 ghostTilemap sortingOrder = 9999
  - 确认 > farmlandBorderTilemap 的 Sorting Order
  - 验证：CP-K1

- [x] 6.3 正确性属性逐项审查
  - 逐条检查 CP-H1~H4、CP-I1~I4、CP-L1~L6、CP-J1~J3、CP-K1（共18条）
  - 确认每条属性在代码中有对应保证

- [x] 6.4 创建验收指南V4
  - 文件：`验收指南V4.md`
  - 内容：针对 P1~P4 + P6 的验收步骤

- [x] 6.5 更新 memory
  - 追加会话2续5记录到子 memory
  - 同步主 memory

