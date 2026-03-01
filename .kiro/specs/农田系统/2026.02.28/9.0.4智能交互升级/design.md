# 9.0.4 智能交互升级 - 设计文档

## 架构概述

### 核心设计变更

**9.0.3 架构（短视模式）**：
```
IsValid = 逻辑合法 AND 物理合法 AND 距离够
```

**9.0.4 架构（智能模式）**：
```
IsValid = 逻辑合法 AND 物理合法
IsInRange = 距离够
```

### 职责分离

| 组件 | 职责 |
|------|------|
| FarmToolPreview | 视觉层：计算 IsValid、IsInRange，显示光标颜色 |
| GameInputManager | 决策层：根据 IsValid + IsInRange 决定立即执行或导航 |
| PlayerAutoNavigator | 执行层：导航到目标，执行回调 |

## 组件设计

### 0. FarmingSnapshot 快照机制（锐评006补充）

#### 0.1 设计目标

防止"种瓜得豆"问题：导航期间物品可能被消耗、移动或替换，到达后需要校验物品状态。

#### 0.2 快照数据结构

```csharp
/// <summary>
/// 农田操作快照 - 记录导航开始时的物品状态
/// 用于防止"种瓜得豆"问题
/// </summary>
private struct FarmingSnapshot
{
    public int itemId;      // 物品 ID
    public int slotIndex;   // 槽位索引
    public int count;       // 数量
    public bool isValid;    // 快照是否有效
    
    public static FarmingSnapshot Invalid => new FarmingSnapshot { isValid = false };
    
    public static FarmingSnapshot Create(int itemId, int slotIndex, int count)
    {
        return new FarmingSnapshot
        {
            itemId = itemId,
            slotIndex = slotIndex,
            count = count,
            isValid = true
        };
    }
}

private FarmingSnapshot _farmingSnapshot = FarmingSnapshot.Invalid;
```

#### 0.3 快照校验逻辑

```csharp
/// <summary>
/// 校验快照是否仍然有效
/// </summary>
private bool ValidateSnapshot()
{
    if (!_farmingSnapshot.isValid) return false;
    if (inventory == null || database == null) return false;
    
    var slot = inventory.GetSlot(_farmingSnapshot.slotIndex);
    
    // 校验：槽位非空 && 物品 ID 匹配 && 数量足够
    return !slot.IsEmpty && 
           slot.itemId == _farmingSnapshot.itemId && 
           slot.count >= _farmingSnapshot.count;
}

/// <summary>
/// 清除快照
/// </summary>
private void ClearSnapshot()
{
    _farmingSnapshot = FarmingSnapshot.Invalid;
}
```

#### 0.4 使用场景

| 场景 | 是否需要快照 | 原因 |
|------|-------------|------|
| 锄头远距离锄地 | 否 | 锄头不消耗，无状态变化风险 |
| 水壶远距离浇水 | 否 | 水壶不消耗，无状态变化风险 |
| 种子远距离种植 | 是 | 种子会消耗，导航期间可能被其他操作消耗 |

### 1. FarmToolPreview 修改

#### 1.1 新增公开属性

```csharp
/// <summary>
/// 当前预览是否有效（逻辑合法 + 物理合法，不含距离）
/// </summary>
public bool IsValid => currentState == FarmPreviewState.Valid;

/// <summary>
/// 当前目标是否在使用距离内
/// </summary>
public bool IsInRange { get; private set; }

/// <summary>
/// 当前光标位置（格子中心世界坐标）
/// </summary>
public Vector3 CurrentCursorPos { get; private set; }

/// <summary>
/// 当前目标的格子坐标
/// </summary>
public Vector3Int CurrentCellPos { get; private set; }

/// <summary>
/// 当前目标的楼层索引
/// </summary>
public int CurrentLayerIndex { get; private set; }
```

#### 1.2 UpdateHoePreview 修改

```csharp
public void UpdateHoePreview(int layerIndex, Vector3Int cellPos, Transform playerTransform = null, float reach = 1.5f)
{
    // ... 现有逻辑 ...
    
    // 🔥 Step 4: 检查是否可以锄地
    bool canTill = FarmTileManager.Instance != null && 
                   FarmTileManager.Instance.CanTillAt(layerIndex, cellPos);
    
    // 🔥 9.0.4 修改：IsValid 不再包含距离判断
    bool isValid = !hasObstacle && canTill;
    
    // 🔥 9.0.4 新增：单独记录距离状态
    IsInRange = withinReach;
    
    // 🔥 9.0.4 新增：记录当前位置信息
    CurrentCursorPos = cellCenter;
    CurrentCellPos = cellPos;
    CurrentLayerIndex = layerIndex;
    
    // 更新状态
    currentState = isValid ? FarmPreviewState.Valid : FarmPreviewState.Invalid;
    
    // ... 后续逻辑不变 ...
}
```

#### 1.3 UpdateWateringPreview 修改

同 UpdateHoePreview，移除 `&& withinReach` 条件。

#### 1.4 UpdateSeedPreview 修改

同 UpdateHoePreview，移除 `&& withinReach` 条件。

### 2. GameInputManager 修改

#### 2.1 TryTillSoil 重构

```csharp
private bool TryTillSoil(Vector3 worldPosition)
{
    var farmPreview = FarmGame.Farm.FarmToolPreview.Instance;
    
    // 🔥 Step 1: 检查目标是否有效（不含距离）
    if (farmPreview == null || !farmPreview.IsValid)
    {
        if (showDebugInfo)
            Debug.Log("[GameInputManager] 锄地失败：目标无效");
        return false;
    }
    
    // 🔥 Step 2: 获取目标位置
    Vector3 targetPos = farmPreview.CurrentCursorPos;
    int layerIndex = farmPreview.CurrentLayerIndex;
    Vector3Int cellPos = farmPreview.CurrentCellPos;
    
    // 🔥 Step 3: 距离分流
    if (farmPreview.IsInRange)
    {
        // A. 近距离 → 立即执行
        return ExecuteTillSoil(layerIndex, cellPos);
    }
    else
    {
        // B. 远距离 → 导航后执行
        StartFarmingNavigation(targetPos, () =>
        {
            // 二次检查：到达后重新验证
            if (farmPreview.IsValid && farmPreview.IsInRange)
            {
                ExecuteTillSoil(layerIndex, cellPos);
            }
            else if (showDebugInfo)
            {
                Debug.Log("[GameInputManager] 导航到达但目标已失效");
            }
        });
        return true; // 导航已启动
    }
}

/// <summary>
/// 执行锄地动作（纯逻辑，不含距离检查）
/// </summary>
private bool ExecuteTillSoil(int layerIndex, Vector3Int cellPos)
{
    var farmTileManager = FarmGame.Farm.FarmTileManager.Instance;
    if (farmTileManager == null) return false;
    
    if (!farmTileManager.CanTillAt(layerIndex, cellPos))
        return false;
    
    return farmTileManager.CreateTile(layerIndex, cellPos);
}
```

#### 2.2 TryWaterTile 重构

```csharp
private bool TryWaterTile(Vector3 worldPosition)
{
    var farmPreview = FarmGame.Farm.FarmToolPreview.Instance;
    
    // 🔥 Step 1: 检查目标是否有效
    if (farmPreview == null || !farmPreview.IsValid)
    {
        if (showDebugInfo)
            Debug.Log("[GameInputManager] 浇水失败：目标无效");
        return false;
    }
    
    // 🔥 Step 2: 获取目标位置
    Vector3 targetPos = farmPreview.CurrentCursorPos;
    int layerIndex = farmPreview.CurrentLayerIndex;
    Vector3Int cellPos = farmPreview.CurrentCellPos;
    
    // 🔥 Step 3: 距离分流
    if (farmPreview.IsInRange)
    {
        // A. 近距离 → 立即执行
        return ExecuteWaterTile(layerIndex, cellPos);
    }
    else
    {
        // B. 远距离 → 导航后执行
        StartFarmingNavigation(targetPos, () =>
        {
            // 二次检查
            if (farmPreview.IsValid && farmPreview.IsInRange)
            {
                ExecuteWaterTile(layerIndex, cellPos);
            }
        });
        return true;
    }
}

/// <summary>
/// 执行浇水动作
/// </summary>
private bool ExecuteWaterTile(int layerIndex, Vector3Int cellPos)
{
    var farmTileManager = FarmGame.Farm.FarmTileManager.Instance;
    if (farmTileManager == null) return false;
    
    float currentHour = TimeManager.Instance != null ? TimeManager.Instance.GetHour() : 0f;
    return farmTileManager.SetWatered(layerIndex, cellPos, currentHour);
}
```

#### 2.3 TryPlantSeed 重构

```csharp
private bool TryPlantSeed(SeedData seedData)
{
    if (seedData == null) return false;
    
    var farmPreview = FarmGame.Farm.FarmToolPreview.Instance;
    
    // 🔥 Step 1: 检查目标是否有效
    if (farmPreview == null || !farmPreview.IsValid)
    {
        if (showDebugInfo)
            Debug.Log("[GameInputManager] 种植失败：目标无效");
        return false;
    }
    
    // 🔥 Step 2: 获取目标位置
    Vector3 targetPos = farmPreview.CurrentCursorPos;
    int layerIndex = farmPreview.CurrentLayerIndex;
    Vector3Int cellPos = farmPreview.CurrentCellPos;
    
    // 🔥 Step 3: 距离分流
    if (farmPreview.IsInRange)
    {
        // A. 近距离 → 立即执行
        return ExecutePlantSeed(seedData, layerIndex, cellPos);
    }
    else
    {
        // B. 远距离 → 导航后执行
        // 🔥 注意：需要缓存 seedData，因为导航期间可能切换物品
        var cachedSeedData = seedData;
        StartFarmingNavigation(targetPos, () =>
        {
            // 二次检查：验证手持物品仍是同一种子
            if (farmPreview.IsValid && farmPreview.IsInRange && IsHoldingSameSeed(cachedSeedData))
            {
                ExecutePlantSeed(cachedSeedData, layerIndex, cellPos);
            }
        });
        return true;
    }
}

/// <summary>
/// 检查当前手持物品是否是指定种子
/// </summary>
private bool IsHoldingSameSeed(SeedData expectedSeed)
{
    if (inventory == null || database == null || hotbarSelection == null)
        return false;
    
    int idx = Mathf.Clamp(hotbarSelection.selectedIndex, 0, InventoryService.HotbarWidth - 1);
    var slot = inventory.GetSlot(idx);
    
    if (slot.IsEmpty) return false;
    
    var itemData = database.GetItemByID(slot.itemId);
    return itemData is SeedData seed && seed.itemID == expectedSeed.itemID;
}
```

#### 2.4 新增：StartFarmingNavigation

```csharp
/// <summary>
/// 启动农田工具导航
/// </summary>
/// <param name="targetPos">目标位置（格子中心）</param>
/// <param name="onArrived">到达后的回调</param>
private void StartFarmingNavigation(Vector3 targetPos, System.Action onArrived)
{
    if (autoNavigator == null)
    {
        Debug.LogWarning("[GameInputManager] PlayerAutoNavigator 未初始化，无法导航");
        return;
    }
    
    // 取消当前导航
    autoNavigator.ForceCancel();
    
    // 计算停止距离（略小于工具使用距离）
    float stopDistance = farmToolReach * 0.8f;
    
    // 创建临时目标点（用于导航）
    // 注意：不能使用 FollowTarget，因为目标是地面格子，不是 Transform
    autoNavigator.SetDestination(targetPos);
    
    // 🔥 使用协程监控导航完成
    StartCoroutine(WaitForNavigationComplete(targetPos, stopDistance, onArrived));
}

/// <summary>
/// 等待导航完成的协程
/// </summary>
private System.Collections.IEnumerator WaitForNavigationComplete(Vector3 targetPos, float stopDistance, System.Action onArrived)
{
    // 等待导航开始
    yield return null;
    
    // 监控导航状态
    while (autoNavigator != null && autoNavigator.IsActive)
    {
        // 检查是否已到达
        Vector2 playerPos = GetPlayerCenter();
        float distance = Vector2.Distance(playerPos, targetPos);
        
        if (distance <= stopDistance)
        {
            // 已到达，停止导航并执行回调
            autoNavigator.ForceCancel();
            onArrived?.Invoke();
            yield break;
        }
        
        yield return null;
    }
    
    // 导航结束（可能被取消或卡住）
    // 检查是否在范围内
    Vector2 finalPos = GetPlayerCenter();
    float finalDistance = Vector2.Distance(finalPos, targetPos);
    
    if (finalDistance <= stopDistance * 1.2f) // 允许 20% 容差
    {
        onArrived?.Invoke();
    }
}
```

### 3. 边界情况处理

#### 3.1 导航过程中切换工具

在 `HandleHotbarSelection` 中，切换工具时取消农田导航：

```csharp
void HandleHotbarSelection()
{
    // ... 现有逻辑 ...
    
    // 🔥 切换工具时取消农田导航
    if (keyIndex >= 0 || scroll != 0f)
    {
        CancelFarmingNavigation();
    }
}

private Coroutine _farmingNavigationCoroutine;

private void CancelFarmingNavigation()
{
    if (_farmingNavigationCoroutine != null)
    {
        StopCoroutine(_farmingNavigationCoroutine);
        _farmingNavigationCoroutine = null;
    }
}
```

#### 3.2 导航过程中打开面板

在 `HandlePanelHotkeys` 中，打开面板时取消农田导航：

```csharp
void HandlePanelHotkeys()
{
    // ... 现有逻辑 ...
    
    // 🔥 打开任何面板时取消农田导航
    if (Input.GetKeyDown(KeyCode.Tab) || Input.GetKeyDown(KeyCode.B) || ...)
    {
        CancelFarmingNavigation();
    }
}
```

#### 3.3 导航过程中玩家手动移动

在 `HandleMovement` 中，玩家输入时取消农田导航：

```csharp
void HandleMovement()
{
    // ... 现有逻辑 ...
    
    // 🔥 玩家手动移动时取消农田导航
    if (Mathf.Abs(input.x) > 0.01f || Mathf.Abs(input.y) > 0.01f)
    {
        CancelFarmingNavigation();
    }
}
```

## 状态流转图

### 农田导航状态机（锐评002补充）

**状态定义**：

```csharp
private enum FarmNavState 
{ 
    Idle,       // 空闲，无导航任务
    Navigating, // 正在导航
    Executing   // 正在执行动作
}

private FarmNavState _farmNavState = FarmNavState.Idle;
private System.Action _farmNavigationAction = null;
private SeedData _cachedSeedData = null;
```

**状态转换图**：

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────┐    点击远距离有效目标    ┌────────────┐           │
│  │ Idle │ ──────────────────────> │ Navigating │           │
│  └──────┘                         └────────────┘           │
│      ▲                                  │                   │
│      │                                  │ 到达              │
│      │                                  ▼                   │
│      │    动作完成/失败          ┌────────────┐           │
│      └────────────────────────── │ Executing  │           │
│                                  └────────────┘           │
│                                                             │
│  中断条件（任意状态 → Idle）：                              │
│  - 手动移动（WASD）                                         │
│  - 切换快捷栏（数字键/滚轮）                                │
│  - 打开面板（Tab/ESC/B/M/L/O）                              │
│  - 受到攻击                                                 │
└─────────────────────────────────────────────────────────────┘
```

### 导航回调时序（锐评002补充）

**关键约束**：T1 到 T3 必须在同一帧内完成，避免滑步。

| 时序点 | 说明 | 帧要求 |
|--------|------|--------|
| T0 | 玩家点击，启动导航 | Frame N |
| T1 | 导航到达，触发 onComplete | Frame N+X |
| T2 | onComplete 内执行二次检查 | Frame N+X（同帧） |
| T3 | 二次检查通过，执行动作 | Frame N+X（同帧） |

**验证代码**：
```csharp
// 在 onComplete 回调中
Debug.Log($"[FarmNav] onComplete at frame {Time.frameCount}");
// 在 ExecuteAction 中
Debug.Log($"[FarmNav] ExecuteAction at frame {Time.frameCount}");
// 两者必须相同
```

### 用户点击流程图

### 用户点击流程图

```
┌─────────────────────────────────────────────────────────────┐
│                      用户点击左键                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ FarmToolPreview │
                    │    IsValid?     │
                    └─────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
        ┌─────────┐                     ┌─────────┐
        │  false  │                     │  true   │
        └─────────┘                     └─────────┘
              │                               │
              ▼                               ▼
        ┌─────────┐                   ┌─────────────┐
        │ 无动作  │                   │  IsInRange? │
        └─────────┘                   └─────────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              │                               │
                              ▼                               ▼
                        ┌─────────┐                     ┌─────────┐
                        │  true   │                     │  false  │
                        └─────────┘                     └─────────┘
                              │                               │
                              ▼                               ▼
                    ┌─────────────────┐           ┌─────────────────┐
                    │   立即执行动作   │           │   启动导航      │
                    └─────────────────┘           └─────────────────┘
                                                          │
                                                          ▼
                                                ┌─────────────────┐
                                                │   等待到达      │
                                                └─────────────────┘
                                                          │
                                                          ▼
                                                ┌─────────────────┐
                                                │   二次检查      │
                                                │ IsValid && InRange │
                                                └─────────────────┘
                                                          │
                                          ┌───────────────┴───────────────┐
                                          │                               │
                                          ▼                               ▼
                                    ┌─────────┐                     ┌─────────┐
                                    │  true   │                     │  false  │
                                    └─────────┘                     └─────────┘
                                          │                               │
                                          ▼                               ▼
                                ┌─────────────────┐               ┌─────────┐
                                │   执行动作      │               │ 无动作  │
                                └─────────────────┘               └─────────┘
```

## 正确性属性

### 核心属性

### P1：IsValid 与距离解耦
```
∀ preview ∈ FarmToolPreview:
  preview.IsValid ⟺ (逻辑合法 ∧ 物理合法)
  preview.IsValid ⊥ preview.IsInRange  // 独立
```

### P2：近距离立即执行
```
∀ click ∈ LeftClick:
  (preview.IsValid ∧ preview.IsInRange) ⟹ ExecuteImmediately()
```

### P3：远距离触发导航
```
∀ click ∈ LeftClick:
  (preview.IsValid ∧ ¬preview.IsInRange) ⟹ StartNavigation()
```

### P4：无效目标无动作
```
∀ click ∈ LeftClick:
  ¬preview.IsValid ⟹ NoAction()
```

### P5：二次检查保护
```
∀ arrival ∈ NavigationComplete:
  ExecuteAction() ⟹ (preview.IsValid ∧ preview.IsInRange)
```

### 边界属性（锐评002补充）

### P6：状态机完整性
```
∀ state ∈ FarmNavState:
  state ∈ {Idle, Navigating, Executing}
  
∀ interrupt ∈ {WASD, Hotbar, Panel, Attack}:
  interrupt ⟹ state := Idle
```

### P7：回调时序一致性
```
∀ navigation ∈ FarmingNavigation:
  onComplete.frame == ExecuteAction.frame  // 同帧执行
```

### P8：种子缓存一致性
```
∀ seedNavigation ∈ SeedNavigation:
  ExecutePlant() ⟹ (cachedSeed == currentHeldSeed)
```

### P9：中断清理完整性
```
∀ cancel ∈ CancelFarmingNavigation:
  cancel ⟹ (_farmNavState == Idle ∧ _farmNavigationAction == null ∧ _cachedSeedData == null)
```

### P10：快照校验一致性（锐评006补充）
```
∀ seedNavigation ∈ SeedNavigation:
  StartNavigation() ⟹ CreateSnapshot(itemId, slotIndex, count)
  ExecutePlant() ⟹ ValidateSnapshot() == true
```

### P11：快照失效处理（锐评006补充）
```
∀ arrival ∈ NavigationComplete:
  ¬ValidateSnapshot() ⟹ (NoAction() ∧ PlayCancelSound())
```

## 测试策略

### 单元测试
- FarmToolPreview.IsValid 与 IsInRange 独立性
- 距离计算正确性

### 集成测试
- 近距离锄地/浇水/种植
- 远距离导航→锄地/浇水/种植
- 导航中断场景（切换工具、打开面板、手动移动）
- 二次检查失败场景

### 手动测试
- 大范围农田操作流畅度
- 边界情况用户体验
