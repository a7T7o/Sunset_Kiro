# 9.0.5 智能交互Bug修复 - 设计文档

**创建日期**: 2026-02-09
**工作区**: `.kiro/specs/农田系统/9.0.5智能交互bug修复/`

---

## 一、架构概述

### 1.1 当前架构（9.0.4）

```
GameInputManager.Update()
    ↓
UpdatePreviews()  ← 每帧无条件更新预览位置
    ↓
HandleUseCurrentTool()
    ↓
TryTillSoil / TryWaterTile / TryPlantSeed
    ↓
IsValid? → IsInRange?
    ├─ 近距离 → RequestAction + Execute
    └─ 远距离 → StartFarmingNavigation（预览仍跟随鼠标！）
```

**问题**：`UpdatePreviews()` 不检查导航状态，导航中预览仍跟随鼠标。

### 1.2 目标架构（9.0.5）

```
GameInputManager.Update()
    ↓
UpdatePreviews()  ← 检查 FarmNavState，锁定时不更新位置
    ↓
HandleUseCurrentTool()
    ↓
TryTillSoil / TryWaterTile / TryPlantSeed
    ↓
IsValid? → 锁定预览（LockPosition）→ IsInRange?
    ├─ 近距离 → Execute → 解锁预览（UnlockPosition）
    └─ 远距离 → StartFarmingNavigation（预览保持锁定）
                    ↓
              导航完成/取消 → 解锁预览（UnlockPosition）
```

---

## 二、状态机设计

### 2.1 FarmNavState 扩展

```csharp
private enum FarmNavState
{
    Idle,       // 空闲，无预览（手持非农具）
    Preview,    // 预览跟随鼠标（手持农具/种子）
    Locked,     // 预览锁定在目标位置（点击后，判断距离前）
    Navigating, // 导航中，预览保持锁定
    Executing   // 执行中，预览保持锁定
}
```

### 2.2 状态转换图

```
Idle ──(手持农具)──→ Preview
  ↑                    ↓
  │              (点击有效位置)
  │                    ↓
  │                 Locked
  │                    ↓
  │            ┌─(近距离)─→ Executing ──(完成)──→ Preview
  │            │                                     ↑
  │            └─(远距离)─→ Navigating               │
  │                           ↓                      │
  │                    ┌─(到达)─→ Executing ──(完成)──┘
  │                    │
  │                    ├─(WASD/ESC/切换物品)──→ Preview
  │                    │
  │                    └─(点击新位置)──→ Locked（中断+重启）
  │                                       │
  └──────(切换到非农具)───────────────────┘

注意：打开面板不改变状态，面板冻结所有事件，关闭面板后恢复
```

### 2.3 状态转换规则

| 当前状态 | 触发条件 | 目标状态 | 动作 |
|---------|---------|---------|------|
| Idle | 手持农具/种子 | Preview | 显示预览 |
| Preview | 点击有效位置 | Locked | LockPosition + 创建快照 |
| Preview | 切换到非农具 | Idle | 隐藏预览 |
| Locked | 近距离 | Executing | RequestAction + Execute |
| Locked | 远距离 | Navigating | StartFarmingNavigation |
| Navigating | 到达 | Executing | 二次校验 + Execute |
| Navigating | WASD/ESC/切换物品 | Preview | CancelNavigation + UnlockPosition（回到跟随鼠标） |
| Navigating | 点击新有效位置 | Locked | CancelNavigation + 重新锁定新位置 + 重新开始（完整中断+重启） |
| Executing | 完成 | Preview | UnlockPosition |
| 任何状态 | 打开面板 | （不变） | 面板冻结所有事件，导航状态保留，预览保持当前状态 |
| 任何状态 | 关闭面板 | （不变） | 恢复之前的状态，继续之前的导航/预览 |

---

## 三、FarmToolPreview 修改设计

### 3.1 新增字段

```csharp
// === 锁定机制 ===
private bool _isLocked = false;

// === 锁定时的视觉数据（冻结，不跟随鼠标） ===
private Vector3 _lockedWorldPosition;
private Vector3Int _lockedCellPos;
private int _lockedLayerIndex;

// === 实时数据（永远跟随鼠标更新，即使锁定状态） ===
// 已有的 CurrentCursorPos / CurrentCellPos / CurrentLayerIndex / IsInRange / IsValid
// 这些属性在锁定状态下仍然实时更新，供 GameInputManager 判断新点击
```

### 3.2 新增方法

```csharp
/// <summary>
/// 锁定预览位置（点击后调用）
/// 冻结视觉显示，但实时数据继续更新
/// </summary>
public void LockPosition(Vector3 worldPos, Vector3Int cellPos, int layerIndex)
{
    _isLocked = true;
    _lockedWorldPosition = worldPos;
    _lockedCellPos = cellPos;
    _lockedLayerIndex = layerIndex;
}

/// <summary>
/// 解锁预览位置（执行完成/取消后调用）
/// 恢复视觉跟随鼠标
/// </summary>
public void UnlockPosition()
{
    _isLocked = false;
}

/// <summary>
/// 是否处于锁定状态
/// </summary>
public bool IsLocked => _isLocked;
```

### 3.3 修改 UpdateXxxPreview 方法 — 视觉与数据分离

**核心修正（锐评002"盲人导航"修复）**：锁定状态下，实时数据（CurrentCellPos/IsValid/IsInRange）仍然跟随鼠标更新，只有视觉渲染（GhostTilemap/CursorRenderer）保持冻结。

```csharp
public void UpdateHoePreview(int layerIndex, Vector3Int cellPos, Transform playerTransform, float reach)
{
    // ★ 第一步：永远更新实时数据（不管是否锁定）
    UpdateRealtimeData(layerIndex, cellPos, playerTransform, reach);
    
    // ★ 第二步：锁定状态下不更新视觉
    if (_isLocked) return;
    
    // ... 原有的 GhostTilemap 渲染、CursorRenderer 位置/颜色更新逻辑
}
```

### 3.4 新增 UpdateRealtimeData 轻量方法

```csharp
/// <summary>
/// 轻量级实时数据更新（不触碰视觉渲染）
/// 锁定状态下也会被调用，确保 GameInputManager 能读到最新的鼠标位置数据
/// </summary>
private void UpdateRealtimeData(int layerIndex, Vector3Int cellPos, Transform playerTransform, float reach)
{
    Vector3 cellCenter = GetCellCenterWorld(layerIndex, cellPos);
    
    // 更新实时位置
    CurrentCursorPos = cellCenter;
    CurrentCellPos = cellPos;
    CurrentLayerIndex = layerIndex;
    
    // 更新距离状态
    IsInRange = playerTransform != null ? IsWithinReach(playerTransform, cellCenter, reach) : true;
    
    // 更新有效性（根据工具类型不同，由各 UpdateXxxPreview 在调用前设置 isHoeMode/isSeedMode）
    // 具体有效性判断逻辑保留在各 UpdateXxxPreview 方法中
}
```

---

## 四、GameInputManager 修改设计

### 4.1 UpdatePreviews 修改

**核心修正（锐评002"盲人导航"修复）**：`UpdatePreviews` 在锁定/导航/执行状态下**不能直接 return**，必须仍然调用 `UpdateFarmToolPreview` 以更新实时数据。视觉冻结由 `FarmToolPreview` 内部的 `_isLocked` 控制。

```csharp
private void UpdatePreviews()
{
    // 面板打开时：不隐藏预览，不更新预览
    // 面板本身就是事件屏蔽器，导航状态保留，预览保持锁定
    if (IsAnyPanelOpen())
    {
        // 🔥 9.0.5 修正：面板打开时不做任何预览操作
        // 不隐藏（保留锁定状态），不更新（面板冻结事件）
        return;
    }
    
    // 鼠标在 UI 上时隐藏
    if (EventSystem.current != null && EventSystem.current.IsPointerOverGameObject())
    { HideAllPreviews(); return; }
    
    // 🔥 9.0.5 修正：不再在此处检查 FarmNavState
    // 即使在 Locked/Navigating/Executing 状态，也要调用 UpdateFarmToolPreview
    // 让 FarmToolPreview 内部的 _isLocked 控制视觉冻结
    // 实时数据（CurrentCellPos/IsValid/IsInRange）始终更新
    
    // ... 原有路由逻辑（根据手持物品类型路由到对应预览）
}
```

### 4.2 TryTillSoil / TryWaterTile / TryPlantSeed 修改

在 IsValid 检查通过后，先锁定预览，再判断距离：

```csharp
private bool TryTillSoil(Vector3 worldPosition)
{
    var farmPreview = FarmToolPreview.Instance;
    if (farmPreview == null || !farmPreview.IsValid()) return false;
    
    // 🔥 9.0.5：锁定预览
    Vector3 targetPos = farmPreview.CurrentCursorPos;
    int layerIndex = farmPreview.CurrentLayerIndex;
    Vector3Int cellPos = farmPreview.CurrentCellPos;
    
    farmPreview.LockPosition(targetPos, cellPos, layerIndex);
    _farmNavState = FarmNavState.Locked;
    
    // 创建快照
    // ...
    
    if (farmPreview.IsInRange)
    {
        // 近距离执行
        _farmNavState = FarmNavState.Executing;
        playerInteraction?.RequestAction(Crush);
        ExecuteTillSoil(layerIndex, cellPos);
        
        // 执行完成，解锁
        farmPreview.UnlockPosition();
        _farmNavState = FarmNavState.Preview;
        return true;
    }
    else
    {
        // 远距离导航
        StartFarmingNavigation(targetPos, () => {
            // 到达后校验 + 执行 + 解锁
        });
        return true;
    }
}
```

### 4.3 CancelFarmingNavigation 修改

```csharp
private void CancelFarmingNavigation()
{
    if (_farmNavState == FarmNavState.Idle || _farmNavState == FarmNavState.Preview) return;
    
    // 停止协程
    if (_farmingNavigationCoroutine != null)
    {
        StopCoroutine(_farmingNavigationCoroutine);
        _farmingNavigationCoroutine = null;
    }
    
    // 🔥 9.0.5：解锁预览（原子性保证）
    var farmPreview = FarmToolPreview.Instance;
    if (farmPreview != null)
    {
        farmPreview.UnlockPosition();
    }
    
    // 重置状态 → 回到 Preview（如果仍持有农具）
    _farmNavState = FarmNavState.Preview; // 而非 Idle
    _farmNavigationAction = null;
    _cachedSeedData = null;
    ClearSnapshot();
}
```

### 4.3.1 面板热键不取消导航

**核心修正**：面板打开/关闭不影响农田导航状态。面板本身就是事件屏蔽器，打开面板时所有输入事件被冻结，导航状态保留。

```csharp
// HandlePanelHotkeys 中：
// ❌ 旧设计：CancelFarmingNavigation(); 在每个面板热键前调用
// ✅ 新设计：移除面板热键中的 CancelFarmingNavigation() 调用
// 面板打开后 IsAnyPanelOpen() 返回 true，Update 中的输入处理自然被屏蔽
// 关闭面板后恢复正常，导航继续
```

**注意**：ESC 键是特殊情况 — ESC 在没有面板打开时应该中断导航：
```csharp
if (Input.GetKeyDown(KeyCode.Escape))
{
    // 优先关闭面板
    if (IsAnyPanelOpen())
    {
        // 关闭面板逻辑（不取消导航）
        return;
    }
    
    // 没有面板打开时，ESC 中断导航
    if (_farmNavState == FarmNavState.Navigating || _farmNavState == FarmNavState.Locked)
    {
        CancelFarmingNavigation();
        return;
    }
}
```

### 4.4 执行保护标志

```csharp
private bool _isExecutingFarming = false;

// 在 Execute 方法中使用
private bool ExecuteTillSoil(int layerIndex, Vector3Int cellPos)
{
    if (_isExecutingFarming) return false;
    _isExecutingFarming = true;
    
    try
    {
        // ... 执行逻辑
    }
    finally
    {
        _isExecutingFarming = false;
    }
}
```

### 4.5 浇水失败原因增强

在 `FarmToolPreview` 中添加失败原因记录：

```csharp
public enum WateringFailureReason
{
    None,
    NoFarmland,      // 没有耕地数据
    NotTilled,       // 未耕作
    AlreadyWatered,  // 已浇水
    HasObstacle,     // 有障碍物
    ManagerNull      // FarmTileManager 为 null
}

public WateringFailureReason LastWateringFailure { get; private set; }
```

---

## 五、正确性属性

### P1：预览锁定不变性
当 `FarmNavState ∈ {Locked, Navigating, Executing}` 时，`FarmToolPreview.IsLocked == true` 且预览位置不变。

### P2：状态机完整性
`FarmNavState` 的每次转换都必须在状态转换表中有定义，不允许非法转换。

### P3：中断恢复一致性
任何中断操作（WASD/ESC/切换物品/打开面板）后，`FarmToolPreview.IsLocked == false` 且 `FarmNavState ∈ {Idle, Preview}`。

### P4：快照有效性
当 `FarmNavState == Navigating` 时，`_farmingSnapshot.isValid == true`。

### P5：执行保护互斥
当 `_isExecutingFarming == true` 时，不响应新的点击事件和切换物品事件。

### P6：预览-状态同步
`FarmToolPreview.IsLocked` 与 `FarmNavState` 始终同步：
- IsLocked == true ↔ FarmNavState ∈ {Locked, Navigating, Executing}
- IsLocked == false ↔ FarmNavState ∈ {Idle, Preview}

---

## 六、测试框架

使用 Unity Test Framework (NUnit) 进行 EditMode 测试。

---

## 七、与现有系统的兼容性

| 系统 | 影响 | 处理方式 |
|------|------|---------|
| PlacementManager | 无影响 | 独立系统，不共享状态 |
| PlayerAutoNavigator | 无修改 | 只调用现有 API |
| PlayerInteraction | 无修改 | 只调用 RequestAction |
| HotbarSelectionService | 无修改 | 只读取 selectedIndex |
| InventoryService | 无修改 | 只调用 GetSlot/RemoveItem |

---

## 八、风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| 实时数据更新增加每帧开销 | 低 | 性能微降 | UpdateRealtimeData 是轻量计算，无 GC |
| CancelFarmingNavigation 调用时机不对 | 低 | 状态不一致 | 添加状态检查 + UnlockPosition 原子性保证 |
| 执行保护标志未正确重置 | 低 | 无法再次操作 | 使用 try/finally |
| 面板关闭后导航状态恢复异常 | 中 | 导航卡住 | 面板不修改导航状态，自然恢复 |

## 九、中断行为分类（用户修正）

| 操作 | 行为 | 说明 |
|------|------|------|
| WASD 移动 | 中断导航，解锁预览，回到跟随鼠标 | 玩家主动移动 = 放弃当前操作 |
| 切换物品（滚轮/数字键） | 中断导航，解锁预览，回到跟随鼠标 | 切换工具 = 放弃当前操作 |
| ESC（无面板打开时） | 中断导航，解锁预览，回到跟随鼠标 | ESC = 取消 |
| 左键点击新位置 | 中断当前导航，重新锁定+重新开始 | 完整中断+重启，非"修改目的地" |
| 打开面板（Tab/B/M/L/O） | **不中断**，面板冻结事件，状态保留 | 面板是事件屏蔽器 |
| 关闭面板 | 恢复之前状态，继续导航 | 自然恢复 |
