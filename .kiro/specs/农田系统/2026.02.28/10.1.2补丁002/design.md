# 10.1.1 补丁002 — 设计文档 V3

> V3 重组原则：按「修改目标（文件 × 方法/区域）」为主轴。每个版块自包含——现状代码摘要、所有需要的改动、改动后完整伪代码、该版块保证的正确性属性。执行时只需看对应版块。
>
> 来源：`补丁002全面分析与修复方案.md`（11 个改进点：P1/P2a/P2b/P3/P5 + V1/V2/V3/V4/V5/V6）
> 用户确认：Q1 连续点击+FIFO队列（非长按）、Q2 移除右键收获改左键+Collect动画+队列、Q3 成熟作物收获最高优先级、Q4 LockPosition时渲染锁定位置1+8

---

## 目录

1. [全局架构：FIFO 操作队列概览](#一全局架构fifo-操作队列概览)
2. [模块 A：GameInputManager — 队列基础设施](#二模块-agameinputmanager--队列基础设施)
3. [模块 B：GameInputManager.HandleUseCurrentTool — 入队入口全面改造](#三模块-bgameinputmanagerhandleuseCurrenttool--入队入口全面改造)
4. [模块 C：GameInputManager — 队列执行引擎与回调](#四模块-cgameinputmanager--队列执行引擎与回调)
5. [模块 D：GameInputManager.HandleMovement — WASD 中断机制](#五模块-dgameinputmanagerhandlemovement--wasd-中断机制)
6. [模块 E：GameInputManager.HandleRightClickAutoNav — 作物过滤](#六模块-egameinputmanagerhandlerightclickautonav--作物过滤)
7. [模块 F：GameInputManager — 面板暂停/恢复与旧方法废弃](#七模块-fgameinputmanager--面板暂停恢复与旧方法废弃)
8. [模块 G：PlayerInteraction.OnActionComplete — 分支改造](#八模块-gplayerinteractiononactioncomplete--分支改造)
9. [模块 H：FarmToolPreview.LockPosition — 1+8 渲染修复](#九模块-hfarmtoolpreviewlockposition--18-渲染修复)
10. [模块 I：CropController — 作物底部对齐](#十模块-icropcontroller--作物底部对齐)
11. [完整交互矩阵](#十一完整交互矩阵)
12. [正确性属性与保证关系](#十二正确性属性与保证关系)
13. [测试框架](#十三测试框架)
14. [涉及文件汇总](#十四涉及文件汇总)

---

## 一、全局架构：FIFO 操作队列概览

### 1.1 设计动机

当前农田操作（锄地/浇水/种植/收获）全部是「单次触发 → 立即执行」模式，辅以 `CacheFarmInput` 单缓存机制（只能缓存一个操作）。玩家无法快速连续点击多块地来排队操作。

用户确认的目标交互模式：
- 锄头/浇水/种子/收获全部改为「连续左键点击 + FIFO 队列缓存」
- 废弃长按连续操作（P1 原始需求的长按方案被用户否决）
- 收获从右键迁移到左键，优先级最高，有 Collect 动画
- 镐子/斧头等通用工具保持原有长按行为不变

### 1.2 队列生命周期

```
玩家左键点击 → HandleUseCurrentTool
  ├─ 收获检测（最高优先级）→ EnqueueAction(Harvest)
  ├─ 锄头 Valid → EnqueueAction(Till)
  ├─ 浇水 Valid → EnqueueAction(Water)
  └─ 种子 Valid → EnqueueAction(PlantSeed)

EnqueueAction → 防重复 → 入队 → 队列之前为空且未暂停 → ProcessNextAction()

ProcessNextAction → 暂停检查 → 队列空则结束 → 取队首
  → 二次验证 → LockPosition → 距离判断
  → 近距离直接执行 / 远距离导航
  → 动画完成回调 → ProcessNextAction()（循环）

中断:
  ├─ WASD → 清空队列 + 取消导航 + 解锁 + ForceUnlock
  ├─ ESC/切换快捷栏 → 清空队列 + 取消导航 + 解锁
  └─ 面板打开 → 暂停队列（不清空）→ 面板关闭 → 恢复
```

### 1.3 与现有系统的替换关系

| 旧机制 | 新机制 | 说明 |
|--------|--------|------|
| `CacheFarmInput(itemId)` | `EnqueueAction(request)` | 单缓存 → FIFO 队列 |
| `_hasPendingFarmInput` | `_farmActionQueue.Count > 0` | 单标志 → 队列长度 |
| `ConsumePendingFarmInput()` | `ProcessNextAction()` | 消费单缓存 → 出队执行 |
| `ProcessFarmInputAt(worldPos)` | 废弃 | 队列内部处理 |
| `TryTillSoil` 近距离直接执行 | `ExecuteFarmAction` 统一分发 | 统一入口 |
| `TryWaterTile` 近距离直接执行 | `ExecuteFarmAction` 统一分发 | 统一入口 |
| `TryPlantSeed` 近距离直接执行 | `ExecuteFarmAction` 统一分发 | 统一入口 |

### 1.4 废弃方法清单

以下方法在队列化改造后废弃（标记 `[Obsolete]` 或删除）：
- `CacheFarmInput(int itemId)` — 被 `EnqueueAction` 替代
- `ConsumePendingFarmInput()` — 被 `ProcessNextAction` 替代
- `ProcessFarmInputAt(Vector3 worldPos)` — 被队列内部逻辑替代
- `ForceUpdatePreviewToPosition(Vector3, ItemData)` — 队列通过 `LockPosition` 直接处理预览
- `_hasPendingFarmInput` / `_pendingFarmWorldPos` / `_pendingFarmItemId` — 被队列字段替代


---

## 二、模块 A：GameInputManager — 队列基础设施

> 涉及改进点：P1、P2a、P2b、V1、V2
> 本版块只定义数据结构和字段，不含逻辑方法（方法在模块 B/C/D/E/F/G 中）

### 2.1 新增枚举与结构体

```csharp
// 农田操作类型
public enum FarmActionType { Till, Water, PlantSeed, Harvest }

// 操作请求（值类型，轻量）
public struct FarmActionRequest
{
    public FarmActionType type;
    public Vector3Int cellPos;        // 目标格子坐标
    public int layerIndex;            // 目标层级索引
    public Vector3 worldPos;          // 目标世界坐标（格子中心，用于导航和距离判断）
    public CropController targetCrop; // 仅 Harvest 类型使用，其他类型为 null
}
```

### 2.2 新增字段

```csharp
// ===== FIFO 操作队列 =====
private Queue<FarmActionRequest> _farmActionQueue = new();
private HashSet<(int layerIndex, Vector3Int cellPos)> _queuedPositions = new();
private bool _isProcessingQueue = false;   // 队列正在执行中（有操作在处理）
private bool _isQueuePaused = false;       // 队列暂停（面板打开时）

// ===== 收获相关 =====
private CropController _currentHarvestTarget = null;  // 当前正在收获的作物（Collect 动画期间持有）
private FarmActionRequest _currentProcessingRequest;   // 当前正在处理的请求（用于回调时获取上下文）
```

### 2.3 `_isExecutingFarming` 语义变更（V2 漏洞修补）

现状：`_isExecutingFarming` 在 `TryTillSoil`/`TryWaterTile`/`TryPlantSeed` 近距离执行时设为 true，在 finally 块中设为 false。用于 `HandleUseCurrentTool` 入口的保护分支。

改造后语义：
- 设为 true 的时机：`ProcessNextAction` 开始执行某个操作时（无论近距离还是远距离）
- 设为 false 的时机：该操作完成回调后（`OnFarmActionAnimationComplete` / `OnCollectAnimationComplete` / 种子直接完成）
- `ClearActionQueue` 时也重置为 false
- 保护分支中不再调用 `CacheFarmInput`，而是直接走 `EnqueueAction`（V1 修补）

### 2.4 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-2 | 同一 (layerIndex, cellPos) 不会在队列中出现两次 | `_queuedPositions` HashSet 防重复 |
| CP-3 | ClearActionQueue 后队列为空、防重复集合为空、_isProcessingQueue = false | ClearActionQueue 方法实现 |

---

## 三、模块 B：GameInputManager.HandleUseCurrentTool — 入队入口全面改造

> 涉及改进点：P1、P2a、P2b、V1、V2、V6、CP-5/CP-6/CP-9/CP-11
> 这是改动量最大的方法，需要完整重写左键点击的处理逻辑

### 3.1 现状代码结构（当前）

```
HandleUseCurrentTool():
  if (uiOpen) return
  
  // 保护分支1：_isExecutingFarming → CacheFarmInput → return
  // 保护分支2：isPerformingAction → CacheFarmInput → return
  
  if (GetMouseButtonDown(0)):
    // 导航中重新点击处理
    // 放置模式处理
    // 工具分发：Hoe/WateringCan → TryHandleFarmingTool → return
    // 种子 → TryPlantSeed
    // 武器 → RequestAction
```

### 3.2 改造后完整伪代码

```
HandleUseCurrentTool():
  if (uiOpen) return
  
  // ===== 保护分支改造（V1 漏洞修补）=====
  // 原来：_isExecutingFarming → CacheFarmInput（单缓存）
  // 改为：_isExecutingFarming → 尝试入队（FIFO 队列）
  if (_isExecutingFarming || (playerInteraction != null && playerInteraction.IsPerformingAction()))
  {
    if (Input.GetMouseButtonDown(0) && !IsPointerOverUI())
    {
      // 尝试入队：先检测收获，再检测工具/种子
      TryEnqueueFromCurrentInput()
    }
    return  // 无论是否成功入队，都 return（不穿透到下面的正常流程）
  }
  
  if (!Input.GetMouseButtonDown(0)) return
  if (IsPointerOverUI()) return
  
  // ===== 导航中重新点击（保持原有逻辑，但适配队列）=====
  if (_farmNavState == FarmNavState.Navigating || _farmNavState == FarmNavState.Locked)
  {
    // 如果队列正在处理，新点击走入队
    if (_isProcessingQueue)
    {
      TryEnqueueFromCurrentInput()
      return
    }
    // 否则保持原有的"中断导航 → 重新开始"逻辑
    var farmPreview = FarmToolPreview.Instance
    if (farmPreview != null && farmPreview.IsValid())
    {
      if (farmPreview.IsLocked && farmPreview.CurrentCellPos == farmPreview.LockedCellPos)
        return  // 点击同一位置，不中断
      CancelFarmingNavigation()
      // 继续往下走
    }
    else
    {
      CancelFarmingNavigation()
      return
    }
  }
  
  // ===== 放置模式（保持不变）=====
  if (PlacementManager.Instance?.IsPlacementMode == true)
  {
    PlacementManager.Instance.OnLeftClick()
    return
  }
  
  if (inventory == null || database == null || hotbarSelection == null) return
  
  // ===== 🔴 第一优先级：收获检测（CP-5、CP-6）=====
  if (TryDetectAndEnqueueHarvest())
    return
  
  // ===== 第二优先级：农田工具/种子入队 =====
  int idx = Clamp(hotbarSelection.selectedIndex)
  var slot = inventory.GetSlot(idx)
  if (slot.IsEmpty) return
  var itemData = database.GetItemByID(slot.itemId)
  if (itemData == null) return
  
  if (itemData is ToolData tool)
  {
    if (tool.toolType == Hoe || tool.toolType == WateringCan)
    {
      TryEnqueueFarmTool(tool)  // 入队而非直接执行
      return  // 始终 return，不穿透
    }
    // 其他工具（镐子/斧头等）：保持原有逻辑不变
    var toolAction = ResolveAction(tool.toolType)
    playerInteraction?.RequestAction(toolAction)
  }
  else if (itemData is SeedData seedData)
  {
    TryEnqueueSeed(seedData)  // 入队而非直接执行
  }
  else if (itemData is WeaponData weapon)
  {
    var action = ResolveWeaponAction(weapon.animActionType)
    playerInteraction?.RequestAction(action)
  }
```

### 3.3 TryDetectAndEnqueueHarvest — 收获检测方法（新增）

```
bool TryDetectAndEnqueueHarvest():
  Vector3 mouseWorldPos = GetMouseWorldPosition()
  var hits = Physics2D.OverlapPointAll(mouseWorldPos)
  int playerLayer = GetCurrentPlayerLayerIndex()  // 🔴 玩家当前层级
  
  foreach (var hit in hits):
    // 跳过玩家自身
    if (hit.transform == playerMovement.transform) continue
    
    var interactable = hit.GetComponent<IInteractable>()
    if (interactable == null)
      interactable = hit.GetComponentInParent<IInteractable>()
    
    if (interactable is CropController crop):
      // V6 层级过滤：只检测玩家当前层级的作物
      if (crop.layerIndex != playerLayer) continue
      // 可收获检测（Mature 或 WitheredMature）
      if (!crop.CanInteract(null)) continue
      
      // 构建请求并入队
      Vector3 cropWorldPos = crop.transform.position
      EnqueueAction(new FarmActionRequest {
        type = Harvest,
        cellPos = crop.CellPos,       // CropController 上的格子坐标
        layerIndex = crop.layerIndex,
        worldPos = cropWorldPos,
        targetCrop = crop
      })
      return true
  
  return false
```

关键设计决策：
- 走 `IInteractable` 接口检测，不绕过接口抽象（10.1.5 交互细节）
- 层级过滤在 `CanInteract` 之前（CP-6）
- 无论手持什么物品都执行检测（CP-5）
- 收获检测在所有工具操作之前执行（最高优先级）

### 3.4 TryEnqueueFarmTool — 农田工具入队（新增）

```
void TryEnqueueFarmTool(ToolData tool):
  var farmPreview = FarmToolPreview.Instance
  if (farmPreview == null || !farmPreview.IsValid()) return  // CP-9：预览无效不入队
  
  var type = tool.toolType == Hoe ? FarmActionType.Till : FarmActionType.Water
  EnqueueAction(new FarmActionRequest {
    type = type,
    cellPos = farmPreview.CurrentCellPos,
    layerIndex = farmPreview.CurrentLayerIndex,
    worldPos = farmPreview.CurrentCursorPos,
    targetCrop = null
  })
```

### 3.5 TryEnqueueSeed — 种子入队（新增）

```
void TryEnqueueSeed(SeedData seedData):
  var farmPreview = FarmToolPreview.Instance
  if (farmPreview == null || !farmPreview.IsValid()) return  // CP-9
  
  EnqueueAction(new FarmActionRequest {
    type = PlantSeed,
    cellPos = farmPreview.CurrentCellPos,
    layerIndex = farmPreview.CurrentLayerIndex,
    worldPos = farmPreview.CurrentCursorPos,
    targetCrop = null
  })
```

### 3.6 TryEnqueueFromCurrentInput — 保护分支统一入队（新增）

在 `_isExecutingFarming` 或 `isPerformingAction` 保护分支中调用，替代旧的 `CacheFarmInput`：

```
void TryEnqueueFromCurrentInput():
  // 先尝试收获
  if (TryDetectAndEnqueueHarvest()) return
  
  // 再尝试工具/种子
  int idx = Clamp(hotbarSelection.selectedIndex)
  var slot = inventory.GetSlot(idx)
  if (slot.IsEmpty) return
  var itemData = database.GetItemByID(slot.itemId)
  
  if (itemData is ToolData tool && (tool.toolType == Hoe || tool.toolType == WateringCan))
    TryEnqueueFarmTool(tool)
  else if (itemData is SeedData seedData)
    TryEnqueueSeed(seedData)
```

### 3.7 GetCurrentPlayerLayerIndex — 玩家层级获取

需要确认现有代码中获取玩家当前层级的 API。可能的来源：
- `PlacementLayerDetector` 组件
- `PlayerLayerManager` 组件
- 或通过 `FarmTileManager.GetCurrentLayerIndex(playerPos)` 计算

实现时需要在代码中确认具体 API 并使用。

### 3.8 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-5 | 鼠标位置有可收获作物（同层级）时，无论手持什么，左键触发收获 | `TryDetectAndEnqueueHarvest` 在工具检测之前执行 |
| CP-6 | 只能收获玩家当前层级的作物 | `crop.layerIndex != playerLayer` 过滤 |
| CP-9 | 只有预览 Valid 时才入队 | `TryEnqueueFarmTool`/`TryEnqueueSeed` 开头检查 `IsValid()` |
| CP-11 | 动画期间/执行期间的点击走入队逻辑 | 保护分支调用 `TryEnqueueFromCurrentInput` 替代 `CacheFarmInput` |

---

## 四、模块 C：GameInputManager — 队列执行引擎与回调

> 涉及改进点：P1、P2a、P2b、V1、V2
> 本版块包含队列的核心执行逻辑：入队、出队、执行分发、完成回调

### 4.1 EnqueueAction — 入队方法

```
EnqueueAction(FarmActionRequest request):
  // 防重复：同一 (layerIndex, cellPos) 不重复入队（CP-2）
  var key = (request.layerIndex, request.cellPos)
  if (_queuedPositions.Contains(key)) return
  
  // Harvest 额外防重复：同一 CropController 实例不重复
  if (request.type == Harvest):
    foreach (var existing in _farmActionQueue):
      if (existing.type == Harvest && existing.targetCrop == request.targetCrop) return
  
  _queuedPositions.Add(key)
  _farmActionQueue.Enqueue(request)
  
  // 如果队列之前为空且未暂停且没有正在执行的操作 → 启动处理
  if (!_isProcessingQueue && !_isQueuePaused):
    ProcessNextAction()
```

### 4.2 ProcessNextAction — 出队执行方法

```
ProcessNextAction():
  // 暂停检查（V5）
  if (_isQueuePaused) return
  
  // 队列为空 → 结束
  if (_farmActionQueue.Count == 0):
    _isProcessingQueue = false
    _isExecutingFarming = false
    _queuedPositions.Clear()
    // 解锁预览，恢复鼠标跟随
    var farmPreview = FarmToolPreview.Instance
    if (farmPreview != null) farmPreview.UnlockPosition()
    _farmNavState = FarmNavState.Preview
    return
  
  _isProcessingQueue = true
  var request = _farmActionQueue.Dequeue()
  _currentProcessingRequest = request
  
  // ===== 二次验证 =====
  switch (request.type):
    case PlantSeed:
      // 种子用完检测（CP-10）：检查手持是否仍为种子且余量 > 0
      if (!HasSeedRemaining()):
        _queuedPositions.Remove((request.layerIndex, request.cellPos))
        ProcessNextAction()  // 跳过，继续下一个（不清空整个队列）
        return
    case Harvest:
      // 作物可收获二次验证（CP-7）
      if (request.targetCrop == null || !request.targetCrop.CanInteract(null)):
        _queuedPositions.Remove((request.layerIndex, request.cellPos))
        ProcessNextAction()  // 跳过
        return
  
  // ===== 距离判断 =====
  // 🔴 玩家位置 = Collider 中心（最高优先级规则）
  float distance = Vector2.Distance(
    playerCollider.bounds.center,
    request.worldPos
  )
  
  _isExecutingFarming = true
  
  // 锁定预览到目标位置（P3 修复：LockPosition 内部会渲染 1+8）
  var preview = FarmToolPreview.Instance
  if (preview != null)
    preview.LockPosition(request.worldPos, request.cellPos, request.layerIndex)
  
  if (distance <= farmToolReach):
    // 近距离：直接执行
    _farmNavState = FarmNavState.Executing
    ExecuteFarmAction(request)
  else:
    // 远距离：导航到目标后执行
    _farmNavState = FarmNavState.Locked
    StartFarmingNavigation(request.worldPos, () => {
      // 导航到达回调
      if (_isQueuePaused) return  // V5：面板打开期间到达，不执行
      _farmNavState = FarmNavState.Executing
      ExecuteFarmAction(request)
    })
```

### 4.3 ExecuteFarmAction — 执行分发方法

```
ExecuteFarmAction(FarmActionRequest request):
  // 面向目标
  FaceTarget(request.worldPos)
  
  switch (request.type):
    case Till:
      playerInteraction?.RequestAction(AnimState.Crush)
      ExecuteTillSoil(request.layerIndex, request.cellPos)
      // 动画完成后由 OnActionComplete → OnFarmActionAnimationComplete 回调
      break
    
    case Water:
      playerInteraction?.RequestAction(AnimState.Watering)
      ExecuteWaterTile(request.layerIndex, request.cellPos)
      // 动画完成后由 OnActionComplete → OnFarmActionAnimationComplete 回调
      break
    
    case PlantSeed:
      // 种子无动画，直接执行
      var seedData = GetCurrentSeedData()  // 从当前手持物品获取
      if (seedData != null)
        ExecutePlantSeed(seedData, request.layerIndex, request.cellPos)
      _isExecutingFarming = false
      _queuedPositions.Remove((request.layerIndex, request.cellPos))
      ProcessNextAction()  // 立即取下一个
      break
    
    case Harvest:
      _currentHarvestTarget = request.targetCrop
      playerInteraction?.RequestAction(AnimState.Collect)
      // 动画完成后由 OnActionComplete → OnCollectAnimationComplete 回调
      break
```

### 4.4 OnFarmActionAnimationComplete — 农田工具动画完成回调（新增）

由 `PlayerInteraction.OnActionComplete` 中的 Crush/Watering 农田工具分支调用：

```
public void OnFarmActionAnimationComplete():
  _isExecutingFarming = false
  _queuedPositions.Remove((_currentProcessingRequest.layerIndex, _currentProcessingRequest.cellPos))
  ProcessNextAction()
```

### 4.5 OnCollectAnimationComplete — 收获动画完成回调（新增）

由 `PlayerInteraction.OnActionComplete` 中的 Collect 专用分支调用：

```
public void OnCollectAnimationComplete():
  // 执行收获逻辑（走 IInteractable 接口）
  if (_currentHarvestTarget != null && _currentHarvestTarget.CanInteract(null)):
    var context = BuildInteractionContext()
    _currentHarvestTarget.OnInteract(context)
  
  _currentHarvestTarget = null
  _isExecutingFarming = false
  _queuedPositions.Remove((_currentProcessingRequest.layerIndex, _currentProcessingRequest.cellPos))
  ProcessNextAction()
```

### 4.6 ClearActionQueue — 清空方法

```
ClearActionQueue():
  _farmActionQueue.Clear()
  _queuedPositions.Clear()
  _isProcessingQueue = false
  _isExecutingFarming = false
  _currentHarvestTarget = null
  _currentProcessingRequest = default
```

### 4.7 HasSeedRemaining — 种子余量检查

```
bool HasSeedRemaining():
  int idx = Clamp(hotbarSelection.selectedIndex)
  var slot = inventory.GetSlot(idx)
  if (slot.IsEmpty) return false
  var itemData = database.GetItemByID(slot.itemId)
  if (itemData is not SeedData) return false
  // 检查种子袋余量（如果使用 SeedBagHelper）
  // 或直接检查 slot.amount > 0
  return slot.amount > 0
```

### 4.8 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-1 | 入队顺序 = 执行顺序（FIFO） | `Queue<T>` 数据结构保证 |
| CP-2 | 同一格子不重复入队 | `_queuedPositions` HashSet |
| CP-7 | Harvest 执行前重新检查 CanInteract | `ProcessNextAction` 二次验证 |
| CP-10 | 种子用完后队列中剩余种子操作被跳过 | `ProcessNextAction` 中 `HasSeedRemaining` 检查 |
| CP-14 | 队列为空时预览跟随鼠标，执行时锁定到当前目标 | `ProcessNextAction` 队列空时 `UnlockPosition`，执行时 `LockPosition` |

---

## 五、模块 D：GameInputManager.HandleMovement — WASD 中断机制

> 涉及改进点：V4
> 核心问题：队列执行期间 `lockManager.IsLocked` 阻止 WASD 输入，玩家无法主动中断

### 5.1 现状代码关键段

```csharp
// 当前 HandleMovement 中的锁定检查（约第 380 行）
var lockManager = ToolActionLockManager.Instance;
if (lockManager != null && lockManager.IsLocked)
{
    if (input.sqrMagnitude > 0.01f)
        lockManager.CacheDirection(input);
    if (playerMovement != null) playerMovement.SetMovementInput(Vector2.zero, false);
    return;  // ← 队列执行期间 WASD 被拦截在这里
}
```

### 5.2 改造：在 lockManager 检查之前插入队列中断逻辑

```
HandleMovement():
  // ... uiOpen 检查（保持不变）...
  
  Vector2 input = GetMovementInput()
  bool shift = GetShiftKey()
  
  // ===== V4：WASD 中断队列（优先级高于 lockManager）=====
  bool hasWASD = input.sqrMagnitude > 0.01f
  bool hasActiveQueue = _farmActionQueue.Count > 0 || _isProcessingQueue
  
  if (hasWASD && hasActiveQueue)
  {
    ClearActionQueue()
    CancelFarmingNavigation()
    var farmPreview = FarmToolPreview.Instance
    if (farmPreview != null) farmPreview.UnlockPosition()
    var lockMgr = ToolActionLockManager.Instance
    if (lockMgr != null) lockMgr.ForceUnlock()
    // 不 return，继续执行下面的移动逻辑
  }
  
  // ===== 原有 lockManager 检查（保持不变）=====
  var lockManager = ToolActionLockManager.Instance
  if (lockManager != null && lockManager.IsLocked)
  {
    // ... 原有逻辑 ...
    return
  }
  
  // ===== 原有 autoNavigator 检查 =====
  // 注意：这里已有 WASD 取消导航 + 清除缓存 + 解锁预览的逻辑
  // 需要确保不与上面的队列中断逻辑冲突
  // 如果队列已被清空，_farmNavState 已被 CancelFarmingNavigation 重置
  // autoNavigator.IsActive 也已被 ForceCancel，所以不会重复处理
  if (autoNavigator != null && autoNavigator.IsActive)
  {
    if (hasWASD)
    {
      autoNavigator.ForceCancel()
      CancelFarmingNavigation()
      _hasPendingFarmInput = false  // 旧字段，队列化后可移除
      var farmPreviewNav = FarmToolPreview.Instance
      if (farmPreviewNav != null) farmPreviewNav.UnlockPosition()
      if (playerMovement != null) playerMovement.SetMovementInput(input, shift)
    }
    return
  }
  
  // ... 原有 _farmNavState 检查和移动逻辑（保持不变）...
```

### 5.3 关键设计决策

- WASD 中断检测在 `lockManager.IsLocked` 检查之前执行
- 只有当队列非空或正在处理时才触发中断逻辑，避免影响非农田场景的锁定行为
- `ForceUnlock` 确保 lockManager 不会卡住玩家
- 中断后不 return，让移动逻辑正常执行（玩家立即开始移动）

### 5.4 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-12 | WASD 输入时，无论 lockManager 状态，队列被清空、导航被取消 | 中断逻辑在 lockManager 检查之前 |

---

## 六、模块 E：GameInputManager.HandleRightClickAutoNav — 作物过滤

> 涉及改进点：P2b（收获迁移到左键的连带修改）

### 6.1 现状

`HandleRightClickAutoNav` 通过 `Physics2D.OverlapPointAll` 检测所有 `IInteractable`，包括 `CropController`。右键点击成熟作物会触发导航+收获。

### 6.2 改造

在 candidates 筛选循环中，跳过 `CropController` 类型的 `IInteractable`：

```
// 在 HandleRightClickAutoNav 的 candidates 筛选循环中
foreach (var h in hits):
  // ... 忽略自身 ...
  var interactable = h.GetComponent<IInteractable>() ?? h.GetComponentInParent<IInteractable>()
  if (interactable == null) continue
  
  // 🔴 P2b：作物收获已迁移到左键，右键不再触发
  if (interactable is CropController) continue
  
  // ... 原有逻辑 ...
```

注意：只过滤 `CropController`，其他 `IInteractable`（箱子、NPC 等）保持右键交互不变。

### 6.3 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-8 | 右键点击作物不再触发收获 | candidates 循环中 `if (interactable is CropController) continue` |

---

## 七、模块 F：GameInputManager — 面板暂停/恢复与旧方法废弃

> 涉及改进点：V5（面板暂停）、V1（旧方法废弃）

### 7.1 面板暂停/恢复机制

#### 7.1.1 暂停触发

需要在面板打开事件中设置暂停。当前 `HandleUseCurrentTool` 开头 `if (uiOpen) return` 已阻止新点击，但队列中正在执行的导航不会停止。

方案：在 `Update` 循环中检测 `uiOpen` 状态变化，或在面板打开/关闭的事件回调中处理。

```
// 方案 A：在 Update 中检测状态变化
private bool _wasUIOpen = false;

// 在 Update() 中，HandleUseCurrentTool 之前
bool uiOpen = IsAnyPanelOpen()
if (uiOpen && !_wasUIOpen)
{
  // 面板刚打开
  _isQueuePaused = true
  if (_isProcessingQueue)
    CancelCurrentNavigation()  // 只取消当前导航，不清空队列
}
else if (!uiOpen && _wasUIOpen)
{
  // 面板刚关闭
  _isQueuePaused = false
  if (_farmActionQueue.Count > 0 && !_isExecutingFarming)
    ProcessNextAction()  // 恢复执行
}
_wasUIOpen = uiOpen
```

注意：`CancelCurrentNavigation` 不同于 `CancelFarmingNavigation`——前者只取消导航不清空队列不解锁预览，后者是完整清理。需要新增一个轻量版的导航取消方法，或者在 `CancelFarmingNavigation` 中增加参数控制是否清空队列。

#### 7.1.2 CancelCurrentNavigation — 轻量导航取消（新增）

```
CancelCurrentNavigation():
  // 只停止导航协程和导航器，不清空队列、不解锁预览、不重置状态
  if (_farmingNavigationCoroutine != null)
    StopCoroutine(_farmingNavigationCoroutine)
    _farmingNavigationCoroutine = null
  if (autoNavigator != null && autoNavigator.IsActive)
    autoNavigator.ForceCancel()
  _farmNavigationAction = null
```

#### 7.1.3 暂停期间的行为

- `ProcessNextAction` 开头检查 `_isQueuePaused` → return
- 导航到达回调中检查 `_isQueuePaused` → return（不执行操作）
- `HandleUseCurrentTool` 开头 `if (uiOpen) return` 已阻止新入队
- 队列内容保持不变

### 7.2 旧方法废弃

以下方法标记 `[System.Obsolete]` 并在所有调用点替换：

| 废弃方法 | 替代 | 调用点 |
|---------|------|--------|
| `CacheFarmInput(int)` | `TryEnqueueFromCurrentInput()` | HandleUseCurrentTool 保护分支 ×2 |
| `ConsumePendingFarmInput()` | `ProcessNextAction()` | OnActionComplete 长按分支、松开分支 |
| `ProcessFarmInputAt(Vector3)` | 废弃 | OnActionComplete 长按分支 |
| `ClearPendingFarmInput()` | `ClearActionQueue()` | OnActionComplete hotbar 缓存分支 |

### 7.3 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-4 | 暂停前队列内容 = 恢复后队列内容 | 暂停只设标志+取消导航，不清空队列 |
| CP-13 | 面板打开暂停队列，关闭后恢复执行 | `_isQueuePaused` 标志 + 状态变化检测 |

---

## 八、模块 G：PlayerInteraction.OnActionComplete — 分支改造

> 涉及改进点：V3（Collect 专用分支）、CP-17/CP-18/CP-19
> 这是第二个需要重点改造的文件，改动集中在 OnActionComplete 一个方法中

### 8.1 现状代码结构

```
OnActionComplete():
  // Collect → isCarrying = true（当前只用于搬运状态标记）
  // Death → isCarrying = false
  
  ApplyCachedDirectionToFacing()
  
  bool shouldContinue = isCurrentlyHolding && IsToolAction(currentAction)
  // IsToolAction 包含：Slice/Crush/Pierce/Watering（不含 Collect）
  
  if (shouldContinue):
    if (isFarmTool):
      // 农田工具长按分支：EndAction(false) → ConsumePendingFarmInput / ProcessFarmInputAt
    else:
      // 通用工具长按分支：EndAction(true) → StartAction(repeat)
  else:
    // 松开分支：ConsumePendingFarmInput → EndAction(false) → ApplyCachedHotbarSwitch
```

### 8.2 改造后完整伪代码

```
OnActionComplete():
  // ===== 🔴 Collect 专用分支（V3 漏洞修补）=====
  // 必须在 shouldContinue 判断之前处理，因为 IsToolAction(Collect) = false
  if (currentAction == AnimState.Collect):
    // Collect 动画完成 → 通知 GameInputManager 执行收获并取队列下一个
    animController?.StopAnimationTracking()
    lockManager?.EndAction(false)
    lockManager?.ClearAllCache()
    isPerformingAction = false
    GameInputManager.Instance?.OnCollectAnimationComplete()
    return  // 🔴 不进入后续任何分支
  
  // ===== 原有 Collect/Death 状态标记（Collect 已被上面拦截，这里只剩 Death）=====
  if (currentAction == AnimState.Death):
    isCarrying = false
  
  ApplyCachedDirectionToFacing()
  
  bool isCurrentlyHolding = Input.GetMouseButton(0)
  bool shouldContinue = isCurrentlyHolding && IsToolAction(currentAction)
  
  if (shouldContinue):
    var gim = GameInputManager.Instance
    bool isFarmTool = gim != null && gim.IsHoldingFarmTool()
    
    if (isFarmTool):
      // ===== 农田工具分支改造（CP-18）=====
      // 原来：EndAction → ConsumePendingFarmInput / ProcessFarmInputAt（旧缓存）
      // 改为：EndAction → 通知队列取下一个
      animController?.StopAnimationTracking()
      lockManager?.EndAction(false)
      lockManager?.ClearAllCache()
      isPerformingAction = false
      gim.OnFarmActionAnimationComplete()
      // 🔴 不再调用 ConsumePendingFarmInput / ProcessFarmInputAt
    else:
      // ===== 通用工具分支（CP-19：保持原有长按行为不变）=====
      animController?.StopAnimationTracking()
      lockManager?.EndAction(true)
      StartAction(actionToRepeat, true)
  
  else:
    // ===== 松开分支改造 =====
    var gimRelease = GameInputManager.Instance
    if (gimRelease != null):
      if (lockManager != null && lockManager.HasCachedHotbarInput):
        // 动画期间切换了工具栏 → 清空队列（不消费）
        gimRelease.ClearActionQueue()
      else:
        // 🔴 原来：ConsumePendingFarmInput（消费单缓存）
        // 改为：通知队列取下一个
        gimRelease.OnFarmActionAnimationComplete()
    
    layerAnimSync?.ForceHideTool()
    animController?.StopAnimationTracking()
    isPerformingAction = false
    lockManager?.EndAction(false)
    ApplyCachedHotbarSwitch()
    lockManager?.ClearAllCache()
```

### 8.3 关键设计决策

1. Collect 分支在 `shouldContinue` 判断之前拦截，因为 `IsToolAction(Collect)` 返回 false，如果不提前拦截会掉入松开分支
2. Collect 不加入 `IsToolAction`（收获不支持长按连续）
3. 农田工具分支（Crush/Watering）不再检测 `isCurrentlyHolding`——无论鼠标是否按住，都走队列出队。长按行为由队列接管（用户在按住期间的新点击已经入队了）
4. 通用工具（Slice/Pierce）保持原有长按行为完全不变
5. 松开分支中，如果有 hotbar 缓存（动画期间切换了工具栏），清空整个队列而非只清除单缓存

### 8.4 IsToolAction 不修改

`IsToolAction` 保持原样（Slice/Crush/Pierce/Watering），不加入 Collect。Collect 通过专用分支处理。

### 8.5 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-17 | Collect 动画完成走专用分支，不进入 IsToolAction 长按逻辑 | `if (currentAction == Collect)` 在 `shouldContinue` 之前 |
| CP-18 | Crush/Watering 动画完成后走队列出队，不再走旧长按逻辑 | 农田工具分支调用 `OnFarmActionAnimationComplete` |
| CP-19 | Slice/Pierce 保持原有长按行为 | `else` 分支完全不变 |

---

## 九、模块 H：FarmToolPreview.LockPosition — 1+8 渲染修复

> 涉及改进点：P3
> 改动集中在一个方法，但需要理解 UpdateHoePreview 的渲染逻辑

### 9.1 现状代码

```csharp
public void LockPosition(Vector3 worldPos, Vector3Int cellPos, int layerIndex)
{
    _isLocked = true;
    _lockedWorldPosition = worldPos;
    _lockedCellPos = cellPos;
    _lockedLayerIndex = layerIndex;
    UpdateCursor(layerIndex, cellPos);  // ← 只更新光标，不渲染 1+8 GhostTilemap
}
```

锁定后，`UpdateHoePreview` 每帧在 `if (_isLocked) return` 处跳过，GhostTilemap 永远不会更新到锁定位置。

### 9.2 改造后完整伪代码

```
LockPosition(Vector3 worldPos, Vector3Int cellPos, int layerIndex):
  _isLocked = true
  _lockedWorldPosition = worldPos
  _lockedCellPos = cellPos
  _lockedLayerIndex = layerIndex
  
  // 🔴 P3 修复：锁定时执行一次完整的 GhostTilemap 渲染
  if (isHoeMode && FarmlandBorderManager.Instance != null):
    ClearGhostTilemap()
    
    // 检查该位置是否可以锄地（决定是否显示 1+8）
    bool canTill = FarmTileManager.Instance != null && 
                   FarmTileManager.Instance.CanTillAt(layerIndex, cellPos)
    
    // 也检查枯萎作物清除（与 UpdateHoePreview 逻辑一致）
    bool canClearWithered = false
    if (!canTill && FarmTileManager.Instance != null):
      var tileData = FarmTileManager.Instance.GetTileData(layerIndex, cellPos)
      if (tileData?.cropController != null && 
          tileData.cropController.GetState() == CropState.WitheredImmature):
        canClearWithered = true
    
    if (canTill):
      var previewTiles = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos)
      foreach (var kvp in previewTiles):
        if (kvp.Value != null):
          ghostTilemap.SetTile(kvp.Key, kvp.Value)
          currentPreviewPositions.Add(kvp.Key)
  
  // 更新光标（保持原有）
  UpdateCursor(layerIndex, cellPos)
```

### 9.3 关键设计决策

- 只有锄头模式（`isHoeMode`）才渲染 1+8 预览，浇水模式只更新光标
- 渲染逻辑与 `UpdateHoePreview` 中的 GhostTilemap 渲染部分保持一致（包括 `canTill` 和 `canClearWithered` 检查）
- `ClearGhostTilemap()` 在渲染前调用，确保清除旧预览
- 这个改动使得每次 `LockPosition` 都会正确显示锁定位置的 1+8 预览

### 9.4 UnlockPosition 不需要修改

`UnlockPosition` 只设置 `_isLocked = false`，解锁后下一帧 `UpdateHoePreview` 会正常执行（不再被 `_isLocked` 跳过），自动恢复鼠标跟随的实时预览。

### 9.5 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-15 | 锄头模式下 LockPosition 后 GhostTilemap 显示锁定位置的 1+8 预览 | LockPosition 内部渲染逻辑 |

---

## 十、模块 I：CropController — 作物底部对齐

> 涉及改进点：P5
> 独立模块，与队列系统无关

### 10.1 现状

`UpdateVisuals()` 只设置 Sprite 和颜色，没有做位置对齐。不同生长阶段 Sprite 高度不同，作物视觉中心会偏移。

```csharp
public void UpdateVisuals()
{
    if (spriteRenderer == null) return;
    Sprite sprite = GetCurrentSprite();
    if (sprite != null) spriteRenderer.sprite = sprite;
    // ... 颜色设置 ...
    // ← 缺少底部对齐
}
```

### 10.2 参考实现

`TreeController.AlignSpriteBottom()` 的逻辑：
```csharp
float spriteBottomOffset = spriteBounds.min.y;
localPos.y = -spriteBottomOffset;
spriteRenderer.transform.localPosition = localPos;
```
原理：将 Sprite 的底部边缘对齐到 GameObject 的原点（即格子中心），确保无论 Sprite 多大，底部始终在格子中心。

### 10.3 改造方案

在 `CropController` 中新增 `AlignSpriteBottom()` 方法：

```
AlignSpriteBottom():
  if (spriteRenderer == null || spriteRenderer.sprite == null) return
  var bounds = spriteRenderer.sprite.bounds
  var localPos = spriteRenderer.transform.localPosition
  localPos.y = -bounds.min.y
  spriteRenderer.transform.localPosition = localPos
```

调用时机：在 `UpdateVisuals()` 末尾，每次切换 Sprite 后调用：

```
UpdateVisuals():
  // ... 原有 Sprite 设置和颜色逻辑 ...
  AlignSpriteBottom()  // 🔴 P5：每次切换 Sprite 后重新对齐底部
```

### 10.4 影响范围

- `UpdateVisuals()` 在以下场景被调用：
  - `Initialize()` 初始化时
  - `Grow()` 生长阶段变化时
  - `SetWithered()` 枯萎时
  - `ResetForReHarvest()` 可重复收获作物重置时
  - `Load()` 存档加载时
- 所有场景都会自动触发底部对齐，无需额外处理

### 10.5 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-16 | 每个生长阶段的作物 Sprite 底部中心对齐格子中心 | `AlignSpriteBottom` 在 `UpdateVisuals` 末尾调用 |

---

## 十一、完整交互矩阵

### 11.1 左键点击入队矩阵

| 编号 | 手持物品 | 鼠标位置 | 预览状态 | 队列状态 | 行为 | 关联属性 |
|------|---------|---------|---------|---------|------|---------|
| S1 | 任意 | 成熟/枯萎成熟作物（同层级） | - | 任意 | 入队 Harvest | CP-5 |
| S2 | 任意 | 成熟作物（不同层级） | - | 任意 | 不触发收获，走 S3-S8 | CP-6 |
| S3 | 锄头 | 可耕地 | Valid | 空/非空 | 入队 Till | CP-9 |
| S4 | 锄头 | 不可耕地 | Invalid | 任意 | 无操作 | CP-9 |
| S5 | 浇水壶 | 可浇水地 | Valid | 空/非空 | 入队 Water | CP-9 |
| S6 | 浇水壶 | 不可浇水地 | Invalid | 任意 | 无操作 | CP-9 |
| S7 | 种子 | 可种植地 | Valid | 空/非空 | 入队 PlantSeed | CP-9 |
| S8 | 种子 | 不可种植地 | Invalid | 任意 | 无操作 | CP-9 |
| S9 | 镐子/斧头/武器 | 无成熟作物 | - | 任意 | 走原有工具/武器逻辑（不入队） | CP-19 |
| S10 | 空手 | 无成熟作物 | - | 任意 | 无操作 | - |
| S11 | 任意 | 已入队的同一格子 | - | 非空 | 忽略（防重复） | CP-2 |

### 11.2 队列执行期间的点击矩阵

| 编号 | 触发条件 | 新点击目标 | 行为 | 关联属性 |
|------|---------|-----------|------|---------|
| E1 | `_isExecutingFarming` = true | 有效农田目标 | 入队（替代旧 CacheFarmInput） | CP-11 |
| E2 | `isPerformingAction` = true | 有效农田目标 | 入队（替代旧 CacheFarmInput） | CP-11 |
| E3 | 正在导航 | 同一格子 | 忽略（防重复） | CP-2 |
| E4 | 正在导航 | 不同有效格子 | 入队 | CP-1 |
| E5 | 队列暂停（面板打开） | 任意 | 不处理（`uiOpen` return） | CP-13 |
| E6 | `_isExecutingFarming` = true | 成熟作物（同层级） | 入队 Harvest | CP-5/CP-11 |

### 11.3 中断矩阵

| 触发 | 队列行为 | 导航行为 | 预览行为 | 锁定行为 | 关联属性 |
|------|---------|---------|---------|---------|---------|
| WASD 移动 | 清空 | 取消 | 恢复跟随 | ForceUnlock | CP-12 |
| ESC | 清空 | 取消 | 恢复跟随 | 解锁 | CP-3 |
| 切换快捷栏 | 清空 | 取消 | 隐藏/切换模式 | 解锁 | CP-3 |
| 打开面板 | 暂停（不清空） | 取消当前导航 | 保持锁定 | 保持 | CP-4/CP-13 |
| 关闭面板 | 恢复执行 | 重新开始 | 锁定到下一个目标 | 重新锁定 | CP-13 |
| 种子用完 | 丢弃剩余种子操作 | 取消（如当前是种子） | 视队列剩余而定 | 视队列剩余而定 | CP-10 |
| 动画期间切换工具栏 | 清空 | - | 切换模式 | 解锁 | CP-3 |

### 11.4 预览状态矩阵

| 队列状态 | 预览显示 | 关联属性 |
|---------|---------|---------|
| 队列为空，无操作 | 鼠标跟随实时预览 | CP-14 |
| 队列执行中（近距离） | 锁定到当前目标位置（含 1+8） | CP-14/CP-15 |
| 队列执行中（远距离导航） | 锁定到当前目标位置（含 1+8） | CP-14/CP-15 |
| 队列暂停（面板打开） | 保持锁定在暂停时的位置 | CP-13 |
| 队列刚清空（中断后） | 解锁，恢复鼠标跟随 | CP-14 |

### 11.5 OnActionComplete 分支矩阵

| currentAction | IsToolAction | 鼠标按住 | 行为 | 关联属性 |
|--------------|-------------|---------|------|---------|
| Collect（收获） | false | - | 专用分支：执行 Harvest → ProcessNextAction | CP-17 |
| Crush（锄地） | true | true | OnFarmActionAnimationComplete → ProcessNextAction | CP-18 |
| Crush（锄地） | true | false | OnFarmActionAnimationComplete → ProcessNextAction | CP-18 |
| Watering | true | true | OnFarmActionAnimationComplete → ProcessNextAction | CP-18 |
| Watering | true | false | OnFarmActionAnimationComplete → ProcessNextAction | CP-18 |
| Slice（斧头） | true | true | 保持原有长按：EndAction(true) → StartAction | CP-19 |
| Slice（斧头） | true | false | 保持原有松开行为 | CP-19 |
| Pierce（镐子） | true | true | 保持原有长按：EndAction(true) → StartAction | CP-19 |
| Pierce（镐子） | true | false | 保持原有松开行为 | CP-19 |

### 11.6 队列二次验证矩阵

| 操作类型 | 验证内容 | 失败行为 | 关联属性 |
|---------|---------|---------|---------|
| Till | 无额外验证（LockPosition 时已验证） | - | - |
| Water | 无额外验证 | - | - |
| PlantSeed | 手持仍为种子 && slot.amount > 0 | 丢弃该请求，取下一个 | CP-10 |
| Harvest | targetCrop != null && CanInteract() | 丢弃该请求，取下一个 | CP-7 |

---

## 十二、正确性属性与保证关系

| 编号 | 属性 | 描述 | 保证模块 |
|------|------|------|---------|
| CP-1 | 队列 FIFO 顺序 | 入队顺序 = 执行顺序 | 模块 C（Queue 数据结构） |
| CP-2 | 防重复入队 | 同一 (layerIndex, cellPos) 不会在队列中出现两次 | 模块 A（_queuedPositions HashSet） |
| CP-3 | 队列清空完整性 | ClearActionQueue 后队列为空、防重复集合为空、_isProcessingQueue = false | 模块 C（ClearActionQueue） |
| CP-4 | 暂停/恢复一致性 | 暂停前队列内容 = 恢复后队列内容 | 模块 F（_isQueuePaused 标志） |
| CP-5 | 收获最高优先级 | 鼠标位置有可收获作物（同层级）时，无论手持什么，左键触发收获 | 模块 B（TryDetectAndEnqueueHarvest 在工具检测之前） |
| CP-6 | 收获层级隔离 | 只能收获玩家当前层级的作物 | 模块 B（layerIndex 过滤） |
| CP-7 | 收获二次验证 | ProcessNextAction 执行 Harvest 前重新检查 CanInteract | 模块 C（ProcessNextAction 二次验证） |
| CP-8 | 右键不触发收获 | 右键点击作物不再触发收获逻辑 | 模块 E（CropController 过滤） |
| CP-9 | 预览有效性前置 | 只有预览 Valid 时才入队 | 模块 B（TryEnqueueFarmTool/TryEnqueueSeed 检查 IsValid） |
| CP-10 | 种子用完自动跳过 | 种子 remaining=0 时，队列中剩余 PlantSeed 请求被丢弃 | 模块 C（HasSeedRemaining 检查） |
| CP-11 | 动画期间可入队 | 执行期间/动画期间的点击走入队逻辑 | 模块 B（保护分支调用 TryEnqueueFromCurrentInput） |
| CP-12 | WASD 中断优先级 | WASD 输入时，无论 lockManager 状态，队列被清空 | 模块 D（中断逻辑在 lockManager 检查之前） |
| CP-13 | 面板暂停不丢失 | 面板打开暂停队列，关闭后恢复执行 | 模块 F（_isQueuePaused + 状态变化检测） |
| CP-14 | 预览跟随/锁定切换 | 队列为空时预览跟随鼠标，执行时锁定到当前目标 | 模块 C（ProcessNextAction 中 LockPosition/UnlockPosition） |
| CP-15 | LockPosition 渲染 1+8 | 锄头模式下 LockPosition 后 GhostTilemap 显示锁定位置的 1+8 | 模块 H（LockPosition 内部渲染） |
| CP-16 | 作物底部对齐 | 每个生长阶段的作物 Sprite 底部中心对齐格子中心 | 模块 I（AlignSpriteBottom） |
| CP-17 | Collect 专用分支 | Collect 动画完成走专用分支，不进入 IsToolAction 长按逻辑 | 模块 G（OnActionComplete Collect 拦截） |
| CP-18 | Crush/Watering 不长按 | 锄头/浇水动画完成后走队列出队 | 模块 G（OnFarmActionAnimationComplete） |
| CP-19 | Slice/Pierce 保持原有 | 斧头/镐子保持原有长按行为 | 模块 G（else 分支不变） |

---

## 十三、测试框架

本补丁为 Unity C# 项目，核心逻辑涉及 MonoBehaviour 和 Unity API（Physics2D、Tilemap 等），不适合纯单元测试。

验证策略：
1. 编译验证：`getDiagnostics` 检查所有修改文件，0 错误 0 警告
2. 逻辑审查：代码审查确认每个正确性属性（CP-1~CP-19）在实现中被满足
3. 运行时验证：用户在 Unity Editor 中按交互矩阵（第十一章）逐项手动测试

后续如需自动化测试，建议将队列逻辑（EnqueueAction/ProcessNextAction/ClearActionQueue）抽离为非 MonoBehaviour 的纯 C# 类，使用 NUnit 进行单元测试。

---

## 十四、涉及文件汇总

| 文件 | 修改内容 | 关联模块 | 关联改进点 |
|------|---------|---------|-----------|
| `GameInputManager.cs` | 队列数据结构、EnqueueAction、ProcessNextAction、ClearActionQueue、ExecuteFarmAction、HandleUseCurrentTool 全面改造（收获检测+工具入队+保护分支替换）、HandleMovement WASD 中断、HandleRightClickAutoNav 作物过滤、面板暂停/恢复、OnCollectAnimationComplete、OnFarmActionAnimationComplete、废弃旧方法 | A/B/C/D/E/F | P1/P2a/P2b/V1/V2/V4/V5/V6 |
| `PlayerInteraction.cs` | OnActionComplete Collect 专用分支、农田工具分支改为队列出队、松开分支改为队列出队 | G | V3/CP-17/CP-18 |
| `FarmToolPreview.cs` | LockPosition 增加 1+8 GhostTilemap 渲染 | H | P3 |
| `CropController.cs` | 新增 AlignSpriteBottom、UpdateVisuals 末尾调用 | I | P5 |
