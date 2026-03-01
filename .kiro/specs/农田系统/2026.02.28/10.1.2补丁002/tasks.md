# 10.1.1 补丁002 — 任务列表 V3

> 基于 design.md V3（按修改目标组织）。每个任务自包含执行所需的全部信息，不需要回翻 design。
> 执行顺序：独立模块优先（H/I），然后队列基础设施（A），再入队入口（B），执行引擎（C），分支改造（G），中断/过滤/暂停（D/E/F），最后集成验证。

---

## Phase 1：独立模块 — 无依赖，可先行

### 任务 1.1：FarmToolPreview.LockPosition 渲染 1+8（模块 H / P3 修复）

- [x] 1.1 修改 `FarmToolPreview.LockPosition()` 方法，在锁定时渲染 GhostTilemap
  - 文件：`FarmToolPreview.cs`，方法 `LockPosition`（约第 294 行）
  - 现状：只调用 `UpdateCursor(layerIndex, cellPos)`，不渲染 GhostTilemap
  - 改动：在 `UpdateCursor` 之前，增加以下逻辑：
    1. 检查 `isHoeMode && FarmlandBorderManager.Instance != null`（只有锄头模式才渲染 1+8）
    2. `ClearGhostTilemap()` 清除旧预览
    3. 检查 `FarmTileManager.Instance.CanTillAt(layerIndex, cellPos)` 是否可锄地
    4. 如果不可锄地，额外检查枯萎作物清除（与 `UpdateHoePreview` 中的 `canClearWithered` 逻辑一致）
    5. 如果 `canTill`：调用 `FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos)` 获取 1+8 Tile
    6. 遍历 `previewTiles`，`ghostTilemap.SetTile(kvp.Key, kvp.Value)` + `currentPreviewPositions.Add(kvp.Key)`
  - `UnlockPosition` 不需要修改（解锁后下一帧 `UpdateHoePreview` 自动恢复鼠标跟随）
  - 验收标准：远距离点击后，1+8 预览立即刷新到锁定位置（CP-15）
  - 回归测试：近距离锄地预览仍正常、浇水模式不显示 1+8

### 任务 1.2：CropController 作物底部对齐（模块 I / P5 修复）

- [x] 1.2 在 `CropController` 中新增 `AlignSpriteBottom()` 并在 `UpdateVisuals()` 末尾调用
  - 文件：`CropController.cs`
  - 新增方法 `AlignSpriteBottom()`：
    ```
    if (spriteRenderer == null || spriteRenderer.sprite == null) return
    var bounds = spriteRenderer.sprite.bounds
    var localPos = spriteRenderer.transform.localPosition
    localPos.y = -bounds.min.y
    spriteRenderer.transform.localPosition = localPos
    ```
    参考：`TreeController.AlignSpriteBottom()` 的实现
  - 在 `UpdateVisuals()` 末尾（颜色设置之后）调用 `AlignSpriteBottom()`
  - 验收标准：每个生长阶段的作物底部中心对齐格子中心（CP-16）
  - 影响范围：Initialize/Grow/SetWithered/ResetForReHarvest/Load 都通过 UpdateVisuals 间接调用

---

## Phase 2：队列基础设施 — 数据结构与核心方法（模块 A + C）

### 任务 2.1：新增队列数据结构

- [x] 2.1 在 `GameInputManager` 中新增枚举、结构体和字段
  - 文件：`GameInputManager.cs`
  - 新增枚举 `FarmActionType`：`Till, Water, PlantSeed, Harvest`
  - 新增结构体 `FarmActionRequest`：
    - `FarmActionType type` — 操作类型
    - `Vector3Int cellPos` — 目标格子坐标
    - `int layerIndex` — 目标层级索引
    - `Vector3 worldPos` — 目标世界坐标（格子中心）
    - `CropController targetCrop` — 仅 Harvest 使用，其他为 null
  - 新增字段：
    - `Queue<FarmActionRequest> _farmActionQueue = new()`
    - `HashSet<(int, Vector3Int)> _queuedPositions = new()`
    - `bool _isProcessingQueue = false`
    - `bool _isQueuePaused = false`
    - `CropController _currentHarvestTarget = null`
    - `FarmActionRequest _currentProcessingRequest`
  - 验收标准：编译通过，无错误

### 任务 2.2：实现 EnqueueAction 入队方法

- [x] 2.2 实现 `EnqueueAction(FarmActionRequest request)` 方法
  - 文件：`GameInputManager.cs`
  - 逻辑：
    1. 防重复：`_queuedPositions.Contains((request.layerIndex, request.cellPos))` → return
    2. Harvest 额外防重复：遍历队列检查同一 `targetCrop` 实例 → return
    3. `_queuedPositions.Add(key)` + `_farmActionQueue.Enqueue(request)`
    4. 如果 `!_isProcessingQueue && !_isQueuePaused` → `ProcessNextAction()`
  - 验收标准：连续点击同一格子只入队一次（CP-2）

### 任务 2.3：实现 ProcessNextAction 出队执行方法

- [x] 2.3 实现 `ProcessNextAction()` 方法
  - 文件：`GameInputManager.cs`
  - 逻辑：
    1. `if (_isQueuePaused) return`
    2. 队列为空 → `_isProcessingQueue = false` + `_isExecutingFarming = false` + `_queuedPositions.Clear()` + `UnlockPosition()` + `_farmNavState = Preview` → return
    3. `_isProcessingQueue = true` + Dequeue + 存入 `_currentProcessingRequest`
    4. 二次验证：
       - PlantSeed → `HasSeedRemaining()` 失败则移除 queuedPositions 并递归 ProcessNextAction
       - Harvest → `targetCrop == null || !CanInteract(null)` 失败则同上
    5. 距离判断：`Vector2.Distance(playerCollider.bounds.center, request.worldPos)`（🔴 玩家位置 = Collider 中心）
    6. `_isExecutingFarming = true`
    7. `LockPosition(request.worldPos, request.cellPos, request.layerIndex)`
    8. 近距离（<= farmToolReach）→ `_farmNavState = Executing` → `ExecuteFarmAction(request)`
    9. 远距离 → `_farmNavState = Locked` → `StartFarmingNavigation(request.worldPos, callback)`
       - 导航到达回调中：检查 `_isQueuePaused` → `_farmNavState = Executing` → `ExecuteFarmAction(request)`
  - 验收标准：队列按 FIFO 顺序执行（CP-1），混合近/远距离操作正确处理

### 任务 2.4：实现 ExecuteFarmAction 执行分发方法

- [x] 2.4 实现 `ExecuteFarmAction(FarmActionRequest request)` 方法
  - 文件：`GameInputManager.cs`
  - 逻辑（面向目标后按类型分发）：
    - Till → `FaceTarget(request.worldPos)` → `RequestAction(Crush)` → `ExecuteTillSoil(layerIndex, cellPos)`
      - 动画完成后由 OnActionComplete → OnFarmActionAnimationComplete 回调
    - Water → `FaceTarget(request.worldPos)` → `RequestAction(Watering)` → `ExecuteWaterTile(layerIndex, cellPos)`
      - 动画完成后同上
    - PlantSeed → 获取当前 SeedData → `ExecutePlantSeed(seedData, layerIndex, cellPos)` → `_isExecutingFarming = false` → 移除 queuedPositions → `ProcessNextAction()`（无动画，立即下一个）
    - Harvest → `_currentHarvestTarget = request.targetCrop` → `FaceTarget` → `RequestAction(Collect)`
      - 动画完成后由 OnActionComplete → OnCollectAnimationComplete 回调
  - 注意：`FaceTarget` 方法需要确认现有代码中是否已有，或需要新增（根据 worldPos 设置玩家朝向）
  - 验收标准：四种操作类型都能正确分发执行

### 任务 2.5：实现回调方法和清空方法

- [x] 2.5 实现 `OnFarmActionAnimationComplete()`、`OnCollectAnimationComplete()`、`ClearActionQueue()`、`HasSeedRemaining()`
  - 文件：`GameInputManager.cs`
  - `OnFarmActionAnimationComplete()`（public，供 PlayerInteraction 调用）：
    - `_isExecutingFarming = false` → 移除 `_currentProcessingRequest` 的 queuedPositions → `ProcessNextAction()`
  - `OnCollectAnimationComplete()`（public，供 PlayerInteraction 调用）：
    - 如果 `_currentHarvestTarget != null && CanInteract(null)` → `BuildInteractionContext()` → `OnInteract(context)`
    - `_currentHarvestTarget = null` → `_isExecutingFarming = false` → 移除 queuedPositions → `ProcessNextAction()`
  - `ClearActionQueue()`（public）：
    - `_farmActionQueue.Clear()` + `_queuedPositions.Clear()` + `_isProcessingQueue = false` + `_isExecutingFarming = false` + `_currentHarvestTarget = null` + `_currentProcessingRequest = default`
  - `HasSeedRemaining()`（private）：
    - 获取当前 hotbar slot → `slot.IsEmpty` → false；`itemData is not SeedData` → false；`slot.amount > 0` → true
  - 验收标准：编译通过，回调方法签名正确（CP-7/CP-10）

---

## Phase 3：入队入口改造 — HandleUseCurrentTool（模块 B）

### 任务 3.1：收获检测方法 TryDetectAndEnqueueHarvest

- [x] 3.1 新增 `TryDetectAndEnqueueHarvest()` 方法
  - 文件：`GameInputManager.cs`
  - 逻辑：
    1. `GetMouseWorldPosition()` 获取鼠标世界坐标
    2. `Physics2D.OverlapPointAll(mouseWorldPos)` 获取所有碰撞体
    3. 获取玩家当前层级索引（需确认 API：可能是 `FarmTileManager.GetCurrentLayerIndex(playerPos)` 或 `PlacementLayerDetector`）
    4. 遍历碰撞体：
       - 跳过玩家自身
       - `GetComponent<IInteractable>()` 或 `GetComponentInParent<IInteractable>()`
       - 判断 `interactable is CropController crop`
       - 层级过滤：`crop.layerIndex != playerLayer` → continue（CP-6）
       - 可收获检测：`!crop.CanInteract(null)` → continue
       - 构建 `FarmActionRequest { type = Harvest, cellPos = crop.CellPos, layerIndex = crop.layerIndex, worldPos = crop.transform.position, targetCrop = crop }`
       - `EnqueueAction(request)` → return true
    5. 无可收获作物 → return false
  - 验收标准：成熟作物左键收获（CP-5），不同层级作物不响应（CP-6）

### 任务 3.2：农田工具和种子入队方法

- [x] 3.2 新增 `TryEnqueueFarmTool(ToolData)`、`TryEnqueueSeed(SeedData)`、`TryEnqueueFromCurrentInput()` 方法
  - 文件：`GameInputManager.cs`
  - `TryEnqueueFarmTool(ToolData tool)`：
    - 检查 `FarmToolPreview.Instance?.IsValid()` → 无效则 return（CP-9）
    - 根据 `tool.toolType` 确定 `FarmActionType`（Hoe → Till，WateringCan → Water）
    - 从 FarmToolPreview 获取 `CurrentCellPos`/`CurrentLayerIndex`/`CurrentCursorPos`
    - `EnqueueAction(request)`
  - `TryEnqueueSeed(SeedData seedData)`：
    - 检查 `FarmToolPreview.Instance?.IsValid()` → 无效则 return（CP-9）
    - 构建 `FarmActionRequest { type = PlantSeed, ... }`
    - `EnqueueAction(request)`
  - `TryEnqueueFromCurrentInput()`：
    - 先调用 `TryDetectAndEnqueueHarvest()` → 成功则 return
    - 获取当前手持物品 → ToolData(Hoe/WateringCan) → `TryEnqueueFarmTool`
    - SeedData → `TryEnqueueSeed`
  - 验收标准：编译通过，预览无效时不入队（CP-9）

### 任务 3.3：HandleUseCurrentTool 全面改造

- [x] 3.3 重写 `HandleUseCurrentTool()` 的左键点击逻辑
  - 文件：`GameInputManager.cs`，方法 `HandleUseCurrentTool`（约第 617 行）
  - 改动要点（按代码顺序）：
    1. 保护分支1（`_isExecutingFarming`）：删除原有 `CacheFarmInput` 调用，改为 `TryEnqueueFromCurrentInput()`（V1 修补）
    2. 保护分支2（`isPerformingAction`）：同上，删除 `CacheFarmInput`，改为 `TryEnqueueFromCurrentInput()`（V1 修补）
    3. 导航中重新点击：如果 `_isProcessingQueue` 为 true，走 `TryEnqueueFromCurrentInput()` + return
    4. 在工具分发之前，插入收获检测：`if (TryDetectAndEnqueueHarvest()) return`（CP-5）
    5. Hoe/WateringCan 分支：`TryHandleFarmingTool(tool)` → 改为 `TryEnqueueFarmTool(tool)` + return
    6. SeedData 分支：`TryPlantSeed(seedData)` → 改为 `TryEnqueueSeed(seedData)`
    7. 其他工具（镐子/斧头/武器）：保持原有逻辑完全不变（CP-19）
  - 🔴 关键：两个保护分支中的 `CacheFarmInput` 必须全部替换，不能遗漏（V1 漏洞的核心修复点）
  - 验收标准：
    - 动画期间点击走入队（CP-11）
    - 收获优先级最高（CP-5）
    - 预览无效不入队（CP-9）
    - 镐子/斧头长按行为不变（CP-19）

---

## Phase 4：OnActionComplete 分支改造（模块 G）

### 任务 4.1：OnActionComplete Collect 专用分支

- [x] 4.1 在 `PlayerInteraction.OnActionComplete()` 中新增 Collect 专用分支
  - 文件：`PlayerInteraction.cs`，方法 `OnActionComplete`（约第 180 行）
  - 改动位置：在 `ApplyCachedDirectionToFacing()` 之前（方法最开头的 Collect/Death 状态标记之后）
  - 新增代码逻辑：
    ```
    if (currentAction == AnimState.Collect)
    {
      animController?.StopAnimationTracking()
      lockManager?.EndAction(false)
      lockManager?.ClearAllCache()
      isPerformingAction = false
      GameInputManager.Instance?.OnCollectAnimationComplete()
      return  // 不进入后续任何分支
    }
    ```
  - 🔴 必须在 `shouldContinue` 判断之前拦截，因为 `IsToolAction(Collect)` 返回 false
  - 移除原有的 `if (currentAction == Collect) isCarrying = true`（Collect 不再设置 isCarrying，收获不是搬运）
  - 验收标准：Collect 动画完成后执行收获并取队列下一个（CP-17）

### 任务 4.2：OnActionComplete 农田工具分支改为队列出队

- [x] 4.2 修改 `OnActionComplete()` 的农田工具长按分支和松开分支
  - 文件：`PlayerInteraction.cs`，方法 `OnActionComplete`
  - 长按分支（`shouldContinue && isFarmTool`）改动：
    - 删除 `ConsumePendingFarmInput()` 和 `ProcessFarmInputAt()` 调用
    - 改为 `GameInputManager.Instance?.OnFarmActionAnimationComplete()`
    - 保持 `EndAction(false)` + `ClearAllCache()` + `isPerformingAction = false`
  - 松开分支（`!shouldContinue`）改动：
    - 删除 `ConsumePendingFarmInput()` 调用
    - 如果 `lockManager.HasCachedHotbarInput`：改为 `ClearActionQueue()`（替代 `ClearPendingFarmInput`）
    - 否则：改为 `GameInputManager.Instance?.OnFarmActionAnimationComplete()`
  - 通用工具分支（`shouldContinue && !isFarmTool`）：完全不变（CP-19）
  - 验收标准：
    - Crush/Watering 动画完成后自动执行队列下一个（CP-18）
    - Slice/Pierce 长按行为不变（CP-19）
    - 动画期间切换工具栏 → 队列被清空

---

## Phase 5：中断、过滤与暂停（模块 D/E/F）

### 任务 5.1：HandleMovement WASD 中断队列

- [x] 5.1 修改 `HandleMovement()` 新增 WASD 中断队列逻辑
  - 文件：`GameInputManager.cs`，方法 `HandleMovement`（约第 349 行）
  - 改动位置：在 `lockManager.IsLocked` 检查之前（约第 380 行之前）
  - 新增逻辑：
    ```
    bool hasWASD = input.sqrMagnitude > 0.01f
    bool hasActiveQueue = _farmActionQueue.Count > 0 || _isProcessingQueue
    if (hasWASD && hasActiveQueue)
    {
      ClearActionQueue()
      CancelFarmingNavigation()
      FarmToolPreview.Instance?.UnlockPosition()
      ToolActionLockManager.Instance?.ForceUnlock()
      // 不 return，继续执行移动逻辑
    }
    ```
  - 🔴 关键：这段代码必须在 `lockManager.IsLocked` 检查之前，否则 WASD 会被锁定拦截
  - 验收标准：队列执行期间 WASD 能立即中断并开始移动（CP-12）

### 任务 5.2：HandleRightClickAutoNav 过滤 CropController

- [x] 5.2 修改 `HandleRightClickAutoNav()` 过滤作物收获
  - 文件：`GameInputManager.cs`，方法 `HandleRightClickAutoNav`（约第 746 行）
  - 改动位置：在 candidates 筛选循环中（约第 870 行附近的 `foreach (var h in hits)` 循环）
  - 新增一行：在 `if (interactable == null) continue` 之后
    ```
    if (interactable is CropController) continue  // 作物收获已迁移到左键
    ```
  - 只过滤 `CropController`，其他 IInteractable（箱子、NPC 等）保持右键交互不变
  - 验收标准：右键点击作物不再触发收获（CP-8）

### 任务 5.3：面板暂停/恢复机制

- [x] 5.3 实现面板打开暂停队列、关闭恢复队列的机制
  - 文件：`GameInputManager.cs`
  - 新增字段：`bool _wasUIOpen = false`
  - 在 `Update()` 中（`HandleUseCurrentTool` 调用之前）新增状态变化检测：
    ```
    bool uiOpen = IsAnyPanelOpen()
    if (uiOpen && !_wasUIOpen)  // 面板刚打开
    {
      _isQueuePaused = true
      CancelCurrentNavigation()  // 只取消导航，不清空队列
    }
    else if (!uiOpen && _wasUIOpen)  // 面板刚关闭
    {
      _isQueuePaused = false
      if (_farmActionQueue.Count > 0 && !_isExecutingFarming)
        ProcessNextAction()
    }
    _wasUIOpen = uiOpen
    ```
  - 新增 `CancelCurrentNavigation()` 方法（轻量版，不同于 `CancelFarmingNavigation`）：
    - 只停止导航协程和导航器
    - 不清空队列、不解锁预览、不重置 `_farmNavState`、不重置 `_isExecutingFarming`
  - `ProcessNextAction` 开头已有 `if (_isQueuePaused) return` 检查（任务 2.3 中实现）
  - 导航到达回调中已有 `if (_isQueuePaused) return` 检查（任务 2.3 中实现）
  - 验收标准：面板打开暂停队列，关闭后队列内容不变并恢复执行（CP-4/CP-13）

### 任务 5.4：ESC 和切换快捷栏清空队列

- [x] 5.4 在 ESC 和切换快捷栏的处理逻辑中加入队列清空
  - 文件：`GameInputManager.cs`
  - ESC 处理（在 `HandlePanelHotkeys` 或相关方法中）：
    - 在现有 ESC 逻辑中增加 `ClearActionQueue()` + `CancelFarmingNavigation()` + `FarmToolPreview.Instance?.UnlockPosition()`
  - 切换快捷栏（在 `HandleHotbarSelection` 中）：
    - 在切换逻辑中增加 `ClearActionQueue()` + `CancelFarmingNavigation()`
    - 预览模式切换保持原有逻辑
  - 验收标准：切换物品/ESC 后队列完全清空，预览恢复（CP-3）

---

## Phase 6：旧方法废弃与清理（模块 F）

### 任务 6.1：废弃旧缓存方法

- [x] 6.1 标记废弃并清理旧的单缓存方法和字段
  - 文件：`GameInputManager.cs`
  - 废弃方法（标记 `[System.Obsolete("10.1.1补丁002：被 FIFO 队列替代")]`）：
    - `CacheFarmInput(int itemId)` — 被 `EnqueueAction` 替代
    - `ConsumePendingFarmInput()` — 被 `ProcessNextAction` 替代
    - `ProcessFarmInputAt(Vector3 worldPos)` — 被队列内部逻辑替代
    - `ForceUpdatePreviewToPosition(Vector3, ItemData)` — 队列通过 LockPosition 处理预览
    - `ClearPendingFarmInput()` — 被 `ClearActionQueue` 替代
  - 废弃字段（可保留但不再使用，或标记 obsolete）：
    - `_hasPendingFarmInput`
    - `_pendingFarmWorldPos`
    - `_pendingFarmItemId`
  - 确认所有调用点已在 Phase 3/4 中替换完毕，无遗漏
  - 验收标准：编译通过，无对废弃方法的调用（getDiagnostics 0 错误）

---

## Phase 7：集成验证

### 任务 7.1：编译验证

- [x] 7.1 getDiagnostics 检查所有修改文件
  - 检查文件列表：
    - `GameInputManager.cs`
    - `PlayerInteraction.cs`
    - `FarmToolPreview.cs`
    - `CropController.cs`
  - 验收标准：0 错误 0 警告

### 任务 7.2：正确性属性逐项审查

- [x] 7.2 代码审查确认 CP-1 ~ CP-19 全部满足
  - 逐项检查每个正确性属性在代码中的实现位置：
    - CP-1（FIFO）：Queue 数据结构
    - CP-2（防重复）：_queuedPositions HashSet
    - CP-3（清空完整性）：ClearActionQueue 方法
    - CP-4（暂停一致性）：_isQueuePaused 标志
    - CP-5（收获最高优先级）：TryDetectAndEnqueueHarvest 在工具检测之前
    - CP-6（收获层级隔离）：layerIndex 过滤
    - CP-7（收获二次验证）：ProcessNextAction 中 CanInteract 检查
    - CP-8（右键不收获）：HandleRightClickAutoNav 中 CropController 过滤
    - CP-9（预览有效性前置）：TryEnqueueFarmTool/TryEnqueueSeed 中 IsValid 检查
    - CP-10（种子用完跳过）：HasSeedRemaining 检查
    - CP-11（动画期间可入队）：保护分支调用 TryEnqueueFromCurrentInput
    - CP-12（WASD 中断）：HandleMovement 中断逻辑在 lockManager 之前
    - CP-13（面板暂停）：_isQueuePaused + 状态变化检测
    - CP-14（预览切换）：ProcessNextAction 中 LockPosition/UnlockPosition
    - CP-15（LockPosition 渲染 1+8）：LockPosition 内部渲染逻辑
    - CP-16（作物底部对齐）：AlignSpriteBottom 在 UpdateVisuals 末尾
    - CP-17（Collect 专用分支）：OnActionComplete 中 Collect 拦截
    - CP-18（Crush/Watering 队列出队）：OnFarmActionAnimationComplete
    - CP-19（Slice/Pierce 不变）：else 分支完全不变
  - 验收标准：每个属性都能在代码中找到对应的保证实现

### 任务 7.3：交互矩阵验证清单

- [x] 7.3 输出运行时验证清单供用户在 Unity Editor 中测试
  - 基于 design V3 第十一章的 6 个矩阵，生成可勾选的测试清单：
    - 左键入队矩阵 S1-S11
    - 队列执行期间点击矩阵 E1-E6
    - 中断矩阵（WASD/ESC/切换快捷栏/面板）
    - 预览状态矩阵
    - OnActionComplete 分支矩阵
    - 队列二次验证矩阵
  - 验收标准：清单完整覆盖所有交互场景

### 任务 7.4：更新 memory.md

- [x] 7.4 追加会话记录到 memory.md
  - 记录所有修改文件
  - 记录完成状态
