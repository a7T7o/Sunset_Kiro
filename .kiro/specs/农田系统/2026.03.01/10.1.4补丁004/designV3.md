# 补丁004V2 设计文档（designV3）

> 基于用户验收后四点纠正 + 预览情况矩阵分析
> 前置文档：`预览情况矩阵分析.md`、`design.md`（V1 设计）
> 创建时间：2026-02-21
> requirements 沿用现有 `requirements.md`

---

## 目录

1. [问题总结与修复目标](#一问题总结与修复目标)
2. [模块 H：ClearActionQueue 执行状态保护](#二模块-h)
3. [模块 I：PromoteToExecutingPreview 时机修正](#三模块-i)
4. [模块 J：canTill=false 红色反馈](#四模块-j)
5. [模块 K：Sorting Order 确认](#五模块-k)
6. [完整数据流（V2 修正版）](#六完整数据流)
7. [正确性属性汇总](#七正确性属性汇总)
8. [涉及文件汇总](#八涉及文件汇总)

---

## 一、问题总结与修复目标

V1 代码修改（design.md 的模块 A-G）已完成并通过编译，但用户验收发现以下问题：

| 编号 | 问题 | 修复模块 |
|------|------|---------|
| P1 | WASD 中断后动画播完不创建耕地 + 执行预览残留 | 模块 H |
| P2 | `PromoteToExecutingPreview` 在出队时调用，导航途中就被保护 | 模块 I |
| P3 | 鼠标在已有耕地上无任何视觉反馈 | 模块 J |
| P4 | Ghost 层与实际层 Sorting Order 需确认 | 模块 K |

设计约束：
- 不修改 V1 已完成的模块 A-G 的核心架构
- 只在 V1 基础上做增量修复
- 新增并兼容，不破坏已有逻辑

---

## 二、模块 H：ClearActionQueue 执行状态保护

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`ClearActionQueue`

### 2.1 现状问题

```csharp
// 当前 ClearActionQueue（V1 修改后）：
public void ClearActionQueue()
{
    _farmActionQueue.Clear();
    _queuedPositions.Clear();
    _isProcessingQueue = false;
    _isExecutingFarming = false;          // ← 问题：动画还在播放
    _currentHarvestTarget = null;
    _currentProcessingRequest = default;   // ← 问题：动画完成后找不到位置
    _pendingTileUpdate = null;             // ← 问题：tile 创建不触发
    _tileUpdateTriggered = false;
    FarmToolPreview.Instance?.ClearAllQueuePreviews();
}
```

WASD 中断时调用此方法，把正在执行的动画操作的状态也清了。

### 2.2 修复方案

在 `ClearActionQueue` 中检查 `_isExecutingFarming`：如果当前有动画在执行，保留执行相关状态。

### 2.3 改动后伪代码

```csharp
public void ClearActionQueue()
{
    _farmActionQueue.Clear();
    _queuedPositions.Clear();
    _isProcessingQueue = false;
    _currentHarvestTarget = null;  // 收获目标可以安全清空（收获动画中不会被 WASD 中断，因为 ToolActionLock）
    
    // 🔴 V2 模块H：如果当前有动画在执行，保留执行状态
    if (!_isExecutingFarming)
    {
        // 没有动画在执行，全部清空
        _currentProcessingRequest = default;
        _pendingTileUpdate = null;
        _tileUpdateTriggered = false;
        _isExecutingFarming = false;
    }
    // else: 动画在执行中，保留 _pendingTileUpdate / _currentProcessingRequest / _isExecutingFarming
    // 动画完成后 OnFarmActionAnimationComplete 会正常清理
    
    FarmToolPreview.Instance?.ClearAllQueuePreviews();
}
```

### 2.4 关键分析：_currentHarvestTarget 的处理

收获动画期间 `ToolActionLockManager` 锁定输入，WASD 不会触发 `ClearActionQueue`。所以 `_currentHarvestTarget = null` 是安全的。

但为了防御性编程，也可以把 `_currentHarvestTarget` 放进 `if (!_isExecutingFarming)` 块中。选择更安全的方案：

```csharp
if (!_isExecutingFarming)
{
    _currentProcessingRequest = default;
    _pendingTileUpdate = null;
    _tileUpdateTriggered = false;
    _isExecutingFarming = false;
    _currentHarvestTarget = null;
}
```

### 2.5 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-H1 | WASD 中断时，正在执行的动画操作不受影响 | `_isExecutingFarming` 为 true 时保留执行状态 |
| CP-H2 | 动画播完后 tile 正常创建 | `_pendingTileUpdate` 未被清空，Update 中进度监听正常触发 |
| CP-H3 | 动画完成后执行预览正常清除 | `_currentProcessingRequest` 未被清空，`RemoveExecutingPreview` 用正确的 cellPos |
| CP-H4 | 队列中等待的操作仍被正常清空 | `_farmActionQueue.Clear()` 和 `ClearAllQueuePreviews()` 正常执行 |


---

## 三、模块 I：PromoteToExecutingPreview 时机修正

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`ProcessNextAction`、`ExecuteFarmAction`

### 3.1 现状问题

```csharp
// ProcessNextAction 中（V1 修改后）：
_isExecutingFarming = true;
FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);  // ← 出队时就提升

if (distance <= farmToolReach)
{
    ExecuteFarmAction(request);  // 近距离直接执行
}
else
{
    StartFarmingNavigation(request.worldPos, () => {
        ExecuteFarmAction(request);  // 远距离导航后执行
    });
}
```

问题：远距离操作时，出队后还在导航中，`PromoteToExecutingPreview` 已经把预览提升为执行预览。WASD 打断导航时，这个操作的预览被保护不被清除，但操作本身已经不会执行了。

### 3.2 修复方案

将 `PromoteToExecutingPreview` 从 `ProcessNextAction` 移到 `ExecuteFarmAction` 的开头。

同时，`_isExecutingFarming = true` 也应该移到 `ExecuteFarmAction`，因为导航途中不算"正在执行"。

### 3.3 改动后伪代码

ProcessNextAction 中移除：
```csharp
// 删除这两行：
// _isExecutingFarming = true;
// FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
```

ExecuteFarmAction 开头新增：
```csharp
private void ExecuteFarmAction(FarmActionRequest request)
{
    // 🔴 V2 模块I：动画开始的瞬间才提升为执行预览
    _isExecutingFarming = true;
    FarmToolPreview.Instance?.PromoteToExecutingPreview(request.cellPos);
    
    // ... 原有的 switch 分支 ...
}
```

### 3.4 PlantSeed 分支特殊处理

PlantSeed 无动画，在 `ExecuteFarmAction` 中直接执行后立即 `RemoveExecutingPreview` + `_isExecutingFarming = false`。这个流程不受影响——`PromoteToExecutingPreview` 在 `ExecuteFarmAction` 开头调用，紧接着 PlantSeed 分支执行完就 `RemoveExecutingPreview`，时序正确。

### 3.5 导航途中 WASD 打断的行为变化

V1：出队 → Promote → 导航 → WASD → ClearActionQueue → 执行预览被保护（但操作不会执行）→ 预览残留
V2：出队 → 导航 → WASD → ClearActionQueue → 操作还在队列预览中 → 被正常清空 ✅

### 3.6 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-I1 | 执行预览只在动画真正开始时才被保护 | `PromoteToExecutingPreview` 在 `ExecuteFarmAction` 中调用 |
| CP-I2 | 导航途中被打断的操作随队列一起清空 | 操作还在队列预览中，`ClearAllQueuePreviews` 正常清除 |
| CP-I3 | `_isExecutingFarming` 只在动画开始时设为 true | 移到 `ExecuteFarmAction` 开头 |
| CP-I4 | 近距离操作行为不变 | 近距离时 `ProcessNextAction` 直接调用 `ExecuteFarmAction`，时序一致 |

---

## 四、模块 J：canTill=false 红色反馈

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`

### 4.1 现状问题

鼠标在已有耕地上时：
- `canTill = false` → `if (canTill && ...)` 不进入 → ghostTilemap 无 tile
- `cursorRenderer.enabled = false`（耕地模式不显示光标）
- shader 叠加色设置了红色但无载体 → 什么都不显示

### 4.2 修复方案：启用 cursorRenderer 显示红色方框

当 `canTill = false` 且 `!canClearWithered`（不是枯萎作物清除场景）时，启用 cursorRenderer 并设置红色。

这是最简单的方案，不需要修改 ghost tile 逻辑。

### 4.3 改动后伪代码

在 `UpdateHoePreview` 的 `if (canTill && ...)` 块之后，`else` 块中：

```csharp
else
{
    _currentGhostTileData?.Clear();
    
    // 🔴 V2 模块J：canTill=false 时显示红色方框反馈
    if (cursorRenderer != null && !canClearWithered)
    {
        cursorRenderer.enabled = true;
        UpdateCursor(layerIndex, cellPos);
    }
}
```

同时需要确认：`UpdateCursor` 方法会根据 `currentState`（此时为 Invalid）设置红色。

### 4.4 枯萎作物场景

`canClearWithered = true` 时，`isValid = true`，`currentState = Valid`。这种情况下 ghost 预览应该显示绿色反馈。当前代码在 `canTill = false` 时不进入 ghost tile 渲染，但枯萎作物清除不需要 1+8 预览（只需要中心块反馈）。

方案：枯萎作物场景也启用 cursorRenderer，但显示绿色（因为 `currentState = Valid`）。

```csharp
else
{
    _currentGhostTileData?.Clear();
    
    // 🔴 V2 模块J：canTill=false 时显示光标反馈
    if (cursorRenderer != null)
    {
        cursorRenderer.enabled = true;
        UpdateCursor(layerIndex, cellPos);
        // UpdateCursor 会根据 currentState 设置颜色：Valid=绿色，Invalid=红色
    }
}
```

### 4.5 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-J1 | 鼠标在已有耕地上时显示红色方框 | `canTill=false` + `!canClearWithered` → cursorRenderer 启用 + Invalid 红色 |
| CP-J2 | 鼠标在枯萎作物上时显示绿色方框 | `canClearWithered=true` → cursorRenderer 启用 + Valid 绿色 |
| CP-J3 | 可锄地时仍然不显示光标（保持 V1 行为） | `canTill=true` 分支中 `cursorRenderer.enabled = false` 不变 |

---

## 五、模块 K：Sorting Order 确认

> 涉及文件：`FarmToolPreview.cs`（EnsureComponents 方法）
> 可能涉及：场景中的 Tilemap 配置

### 5.1 需要确认的内容

Ghost 预览层（`ghostTilemap`）的 Sorting Order 必须高于实际耕地层（`farmlandBorderTilemap`），否则差异化 tile 无法正确遮盖实际层。

### 5.2 处理方式

在代码中读取 `EnsureComponents` 方法确认 ghostTilemap 的 Sorting Order 设置。如果不够高，调整。

这是一个验证性任务，不一定需要代码修改。

### 5.3 正确性属性

| 编号 | 属性 | 保证方式 |
|------|------|---------|
| CP-K1 | ghostTilemap Sorting Order > farmlandBorderTilemap Sorting Order | 代码/场景配置确认 |


---

## 六、完整数据流（V2 修正版）

### 6.1 耕地完整流程（V2）

```
1. 鼠标移动 → UpdateHoePreview
   ├─ canTill=true：
   │   ├─ GetPreviewTiles → 差异化过滤 → ghostTilemap 显示差异 tile
   │   ├─ cursorRenderer.enabled = false（不显示方框）
   │   └─ 缓存 CurrentGhostTileData
   └─ canTill=false：
       ├─ ghostTilemap 无 tile
       ├─ 🔴 V2：cursorRenderer.enabled = true → 红色/绿色方框
       └─ _currentGhostTileData.Clear()

2. 玩家点击 → TryEnqueueFarmTool → EnqueueAction
   ├─ 复制 CurrentGhostTileData 快照
   ├─ AddQueuePreview → queuePreviewTilemap 显示队列预览
   └─ ghost 继续跟随鼠标

3. ProcessNextAction 出队
   ├─ 🔴 V2：不再调用 PromoteToExecutingPreview（移到 ExecuteFarmAction）
   ├─ 🔴 V2：不再设置 _isExecutingFarming = true（移到 ExecuteFarmAction）
   ├─ 距离判断：近距离 → 直接 ExecuteFarmAction
   └─ 远距离 → StartFarmingNavigation → 到达后 ExecuteFarmAction

4. ExecuteFarmAction（动画开始瞬间）
   ├─ 🔴 V2：_isExecutingFarming = true
   ├─ 🔴 V2：PromoteToExecutingPreview(cellPos)
   ├─ FaceTarget → RequestAction → 开始动画
   └─ _pendingTileUpdate = request

5. WASD 中断（如果在导航途中）
   ├─ ClearActionQueue：
   │   ├─ 清空队列
   │   ├─ 🔴 V2：_isExecutingFarming=false → 全部清空（导航途中没有动画在执行）
   │   └─ ClearAllQueuePreviews → 操作还在队列预览中，被正常清空 ✅
   └─ 导航取消

6. WASD 中断（如果在动画执行中）
   ├─ ClearActionQueue：
   │   ├─ 清空队列
   │   ├─ 🔴 V2：_isExecutingFarming=true → 保留执行状态
   │   └─ ClearAllQueuePreviews → 跳过执行预览
   ├─ 动画继续播放
   ├─ Update 进度监听 → _pendingTileUpdate 未被清空 → tile 正常创建 ✅
   └─ OnFarmActionAnimationComplete → RemoveExecutingPreview(正确的 cellPos) ✅

7. 动画正常完成
   ├─ TillAt → UpdateBorderAt（数据层落地）
   ├─ RemoveExecutingPreview(cellPos)
   └─ ProcessNextAction（取下一个）
```

### 6.2 关键时序对比

| 场景 | V1 行为 | V2 行为 |
|------|---------|---------|
| 近距离执行 | 出队→Promote→执行→完成 | 出队→执行(Promote)→完成 ✅ |
| 远距离执行 | 出队→Promote→导航→执行→完成 | 出队→导航→执行(Promote)→完成 ✅ |
| 导航中 WASD | 出队→Promote→导航→WASD→执行预览残留 ❌ | 出队→导航→WASD→队列预览清空 ✅ |
| 动画中 WASD | 出队→Promote→执行→WASD→tile不创建+预览残留 ❌ | 出队→执行(Promote)→WASD→tile正常创建+预览正常清除 ✅ |
| 已有耕地上 | 什么都不显示 ❌ | 红色方框 ✅ |

---

## 七、正确性属性汇总

### V2 新增属性

| 编号 | 属性 | 模块 |
|------|------|------|
| CP-H1 | WASD 中断时正在执行的动画操作不受影响 | H |
| CP-H2 | 动画播完后 tile 正常创建 | H |
| CP-H3 | 动画完成后执行预览正常清除 | H |
| CP-H4 | 队列中等待的操作仍被正常清空 | H |
| CP-I1 | 执行预览只在动画真正开始时才被保护 | I |
| CP-I2 | 导航途中被打断的操作随队列一起清空 | I |
| CP-I3 | `_isExecutingFarming` 只在动画开始时设为 true | I |
| CP-I4 | 近距离操作行为不变 | I |
| CP-J1 | 鼠标在已有耕地上时显示红色方框 | J |
| CP-J2 | 鼠标在枯萎作物上时显示绿色方框 | J |
| CP-J3 | 可锄地时仍然不显示光标 | J |
| CP-K1 | ghostTilemap Sorting Order > farmlandBorderTilemap | K |

### V1 保留属性（不变）

CP-A1~A2、CP-B1~B3、CP-C1~C4、CP-D1~D3、CP-E1~E3、CP-F1~F2、CP-G1~G6

---

## 八、涉及文件汇总

| 文件 | 改动类型 | 涉及模块 |
|------|----------|----------|
| `GameInputManager.cs` | 修改 | H（ClearActionQueue 保护）、I（PromoteToExecutingPreview 时机） |
| `FarmToolPreview.cs` | 修改 | J（canTill=false 红色反馈）、K（Sorting Order 确认） |
