# 10.1.0 全面持久化前夕 — 设计文档

> 来源：requirements.md + 代码事实验证
> 创建时间：2026-02-17
> 工作区：`.kiro/specs/农田系统/10.1.0全面持久化前夕/`
> 测试框架：NUnit（Unity Test Runner EditMode）

---

## 一、US-1 动作生命周期与输入缓存

### 1.1 问题根因（代码事实）

当前 `HandleUseCurrentTool()` 中有两道阻断：
1. `_isExecutingFarming` — 第 614 行，农田工具执行中直接 return
2. `IsPerformingAction()` — 第 654 行，动画播放中直接 return

两道阻断导致动画期间的所有输入被丢弃。`OnActionComplete()` 的长按继续逻辑只重播动画（`StartAction(actionToRepeat, true)`），不通知 `GameInputManager` 执行农田操作。

### 1.2 设计方案

#### 1.2.1 新增输入缓存结构

在 `GameInputManager` 中新增：

```csharp
// 输入缓存（动画期间暂存）
private bool _hasPendingFarmInput;
private Vector3 _pendingFarmWorldPos;  // 缓存鼠标世界坐标
private int _pendingFarmItemId;        // 缓存手持物品 ID
```

#### 1.2.2 修改 `HandleUseCurrentTool()` 的阻断逻辑

当前逻辑（第 654 行）：
```csharp
bool isPerformingAction = playerInteraction != null && playerInteraction.IsPerformingAction();
if (isPerformingAction) return;
```

修改为：
```csharp
bool isPerformingAction = playerInteraction != null && playerInteraction.IsPerformingAction();
if (isPerformingAction)
{
    // 农田工具/种子：缓存输入而非丢弃
    if (itemData is ToolData t && (t.toolType == ToolType.Hoe || t.toolType == ToolType.WateringCan))
    {
        CacheFarmInput(itemData.itemID);
        return;
    }
    if (itemData is SeedData)
    {
        CacheFarmInput(itemData.itemID);
        return;
    }
    // 非农田工具：保持原有阻断
    return;
}
```

#### 1.2.3 新增 `CacheFarmInput()` 方法

```csharp
private void CacheFarmInput(int itemId)
{
    _hasPendingFarmInput = true;
    _pendingFarmWorldPos = GetMouseWorldPosition();
    _pendingFarmItemId = itemId;
}
```

#### 1.2.4 新增 `ConsumePendingFarmInput()` 方法

动画结束后由 `PlayerInteraction.OnActionComplete()` 通过回调通知 `GameInputManager`：

```csharp
public void ConsumePendingFarmInput()
{
    if (!_hasPendingFarmInput) return;
    _hasPendingFarmInput = false;
    
    // 验证手持物品一致性
    int currentItemId = GetCurrentHeldItemId();
    if (currentItemId != _pendingFarmItemId) return;
    
    // 以缓存的世界坐标重新执行完整动作链
    // 这里不直接调用 TryHandleFarmingTool/TryPlantSeed
    // 而是模拟一次完整的输入处理（含预览刷新、有效性判断）
    ProcessFarmInputAt(_pendingFarmWorldPos);
}
```

#### 1.2.5 `PlayerInteraction` 回调机制

在 `PlayerInteraction.OnActionComplete()` 中，长按继续分支和松开分支都需要通知 `GameInputManager`：

```csharp
// OnActionComplete() 中新增：
// 通知 GameInputManager 消费缓存
var gim = GameInputManager.Instance; // 需要新增 Instance 引用
if (gim != null) gim.ConsumePendingFarmInput();
```

注意：这个回调在长按继续时也要触发，因为长按 = 当前鼠标位置作为缓存输入。

#### 1.2.6 长按左键处理

当前 `HandleUseCurrentTool()` 只响应 `GetMouseButtonDown(0)`（第 618 行）。长按处理不在这里改，而是在 `OnActionComplete()` 的回调中：

- 长按继续时，`OnActionComplete()` 检测到 `GetMouseButton(0)` 为 true
- 通知 `GameInputManager.ConsumePendingFarmInput()`
- 如果没有缓存输入，则以当前鼠标位置作为输入（等价于 AC-1.2）

#### 1.2.7 导航中重新输入（AC-1.3）

当前逻辑（第 622-639 行）已处理导航中重新点击：
- 新位置有效：`CancelFarmingNavigation()` → 继续往下走重新进入 `TryHandleFarmingTool`
- 新位置无效：`CancelFarmingNavigation()` → return

问题：`CancelFarmingNavigation()` 调用 `UnlockPosition()`，但紧接着 `TryTillSoil`/`TryWaterTile` 又会 `LockPosition()`。在这个解锁→重锁的间隙，预览可能闪烁。

修复方案：在 `LockPosition()` 内部自动刷新一次视觉（AC-1.4）。

#### 1.2.8 `LockPosition` 视觉同步（AC-1.4）

当前 `LockPosition()` 只设置数据（`_isLocked`, `_lockedWorldPosition` 等），不刷新视觉。

修改 `FarmToolPreview.LockPosition()`：
```csharp
public void LockPosition(Vector3 worldPos, Vector3Int cellPos, int layerIndex)
{
    _isLocked = true;
    _lockedWorldPosition = worldPos;
    _lockedCellPos = cellPos;
    _lockedLayerIndex = layerIndex;
    
    // 🔥 AC-1.4：锁定时自动刷新视觉到锁定位置
    UpdateCursor(layerIndex, cellPos);
}
```

#### 1.2.9 `_isExecutingFarming` 保护调整（AC-1.5）

当前 `_isExecutingFarming` 在 `HandleUseCurrentTool()` 第 614 行一刀切阻断。

修改：`_isExecutingFarming` 为 true 时，农田工具输入走缓存路径而非直接 return。

```csharp
// 原来：
if (_isExecutingFarming) return;

// 改为：
if (_isExecutingFarming)
{
    // 执行中的新输入走缓存
    if (Input.GetMouseButtonDown(0) && !EventSystem.current.IsPointerOverGameObject())
    {
        var item = database.GetItemByID(inventory.GetSlot(hotbarSelection.selectedIndex).itemId);
        if (item is ToolData t && (t.toolType == ToolType.Hoe || t.toolType == ToolType.WateringCan))
            CacheFarmInput(item.itemID);
        else if (item is SeedData)
            CacheFarmInput(item.itemID);
    }
    return;
}
```

### 1.3 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `GameInputManager.cs` | 新增缓存字段、`CacheFarmInput()`、`ConsumePendingFarmInput()`、`ProcessFarmInputAt()`，修改 `HandleUseCurrentTool()` 阻断逻辑 |
| `PlayerInteraction.cs` | `OnActionComplete()` 新增回调通知 `GameInputManager` |
| `FarmToolPreview.cs` | `LockPosition()` 新增视觉刷新 |

### 1.4 不修改的部分

- `PlayerInteraction.IsPerformingAction()` 返回值逻辑不变
- `PlayerInteraction.RequestAction()` 不变
- 非农田工具（武器/其他）的 `IsPerformingAction()` 保护不变
- `FarmNavState` 枚举不变
- `StartFarmingNavigation()` / `WaitForNavigationComplete()` 不变

---

## 二、US-2 存档字段补全

### 2.1 问题根因（代码事实）

`FarmTileSaveData` 当前活跃字段：`tileX`, `tileY`, `layer`, `soilState`, `isWatered`

`FarmTileData` 运行时字段中未被存档覆盖的：
- `wateredYesterday`（bool）— `ResetDailyWaterState()` 中 `wateredYesterday = wateredToday`
- `waterTime`（float）— `SetWatered()` 中设置，`OnHourChanged()` 中用于计算水渍消退
- `puddleVariant`（int）— `SetWatered()` 中随机生成，`UpdateTileVisual()` 中用于选择水渍 Tile

`Save()` 只写入 5 个字段，`CreateTileFromSaveData()` 只恢复 `moistureState` 和 `wateredToday`。

### 2.2 设计方案

#### 2.2.1 `FarmTileSaveData` 新增字段

```csharp
/// <summary>昨天是否浇过水</summary>
public bool wateredYesterday;

/// <summary>浇水时间（游戏小时）</summary>
public float waterTime;

/// <summary>水渍变体索引（0-2）</summary>
public int puddleVariant;
```

JSON 序列化兼容性：`JsonUtility.FromJson` 对缺失字段自动使用类型默认值（bool=false, float=0, int=0），无需额外兼容代码。

#### 2.2.2 `FarmTileManager.Save()` 适配

在 `farmTiles.Add(new FarmTileSaveData { ... })` 中新增 3 个字段写入：

```csharp
farmTiles.Add(new FarmTileSaveData
{
    tileX = tile.position.x,
    tileY = tile.position.y,
    layer = tile.layerIndex,
    soilState = (int)tile.moistureState,
    isWatered = tile.wateredToday,
    // 🔥 新增
    wateredYesterday = tile.wateredYesterday,
    waterTime = tile.waterTime,
    puddleVariant = tile.puddleVariant
});
```

#### 2.2.3 `FarmTileManager.CreateTileFromSaveData()` 适配

在创建 `FarmTileData` 后新增 3 个字段恢复：

```csharp
FarmTileData newTile = new FarmTileData(cellPosition, layerIndex);
newTile.isTilled = true;
newTile.moistureState = (SoilMoistureState)saveData.soilState;
newTile.wateredToday = saveData.isWatered;
// 🔥 新增
newTile.wateredYesterday = saveData.wateredYesterday;
newTile.waterTime = saveData.waterTime;
newTile.puddleVariant = saveData.puddleVariant;
```

#### 2.2.4 旧存档兼容

`JsonUtility.FromJson<FarmTileSaveData>` 对旧存档中不存在的字段自动使用默认值：
- `wateredYesterday` → `false`（bool 默认值）
- `waterTime` → `0f`（float 默认值）
- `puddleVariant` → `0`（int 默认值）

这些默认值是安全的：
- `wateredYesterday = false` → 日结时不会错误触发作物生长
- `waterTime = 0f` → `OnHourChanged()` 中 `hoursSinceWatering` 会很大，水渍直接消退为 WetDark（合理行为）
- `puddleVariant = 0` → 使用第一种水渍变体（视觉可接受）

### 2.3 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `SaveDataDTOs.cs` | `FarmTileSaveData` 新增 3 个字段 |
| `FarmTileManager.cs` | `Save()` 写入新字段，`CreateTileFromSaveData()` 恢复新字段 |

---

## 三、US-3 干燥/湿润耕地视觉渲染

### 3.1 问题根因（代码事实）

- `FarmVisualManager` 有 `dryFarmlandTile` 和 `wetDarkTile` 两个 `[SerializeField]` 字段（第 24-28 行），但 `UpdateTileVisual()` 从未使用
- `UpdateTileVisual()` 只操作 `puddleTilemap`（水渍叠加层），不修改 `farmTilemap`（耕地层）
- `CreateTile()` 创建耕地后没有设置耕地 Tile 视觉
- `SoilMoistureState` 枚举已有 `Dry`/`WetWithPuddle`/`WetDark` 三状态

### 3.2 设计方案

#### 3.2.1 `UpdateTileVisual()` 扩展

在现有水渍叠加层逻辑之外，新增耕地 Tile 渲染逻辑：

```csharp
public void UpdateTileVisual(LayerTilemaps tilemaps, Vector3Int cellPosition, FarmTileData tileData)
{
    // === 现有逻辑：水渍叠加层（不修改）===
    // ... puddleTilemap 操作 ...
    
    // === 新增逻辑：耕地 Tile 渲染 ===
    Tilemap farmTilemap = tilemaps.farmlandCenterTilemap;
    if (farmTilemap == null) return;
    
    switch (tileData.moistureState)
    {
        case SoilMoistureState.Dry:
        case SoilMoistureState.WetWithPuddle:
            // 干燥和有水渍时，耕地显示干燥 Tile
            if (dryFarmlandTile != null)
                farmTilemap.SetTile(cellPosition, dryFarmlandTile);
            break;
            
        case SoilMoistureState.WetDark:
            // 湿润时，耕地显示湿润 Tile
            if (wetDarkTile != null)
                farmTilemap.SetTile(cellPosition, wetDarkTile);
            break;
    }
}
```

#### 3.2.2 锄地时设置干燥耕地 Tile

`FarmTileManager.CreateTile()` 成功后，需要调用 `UpdateTileVisual()` 设置 `dryFarmlandTile`：

```csharp
// CreateTile() 末尾新增：
var tilemaps = GetLayerTilemaps(layerIndex);
if (tilemaps != null && visualManager != null)
{
    visualManager.UpdateTileVisual(tilemaps, cellPosition, newTile);
}
```

#### 3.2.3 水渍消退后的渐变效果（AC-3.3）

当前 `OnHourChanged()` 中水渍消退时直接设置 `moistureState = WetDark`。

渐变方案选择：分阶段 Tile 替换（最稳妥，不需要 Shader）

实现思路：
- 水渍消退后不立即切换为 `wetDarkTile`
- 新增一个渐变协程，在 30 游戏分钟内分 3-5 步从 `dryFarmlandTile` 过渡到 `wetDarkTile`
- 如果有中间过渡 Tile（如半湿润），可以在 Inspector 配置
- 如果没有中间 Tile，使用 `farmTilemap` 的 `color` 属性做颜色插值（从干燥色渐变到湿润色）

推荐方案：使用 `Tilemap.SetColor()` 对单个格子做颜色插值
- 优点：不需要额外 Tile 资源，不需要 Shader
- 缺点：`Tilemap.SetColor()` 是 per-tile 的，性能可接受（只对浇水过的格子操作）

```csharp
// FarmVisualManager 新增：
private IEnumerator GradualMoistureTransition(
    LayerTilemaps tilemaps, Vector3Int cellPos, float durationGameMinutes)
{
    Tilemap farmTilemap = tilemaps.farmlandCenterTilemap;
    if (farmTilemap == null) yield break;
    
    // 先设置为 dryFarmlandTile（如果还不是）
    farmTilemap.SetTile(cellPos, dryFarmlandTile);
    
    // 渐变：通过 SetColor 从白色（原色）渐变到湿润色调
    Color dryColor = Color.white;
    Color wetColor = new Color(0.7f, 0.7f, 0.8f, 1f); // 偏蓝灰的湿润色
    
    float elapsed = 0f;
    float duration = durationGameMinutes; // 游戏分钟
    
    while (elapsed < duration)
    {
        float t = elapsed / duration;
        farmTilemap.SetColor(cellPos, Color.Lerp(dryColor, wetColor, t));
        
        // 等待一段真实时间（对应游戏时间推进）
        yield return new WaitForSeconds(2f); // 每2秒更新一次
        
        // 重新计算已过游戏分钟
        var tm = TimeManager.Instance;
        if (tm == null) break;
        elapsed = /* 根据游戏时间计算 */;
    }
    
    // 渐变完成：替换为 wetDarkTile
    farmTilemap.SetTile(cellPos, wetDarkTile);
    farmTilemap.SetColor(cellPos, Color.white); // 重置颜色
}
```

注意：渐变进度不需要持久化（AC-3.5），读档后直接显示目标状态。

#### 3.2.4 日结回退逻辑（AC-3.4）

当前 `ResetDailyWaterState()` 已将所有耕地重置为 `Dry`。`OnDayChanged()` 之后调用 `RefreshAllTileVisuals()`。

修改后 `RefreshAllTileVisuals()` 会自动根据 `moistureState` 设置正确的耕地 Tile：
- `Dry` → `dryFarmlandTile`
- `WetDark` → `wetDarkTile`

无需额外修改日结逻辑。

#### 3.2.5 读档视觉恢复（AC-3.5）

`CreateTileFromSaveData()` 末尾已调用 `UpdateTileVisual()`。修改后的 `UpdateTileVisual()` 会根据 `moistureState` 设置正确的耕地 Tile，无需额外修改。

### 3.3 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `FarmVisualManager.cs` | `UpdateTileVisual()` 新增耕地 Tile 渲染，新增 `GradualMoistureTransition()` 协程 |
| `FarmTileManager.cs` | `CreateTile()` 末尾新增 `UpdateTileVisual()` 调用，`OnHourChanged()` 水渍消退后启动渐变协程 |


---

## 四、修改影响分析

> 本章列出所有被修改文件的影响范围，确保"新增并兼容"原则。

### 4.1 文件修改总览

| 文件 | 修改类型 | 关联 US |
|------|---------|---------|
| `GameInputManager.cs` | 新增字段 + 新增方法 + 修改阻断逻辑 | US-1 |
| `PlayerInteraction.cs` | `OnActionComplete()` 新增回调 | US-1 |
| `FarmToolPreview.cs` | `LockPosition()` 新增视觉刷新 | US-1 |
| `SaveDataDTOs.cs` | `FarmTileSaveData` 新增 3 字段 | US-2 |
| `FarmTileManager.cs` | `Save()`/`CreateTileFromSaveData()`/`CreateTile()` 适配 | US-2, US-3 |
| `FarmVisualManager.cs` | `UpdateTileVisual()` 扩展 + 新增渐变协程 | US-3 |

### 4.2 逐文件影响分析

#### 4.2.1 `GameInputManager.cs`

**新增内容**（不影响现有逻辑）：
- 3 个缓存字段：`_hasPendingFarmInput` / `_pendingFarmWorldPos` / `_pendingFarmItemId`
- `CacheFarmInput()` 方法：纯新增
- `ConsumePendingFarmInput()` 方法：纯新增
- `ProcessFarmInputAt()` 方法：纯新增，内部复用现有 `TryHandleFarmingTool` / `TryPlantSeed` 逻辑

**修改内容**（需要验证兼容性）：
- `HandleUseCurrentTool()` 第 614 行 `_isExecutingFarming` 阻断：从直接 return 改为缓存后 return
  - 影响范围：仅农田工具（Hoe/WateringCan/Seed），非农田工具不受影响
  - 兼容性：非农田工具走原有 return 路径，行为不变
- `HandleUseCurrentTool()` 第 654 行 `IsPerformingAction()` 阻断：农田工具走缓存路径
  - 影响范围：仅农田工具，非农田工具保持原有阻断
  - 兼容性：通过 `itemData is ToolData` 和 `toolType` 判断精确区分

**不受影响的功能**：
- 武器攻击、其他工具使用（斧头/镐子等非农田工具）
- 背包操作、UI 交互
- 导航系统（`StartFarmingNavigation` / `WaitForNavigationComplete` 不修改）

#### 4.2.2 `PlayerInteraction.cs`

**修改内容**：
- `OnActionComplete()` 末尾新增一行回调：`GameInputManager.Instance.ConsumePendingFarmInput()`
  - 影响范围：所有动作完成后都会触发此回调
  - 兼容性：`ConsumePendingFarmInput()` 内部首先检查 `_hasPendingFarmInput`，无缓存时直接 return，零开销
  - 长按继续分支同样触发，符合 AC-1.2 设计

**不受影响的功能**：
- `IsPerformingAction()` 返回值逻辑不变
- `RequestAction()` 不变
- 动画播放逻辑不变
- 长按检测逻辑不变（只是在长按继续后额外通知 GameInputManager）

#### 4.2.3 `FarmToolPreview.cs`

**修改内容**：
- `LockPosition()` 末尾新增一行：`UpdateCursor(layerIndex, cellPos)`
  - 影响范围：每次锁定预览时多一次视觉刷新
  - 兼容性：`UpdateCursor` 是现有方法，幂等操作，多调用一次无副作用
  - 性能：单次 Tilemap 操作，可忽略

**不受影响的功能**：
- `UnlockPosition()` 不变
- `UpdateHoePreview()` / `UpdateWateringPreview()` 不变
- 实时预览跟随鼠标的逻辑不变


#### 4.2.4 `SaveDataDTOs.cs`

**新增内容**（不影响现有逻辑）：
- `FarmTileSaveData` 新增 3 个 `public` 字段
  - 影响范围：仅影响 JSON 序列化输出（多 3 个键值对）
  - 兼容性：`JsonUtility.FromJson` 对旧存档中缺失的字段自动使用类型默认值
  - 不影响其他 DTO（`CropSaveData` / `PlacedObjectSaveData` 等）

#### 4.2.5 `FarmTileManager.cs`

**修改内容**：
- `Save()`：新增 3 个字段写入（`wateredYesterday` / `waterTime` / `puddleVariant`）
  - 影响范围：存档数据增加 3 个字段
  - 兼容性：纯追加，不修改现有 5 个字段的写入逻辑
- `CreateTileFromSaveData()`：新增 3 个字段恢复
  - 影响范围：读档时多恢复 3 个字段
  - 兼容性：旧存档缺失字段时使用默认值，不影响现有恢复逻辑
- `CreateTile()`：末尾新增 `UpdateTileVisual()` 调用
  - 影响范围：锄地成功后立即显示 `dryFarmlandTile`
  - 兼容性：`UpdateTileVisual` 是现有方法，新增耕地 Tile 渲染逻辑后才有实际效果
- `OnHourChanged()`：水渍消退后启动渐变协程
  - 影响范围：水渍消退时额外启动一个协程
  - 兼容性：不修改现有水渍消退逻辑，只在消退完成后追加渐变

**不受影响的功能**：
- `SetWatered()` 不变
- `OnDayChanged()` / `ResetDailyWaterState()` 不变（日结后 `RefreshAllTileVisuals` 会自动处理）
- 耕地创建/删除逻辑不变
- 作物相关逻辑不变

#### 4.2.6 `FarmVisualManager.cs`

**修改内容**：
- `UpdateTileVisual()`：新增耕地 Tile 渲染逻辑（根据 `moistureState` 设置 `dryFarmlandTile` 或 `wetDarkTile`）
  - 影响范围：每次调用 `UpdateTileVisual` 时额外操作 `farmlandCenterTilemap`
  - 兼容性：现有水渍叠加层逻辑完全不变，新增逻辑在其之后执行
- 新增 `GradualMoistureTransition()` 协程
  - 影响范围：水渍消退后启动，30 游戏分钟内渐变
  - 兼容性：纯新增，不影响现有任何方法

**不受影响的功能**：
- 水渍叠加层的显示/消退逻辑不变
- `RefreshAllTileVisuals()` 不变（内部调用 `UpdateTileVisual`，自动获得新逻辑）
- Inspector 配置的水渍 Tile 变体不变

### 4.3 跨模块影响矩阵

| 修改 | 农田工具 | 武器/其他工具 | 存档系统 | 视觉系统 | 导航系统 |
|------|---------|-------------|---------|---------|---------|
| 输入缓存 | ✅ 受益 | ❌ 不影响 | ❌ 不影响 | ❌ 不影响 | ❌ 不影响 |
| 存档字段 | ❌ 不影响 | ❌ 不影响 | ✅ 受益 | ✅ 间接受益 | ❌ 不影响 |
| 耕地视觉 | ❌ 不影响 | ❌ 不影响 | ❌ 不影响 | ✅ 受益 | ❌ 不影响 |
| LockPosition 视觉 | ✅ 受益 | ❌ 不影响 | ❌ 不影响 | ✅ 受益 | ❌ 不影响 |


---

## 五、正确性属性映射与测试策略

### 5.1 正确性属性 → 设计方案映射

| 属性 | 描述 | 覆盖设计节 | 验证方式 |
|------|------|-----------|---------|
| CP-1 | 动画期间输入不丢失 | 1.2.2（阻断改缓存）+ 1.2.9（执行中缓存） | PBT：随机输入序列 → 缓存状态断言 |
| CP-2 | 缓存最多一个，后覆盖前 | 1.2.3（CacheFarmInput 覆盖写入） | PBT：多次缓存 → 只保留最后一次 |
| CP-3 | 缓存消费 = 完整动作链 | 1.2.4（ConsumePendingFarmInput → ProcessFarmInputAt） | 单元测试：验证调用链 |
| CP-4 | 长按 = 当前位置单击 | 1.2.6（OnActionComplete 长按分支） | 单元测试：模拟长按 → 验证行为等价 |
| CP-5 | 预览=锁定=导航目的地 | 1.2.7 + 1.2.8（LockPosition 视觉同步） | PBT：随机导航序列 → 三者一致性 |
| CP-6 | LockPosition 后视觉=锁定位置 | 1.2.8（UpdateCursor 调用） | 单元测试：Lock → 验证视觉位置 |
| CP-7 | 非农田工具保护不变 | 1.2.2（类型判断精确区分） | 单元测试：非农田工具 → 原有阻断 |
| CP-8 | Save→Load 字段不变 | 2.2.2 + 2.2.3（8 字段完整读写） | PBT：随机 FarmTileData → 存读档 → 字段相等 |
| CP-9 | 旧存档兼容 | 2.2.4（JsonUtility 默认值） | 单元测试：5 字段 JSON → 加载不报错 |
| CP-10 | 状态→视觉一致 | 3.2.1（switch 分支） | PBT：随机状态 → 验证 Tile 匹配 |
| CP-11 | 渐变无瞬间跳变 | 3.2.3（协程渐变） | 集成测试：验证渐变过程连续 |
| CP-12 | 耕地状态独立 | 3.2.1（per-tile 操作） | PBT：随机多块耕地 → 修改一块 → 其他不变 |

### 5.2 测试策略

#### 5.2.1 测试框架

- 框架：NUnit（Unity Test Runner EditMode）
- PBT 库：FsCheck（通过 NuGet 引入）或手写随机生成器
- 测试位置：`Assets/YYY_Tests/Editor/`

#### 5.2.2 PBT 测试计划

**P1：存档往返测试（CP-8）**
```
属性：∀ FarmTileData d, Load(Save(d)) == d
生成器：随机 SoilMoistureState × 随机 bool × 随机 float[0,24] × 随机 int[0,2]
断言：8 个字段逐一比较
```

**P2：输入缓存覆盖测试（CP-1, CP-2）**
```
属性：∀ 输入序列 [i1, i2, ..., in]（动画期间），缓存 == in
生成器：随机长度 1~10 的 Vector3 序列 + 随机 itemId
断言：_hasPendingFarmInput == true, _pendingFarmWorldPos == 最后一个位置
```

**P3：耕地状态独立性测试（CP-12）**
```
属性：∀ 耕地集合 S, 修改 S[i] 的 moistureState 不改变 S[j] (j≠i)
生成器：随机 2~10 块耕地，随机选一块修改
断言：其他块的所有字段不变
```

**P4：状态→视觉映射测试（CP-10）**
```
属性：∀ SoilMoistureState s, UpdateTileVisual(s) 设置的 Tile == 预期 Tile
生成器：枚举所有 SoilMoistureState 值
断言：Dry/WetWithPuddle → dryFarmlandTile, WetDark → wetDarkTile
```

#### 5.2.3 单元测试计划

| 测试 | 验证内容 | 关联属性 |
|------|---------|---------|
| `CacheFarmInput_StoresLastInput` | 多次缓存只保留最后一次 | CP-2 |
| `ConsumePendingFarmInput_ClearsCache` | 消费后缓存清空 | CP-3 |
| `ConsumePendingFarmInput_SkipsOnItemMismatch` | 手持物品不一致时不执行 | E-2 |
| `LockPosition_RefreshesVisual` | 锁定后视觉位置正确 | CP-6 |
| `NonFarmTool_BlockedByIsPerformingAction` | 非农田工具仍被阻断 | CP-7 |
| `OldSaveData_LoadsWithDefaults` | 旧存档 5 字段正常加载 | CP-9 |
| `SaveLoad_PreservesAllFields` | 8 字段存读档一致 | CP-8 |
| `CreateTile_SetsDryFarmlandTile` | 锄地后显示干燥 Tile | AC-3.1 |
| `UpdateTileVisual_MatchesMoistureState` | 状态→Tile 映射正确 | CP-10 |
