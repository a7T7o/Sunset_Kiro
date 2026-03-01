# 补丁004 任务列表 V2 — 三层预览架构重构

> 基于 designV2.md（模块 A-G）。每个任务自包含执行所需全部信息。
> 执行顺序：先移除锁定（A），再改 ghost 逻辑（B/C），然后执行预览层（D/E），队列数据传递（F），最后 GameInputManager 集成（G），编译验证。

---

## Phase 1：FarmToolPreview 锁定机制移除（模块 A）

- [ ] 1.1 移除 LockPosition 机制
  - 文件：`FarmToolPreview.cs`
  - 删除字段（约第 163-166 行）：
    - `private bool _isLocked = false;`
    - `private Vector3 _lockedWorldPosition;`
    - `private Vector3Int _lockedCellPos;`
    - `private int _lockedLayerIndex;`
  - 删除方法 `LockPosition()`（约第 392-440 行）整体
  - 删除方法 `UnlockPosition()`（约第 443-450 行）整体
  - 删除 `UpdateHoePreview` 中的 `if (_isLocked) return;`（约第 510 行）
  - 删除 `UpdateWateringPreview` 中的 `if (_isLocked) return;`（约第 619 行）
  - 搜索 `_isLocked` / `IsLocked` 确认无其他引用
  - 验收标准：编译通过（暂时允许 GameInputManager 中的 LockPosition/UnlockPosition 调用报错，Phase 5 修复）
  - 正确性属性：CP-A1、CP-A2

---

## Phase 2：Ghost 预览逻辑改造（模块 B/C）

- [ ] 2.1 Ghost 差异化过滤 + CurrentGhostTileData 缓存
  - 文件：`FarmToolPreview.cs`
  - 新增字段（字段区域）：
    ```csharp
    private Dictionary<Vector3Int, TileBase> _currentGhostTileData;
    public Dictionary<Vector3Int, TileBase> CurrentGhostTileData => _currentGhostTileData;
    ```
  - 修改 `UpdateHoePreview`（约第 525-545 行的 `foreach` 循环）：
    1. 在循环前获取实际 tilledTilemap：
       ```csharp
       var tilemaps = FarmTileManager.Instance?.GetLayerTilemaps(layerIndex);
       Tilemap actualTilemap = tilemaps?.tilledTilemap;
       ```
    2. 初始化/清空 `_currentGhostTileData`
    3. 循环内增加差异对比：
       ```csharp
       TileBase actualTile = actualTilemap?.GetTile(kvp.Key);
       if (kvp.Value == actualTile) continue;  // 相同则跳过
       ```
    4. 同步更新 `_currentGhostTileData[kvp.Key] = kvp.Value;`
  - 在 `else`（不可锄地）分支中清空 `_currentGhostTileData`
  - 需要确认 `FarmTileManager.GetLayerTilemaps(layerIndex)` 返回类型中是否有 `tilledTilemap` 字段（需读取代码确认具体 API）
  - 验收标准：ghost 预览只显示差异 tile（CP-B1），中心块始终显示（CP-B2）
  - 正确性属性：CP-B1、CP-B2、CP-B3

- [ ] 2.2 浇水 Ghost 进入新格子才随机 + CurrentPuddleVariant
  - 文件：`FarmToolPreview.cs`
  - 新增字段（字段区域）：
    ```csharp
    private Vector3Int _lastWateringCellPos = new Vector3Int(int.MinValue, int.MinValue, 0);
    private int _cachedPuddleVariant = -1;
    public int CurrentPuddleVariant => _cachedPuddleVariant;
    ```
  - 修改 `UpdateWateringPreview`（约第 635-642 行的 `if (isValid)` 块）：
    1. 替换 `GetRandomPuddleTile()` 为缓存逻辑：
       ```csharp
       if (cellPos != _lastWateringCellPos)
       {
           var puddleTiles = FarmVisualManager.Instance?.GetPuddleTiles();
           int count = puddleTiles != null ? puddleTiles.Length : 3;
           _cachedPuddleVariant = Random.Range(0, count);
           _lastWateringCellPos = cellPos;
       }
       var tiles = FarmVisualManager.Instance?.GetPuddleTiles();
       if (tiles != null && _cachedPuddleVariant >= 0 && _cachedPuddleVariant < tiles.Length)
       {
           ghostTilemap.SetTile(cellPos, tiles[_cachedPuddleVariant]);
           currentPreviewPositions.Add(cellPos);
       }
       ```
  - 修改 `Hide()`（约第 786 行）：添加重置
    ```csharp
    _lastWateringCellPos = new Vector3Int(int.MinValue, int.MinValue, 0);
    _cachedPuddleVariant = -1;
    ```
  - 验收标准：同一格子内不闪烁（CP-C1），移出再移回重新随机（CP-C2）
  - 正确性属性：CP-C1、CP-C2、CP-C3、CP-C4

---

## Phase 3：执行预览层（模块 D/E）

- [ ] 3.1 新增执行预览数据结构和方法
  - 文件：`FarmToolPreview.cs`
  - 新增字段（字段区域）：
    ```csharp
    private Dictionary<Vector3Int, List<Vector3Int>> executingTileGroups = new();
    private HashSet<Vector3Int> executingWaterPositions = new();
    private List<(Vector3Int cellPos, SpriteRenderer renderer)> executingSeedPreviews = new();
    ```
  - 新增方法 `PromoteToExecutingPreview(Vector3Int cellPos)`：
    - 耕地：从 `tillQueueTileGroups` 移到 `executingTileGroups`
    - 浇水：从 `queuePreviewPositions` 移到 `executingWaterPositions`（排除种子）
    - 种子：从 `activeSeedQueuePreviews` 移到 `executingSeedPreviews`
    - 从 `queuePreviewPositions` 移除（tile 保留在 tilemap 上）
    - 完整伪代码见 designV2 模块 D 第 5.2 节
  - 新增方法 `RemoveExecutingPreview(Vector3Int cellPos)`：
    - 耕地：从 `executingTileGroups` 获取位置列表，逐个 `SetTile(null)`
    - 浇水：从 `executingWaterPositions` 移除，`SetTile(null)`
    - 种子：回收 SpriteRenderer 到对象池
    - 完整伪代码见 designV2 模块 D 第 5.3 节
  - 验收标准：编译通过
  - 正确性属性：CP-D1、CP-D2、CP-D3

- [ ] 3.2 修复 ClearAllQueuePreviews 边界残留 + 执行预览保护
  - 文件：`FarmToolPreview.cs`，方法 `ClearAllQueuePreviews`（约第 1130 行）
  - 完整重写方法：
    1. 遍历 `tillQueueTileGroups`，逐个清除 tile（跳过 `executingTileGroups` 中的）
    2. 遍历 `queuePreviewPositions`，清除浇水等单点 tile（跳过 `executingWaterPositions` 中的，跳过已在 tillQueueTileGroups 处理的）
    3. 回收种子队列预览（跳过 `executingSeedPreviews` 中的）
    4. 清空 `tillQueueTileGroups`、`queuePreviewPositions`、`activeSeedQueuePreviews`
    5. 不清空 `executingTileGroups`、`executingWaterPositions`、`executingSeedPreviews`
  - 完整伪代码见 designV2 模块 E 第 6.2 节
  - 验收标准：WASD 中断后无边界残留（CP-E2），执行预览不被清除（CP-E3）
  - 正确性属性：CP-E1、CP-E2、CP-E3

---

## Phase 4：队列预览复制 Ghost 数据（模块 F）

- [ ] 4.1 修改 AddQueuePreview 接收 ghostTileData 参数
  - 文件：`FarmToolPreview.cs`，方法 `AddQueuePreview`（约第 1025 行）
  - 签名变更：新增参数 `Dictionary<Vector3Int, TileBase> ghostTileData = null`
    ```csharp
    public void AddQueuePreview(Vector3Int cellPos, int layerIndex, FarmActionType type, 
        int puddleVariant = -1, Dictionary<Vector3Int, TileBase> ghostTileData = null)
    ```
  - Till 分支改造：
    - 如果 `ghostTileData != null && ghostTileData.Count > 0`：使用 ghostTileData
    - 否则：兜底调用 `GetPreviewTiles`（向后兼容）
    - 遍历 tile 数据放到 `queuePreviewTilemap`，记录到 `tillQueueTileGroups`
  - Water 分支：保持现有 puddleVariant 逻辑不变
  - PlantSeed 分支：保持不变
  - 完整伪代码见 designV2 模块 F 第 7.3 节
  - 验收标准：编译通过，队列预览 tile 与 ghost 数据一致（CP-F1）
  - 正确性属性：CP-F1、CP-F2

---

## Phase 5：GameInputManager 集成（模块 G）

- [ ] 5.1 移除所有 LockPosition / UnlockPosition 调用
  - 文件：`GameInputManager.cs`
  - 删除以下调用（逐个确认行号后删除）：
    - `ProcessNextAction` 中的 `FarmToolPreview.Instance?.LockPosition(...)`
    - `ProcessNextAction` 队列空分支中的 `FarmToolPreview.Instance?.UnlockPosition()`
    - `HandleMovement` WASD 中断中的 `FarmToolPreview.Instance?.UnlockPosition()`
    - `HandleMovement` autoNavigator 分支中的 `farmPreviewNav.UnlockPosition()`
    - `CancelFarmingNavigation` 中的 `UnlockPosition()`（如有）
  - 全局搜索 `LockPosition` 和 `UnlockPosition` 确认无遗漏
  - 验收标准：编译通过（配合 Phase 1 的 FarmToolPreview 改动）
  - 正确性属性：CP-G1

- [ ] 5.2 ProcessNextAction 出队时提升为执行预览
  - 文件：`GameInputManager.cs`，方法 `ProcessNextAction`
  - 在 `_isExecutingFarming = true;` 之后，原 `LockPosition` 位置替换为：
    ```csharp
    FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
    ```
  - 验收标准：出队后队列预览转为执行预览（CP-G2）
  - 正确性属性：CP-G2

- [ ] 5.3 动画完成回调改为 RemoveExecutingPreview
  - 文件：`GameInputManager.cs`
  - `OnFarmActionAnimationComplete`（约第 2573 行）：
    - `RemoveQueuePreview` → `RemoveExecutingPreview`
  - `OnCollectAnimationComplete`（约第 2596 行）：
    - `RemoveQueuePreview` → `RemoveExecutingPreview`
  - `ExecuteFarmAction` PlantSeed 分支（约第 2533 行）：
    - 在 `ProcessNextAction()` 之前添加 `RemoveExecutingPreview(request.cellPos)`
  - 验收标准：动画完成后执行预览被清除（CP-G3）
  - 正确性属性：CP-G3

- [ ] 5.4 TryEnqueueFarmTool 传递 Ghost 数据
  - 文件：`GameInputManager.cs`，方法 `TryEnqueueFarmTool`（约第 2689 行）
  - 耕地分支：读取 `FarmToolPreview.Instance.CurrentGhostTileData`，复制快照
  - 浇水分支：读取 `FarmToolPreview.Instance.CurrentPuddleVariant`（替代当前的 `Random.Range`）
  - 传递给 `EnqueueAction` 的新参数
  - 完整伪代码见 designV2 模块 G 第 8.6 节
  - 验收标准：队列预览与 ghost 数据一致（CP-G4、CP-G5）
  - 正确性属性：CP-G4、CP-G5

- [ ] 5.5 EnqueueAction 新增 ghostTileData 参数
  - 文件：`GameInputManager.cs`，方法 `EnqueueAction`（约第 2355 行）
  - 签名变更：`private void EnqueueAction(FarmActionRequest request, Dictionary<Vector3Int, TileBase> ghostTileData = null)`
  - `AddQueuePreview` 调用处传递 `ghostTileData`：
    ```csharp
    FarmToolPreview.Instance?.AddQueuePreview(
        request.cellPos, request.layerIndex, request.type, 
        request.puddleVariant, ghostTileData);
    ```
  - 验收标准：编译通过
  - 正确性属性：CP-F1

---

## Phase 6：编译验证与清理

- [ ] 6.1 编译验证
  - getDiagnostics 检查：
    - `FarmToolPreview.cs`
    - `GameInputManager.cs`
  - 全局搜索确认：
    - `LockPosition` — 无引用（除废弃注释外）
    - `UnlockPosition` — 无引用
    - `_isLocked` — 无引用
  - 验收标准：0 错误 0 警告

- [ ] 6.2 正确性属性逐项审查
  - 逐条检查 CP-A1 ~ CP-G6 共 22 条正确性属性在代码中的对应实现
  - 重点检查：
    - ghost 差异化过滤是否正确获取 tilledTilemap
    - 执行预览保护是否在所有清除路径中生效
    - ghost 数据快照是否是深拷贝（避免后续帧覆盖）

- [ ] 6.3 创建验收指南
  - 在工作区内创建 `验收指南V2.md`
  - 包含：
    - 功能验收清单（可勾选）
    - 测试步骤说明
    - 预期结果描述
    - 已知限制

- [ ] 6.4 更新 memory.md
  - 追加会话记录
  - 记录所有修改文件

---

## 修改文件汇总

| 文件 | 修改内容 |
|------|---------|
| `FarmToolPreview.cs` | 移除锁定机制（A）、差异化过滤+缓存（B）、浇水缓存（C）、执行预览层（D）、ClearAllQueuePreviews 修复（E）、AddQueuePreview 接收 ghost 数据（F） |
| `GameInputManager.cs` | 移除 Lock/Unlock 调用（G）、出队提升执行预览（G）、动画完成清除执行预览（G）、入队传递 ghost 数据（G） |
