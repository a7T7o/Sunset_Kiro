# 9.0.2 农田放置系统 - 设计文档

**创建时间**: 2026-02-06
**工作区**: 9.0.2放置农田
**核心架构**: 联邦制预览系统 + 断言模拟

---

## 1. 架构概览

### 1.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        GameInputManager                          │
│                     (输入分发中心)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ PlacementManager │ │ FarmTileManager │ │ 其他系统...     │
│ (可放置物品)     │ │ (农田工具)      │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
        │                     │
        ▼                     ▼
┌─────────────────┐ ┌─────────────────┐
│ PlacementPreview │ │ FarmToolPreview │  ← 独立的预览组件
│ (NxM 放置预览)   │ │ (1+8 农田预览)  │
└─────────────────┘ └─────────────────┘
        │                     │
        └──────────┬──────────┘
                   ▼
        ┌─────────────────────┐
        │PlacementGridCalculator│  ← 共享的坐标计算
        │   (统一坐标系)        │
        └─────────────────────┘
```

### 1.2 核心设计原则

1. **联邦制**：`FarmToolPreview` 独立于 `PlacementPreview`，互不干扰
2. **统一坐标**：所有系统共享 `PlacementGridCalculator` 进行坐标计算
3. **断言模拟**：通过 `Func` 委托实现无副作用的预览计算

---

## 2. FarmlandBorderManager 重构设计

### 2.1 现状问题

当前 `FarmlandBorderManager` 的核心方法 `UpdateBorderAt` 存在两个耦合：

1. **状态查询耦合**：直接调用 `FarmTileManager.Instance.GetTileData` 查询真实数据
2. **副作用耦合**：直接调用 `tilemap.SetTile` 修改游戏世界

这导致无法实现"只计算不修改"的预览功能。

### 2.2 重构方案：能力解耦

#### 2.2.1 引入断言参数

将硬编码的数据查询替换为可注入的断言：

```csharp
// 重构前
private bool IsCenterBlock(int layerIndex, Vector3Int position)
{
    var tileData = FarmTileManager.Instance?.GetTileData(layerIndex, position);
    return tileData != null && tileData.isTilled;
}

// 重构后
private bool IsCenterBlock(int layerIndex, Vector3Int position, 
                           Func<Vector3Int, bool> isTilledPredicate = null)
{
    if (isTilledPredicate != null)
        return isTilledPredicate(position);
    
    var tileData = FarmTileManager.Instance?.GetTileData(layerIndex, position);
    return tileData != null && tileData.isTilled;
}
```

#### 2.2.2 拆分计算与应用

```csharp
// 纯计算方法（无副作用）
public TileBase CalculateBorderTileAt(
    int layerIndex, 
    Vector3Int targetPos, 
    Func<Vector3Int, bool> isTilledPredicate)
{
    var neighbors = CheckNeighborCentersWithPredicate(layerIndex, targetPos, isTilledPredicate);
    return SelectBorderTile(neighbors.hasU, neighbors.hasD, neighbors.hasL, neighbors.hasR);
}

// 应用方法（有副作用）
public void ApplyBorderTile(int layerIndex, Vector3Int pos, TileBase tile)
{
    var tilemaps = GetLayerTilemaps(layerIndex);
    tilemaps?.farmlandBorderTilemap?.SetTile(pos, tile);
}
```

#### 2.2.3 新增预览接口

```csharp
/// <summary>
/// 获取 1+8 预览数据：如果在 centerPos 锄地，会产生什么变化
/// </summary>
/// <param name="layerIndex">楼层索引</param>
/// <param name="centerPos">锄地位置</param>
/// <returns>位置 -> Tile 的映射，包含中心块和周围边界变化</returns>
public Dictionary<Vector3Int, TileBase> GetPreviewTiles(int layerIndex, Vector3Int centerPos)
{
    var result = new Dictionary<Vector3Int, TileBase>();
    
    // 创建"假装 centerPos 已耕作"的断言
    Func<Vector3Int, bool> previewPredicate = pos =>
    {
        if (pos == centerPos) return true; // 假装这里已耕作
        var tileData = FarmTileManager.Instance?.GetTileData(layerIndex, pos);
        return tileData != null && tileData.isTilled;
    };
    
    // 1. 添加中心块
    result[centerPos] = centerTileUnfertilized;
    
    // 2. 计算周围 8 个位置的边界变化
    for (int dx = -1; dx <= 1; dx++)
    {
        for (int dy = -1; dy <= 1; dy++)
        {
            if (dx == 0 && dy == 0) continue;
            
            Vector3Int neighborPos = centerPos + new Vector3Int(dx, dy, 0);
            
            // 如果邻居已经是中心块，跳过（不需要变化）
            if (previewPredicate(neighborPos) && neighborPos != centerPos) continue;
            
            // 计算该位置的边界 Tile
            TileBase borderTile = CalculateBorderTileAt(layerIndex, neighborPos, previewPredicate);
            
            if (borderTile != null)
                result[neighborPos] = borderTile;
        }
    }
    
    return result;
}
```

### 2.3 方法签名汇总

| 方法 | 类型 | 用途 |
|------|------|------|
| `CalculateBorderTileAt(layer, pos, predicate)` | 纯计算 | 计算指定位置的边界 Tile |
| `ApplyBorderTile(layer, pos, tile)` | 有副作用 | 将 Tile 写入 Tilemap |
| `GetPreviewTiles(layer, centerPos)` | 纯计算 | 获取 1+8 预览数据 |
| `UpdateBorderAt(layer, pos)` | 有副作用 | 原有方法，内部调用上述方法 |

---

## 3. FarmToolPreview 组件设计

### 3.1 组件结构

```
FarmToolPreview (MonoBehaviour)
│
├── [SerializeField] SpriteRenderer cursorRenderer  // 光标（绿/红/蓝框）
├── [SerializeField] Tilemap ghostTilemap           // 半透明预览层
├── [SerializeField] float previewAlpha = 0.5f      // 预览透明度
│
├── 状态
│   ├── bool isActive                               // 是否激活
│   ├── Vector3Int currentCell                      // 当前预览格子
│   └── PreviewState state                          // Valid/Invalid
│
└── 方法
    ├── void Show(ToolType toolType)                // 显示预览
    ├── void Hide()                                 // 隐藏预览
    ├── void UpdatePreview(Vector3 worldPos)        // 更新预览位置
    └── bool IsValid()                              // 当前是否可操作
```

### 3.2 预览状态枚举

```csharp
public enum FarmPreviewState
{
    Hidden,     // 隐藏
    Valid,      // 可操作（绿色/蓝色）
    Invalid     // 不可操作（红色）
}
```

### 3.3 核心逻辑流程

```csharp
public void UpdatePreview(int layerIndex, Vector3 worldPos)
{
    // 1. 坐标对齐
    Vector3Int cellPos = PlacementGridCalculator.WorldToCell(worldPos);
    Vector3 cellCenter = PlacementGridCalculator.GetCellCenter(cellPos);
    
    // 2. 更新光标位置
    cursorRenderer.transform.position = cellCenter;
    
    // 3. 清除旧预览
    ghostTilemap.ClearAllTiles();
    
    // 4. 根据工具类型判断
    if (currentToolType == ToolType.Hoe)
    {
        UpdateHoePreview(layerIndex, cellPos);
    }
    else if (currentToolType == ToolType.WateringCan)
    {
        UpdateWateringPreview(layerIndex, cellPos);
    }
}

private void UpdateHoePreview(int layerIndex, Vector3Int cellPos)
{
    bool canTill = FarmTileManager.Instance.CanTillAt(layerIndex, cellPos);
    
    if (canTill)
    {
        // 获取 1+8 预览数据
        var previewData = FarmlandBorderManager.Instance.GetPreviewTiles(layerIndex, cellPos);
        
        // 渲染半透明预览
        foreach (var kvp in previewData)
        {
            ghostTilemap.SetTile(kvp.Key, kvp.Value);
        }
        
        SetState(FarmPreviewState.Valid);
        cursorRenderer.color = Color.green;
    }
    else
    {
        SetState(FarmPreviewState.Invalid);
        cursorRenderer.color = Color.red;
    }
}
```

### 3.4 层级配置

| 组件 | Sorting Layer | Order in Layer |
|------|---------------|----------------|
| ghostTilemap | Ground | 100（高于真实 Tilemap） |
| cursorRenderer | Effects | 0 |

### 3.5 透明度处理

```csharp
private void SetGhostTilemapAlpha(float alpha)
{
    var color = ghostTilemap.color;
    color.a = alpha;
    ghostTilemap.color = color;
}
```

---

## 4. GameInputManager 接入设计

### 4.1 调用链重组

```csharp
// 在 Update 中
void Update()
{
    // 1. 获取统一坐标
    Vector3 mouseWorld = PlacementGridCalculator.GetMouseWorldPosition();
    
    // 2. 根据手持物品类型分发
    if (IsHoldingPlaceableItem())
    {
        PlacementPreview.Instance.UpdatePreview(mouseWorld);
        FarmToolPreview.Instance.Hide();
    }
    else if (IsHoldingFarmTool())
    {
        FarmToolPreview.Instance.UpdatePreview(currentLayerIndex, mouseWorld);
        PlacementPreview.Instance.Hide();
    }
    else
    {
        PlacementPreview.Instance.Hide();
        FarmToolPreview.Instance.Hide();
    }
}

// 点击处理
void OnLeftClick()
{
    if (IsHoldingFarmTool() && FarmToolPreview.Instance.IsValid())
    {
        ExecuteFarmAction();
    }
    else if (IsHoldingPlaceableItem() && PlacementPreview.Instance.IsValid())
    {
        ExecutePlacement();
    }
}
```

### 4.2 废弃的代码

以下代码需要移除或重构：

- `GameInputManager` 中私有的 `GetMouseWorldPosition` 方法
- 直接使用 `Camera.main.ScreenToWorldPoint` 的逻辑
- 农田工具的独立 Raycast 逻辑

---

## 5. 数据流图

```
用户移动鼠标
      │
      ▼
GameInputManager.Update()
      │
      ├─ 获取鼠标世界坐标 ─────────────────────────────────┐
      │  PlacementGridCalculator.GetMouseWorldPosition()   │
      │                                                    │
      ├─ 判断手持物品类型                                   │
      │                                                    │
      ▼                                                    │
[手持锄头]                                                 │
      │                                                    │
      ▼                                                    │
FarmToolPreview.UpdatePreview(layerIndex, worldPos)        │
      │                                                    │
      ├─ 坐标对齐 ◄────────────────────────────────────────┘
      │  PlacementGridCalculator.WorldToCell()
      │
      ├─ 检查是否可锄
      │  FarmTileManager.CanTillAt()
      │
      ├─ [可锄] 获取预览数据
      │  FarmlandBorderManager.GetPreviewTiles()
      │         │
      │         ├─ 构造断言 predicate
      │         ├─ 计算中心块
      │         └─ 计算 8 个边界变化
      │
      └─ 渲染预览
         ghostTilemap.SetTile()
```

---

## 6. 正确性属性

### P1: 坐标一致性

**属性**: 农田工具的操作坐标必须与放置系统的坐标计算结果一致

**验证方法**: 
```csharp
// 对于任意鼠标位置 mousePos
Vector3Int farmCell = FarmToolPreview.GetCurrentCell();
Vector3Int placeCell = PlacementPreview.GetCurrentCell();
Assert(farmCell == placeCell);
```

### P2: 预览无副作用

**属性**: 调用 `GetPreviewTiles` 不会修改任何真实数据

**验证方法**:
```csharp
// 记录调用前的状态
var beforeState = FarmTileManager.SerializeState();
// 调用预览
FarmlandBorderManager.GetPreviewTiles(layer, pos);
// 验证状态未变
var afterState = FarmTileManager.SerializeState();
Assert(beforeState == afterState);
```

### P3: 1+8 完整性

**属性**: 预览数据必须包含中心块和所有受影响的边界块

**验证方法**:
```csharp
var preview = FarmlandBorderManager.GetPreviewTiles(layer, centerPos);
// 必须包含中心块
Assert(preview.ContainsKey(centerPos));
// 边界块数量 <= 8
Assert(preview.Count <= 9);
```

### P4: 预览与执行一致

**属性**: 预览显示的结果必须与实际执行后的结果一致

**验证方法**:
```csharp
// 获取预览
var preview = FarmlandBorderManager.GetPreviewTiles(layer, pos);
// 执行锄地
FarmTileManager.TillAt(layer, pos);
// 验证结果一致
foreach (var kvp in preview)
{
    var actualTile = tilemap.GetTile(kvp.Key);
    Assert(actualTile == kvp.Value);
}
```

---

## 7. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 重构破坏现有功能 | 高 | 保持 `UpdateBorderAt` 原有行为，内部调用新方法 |
| 预览性能问题 | 中 | 使用缓存，避免每帧重复计算相同位置 |
| 透明度渲染问题 | 低 | 使用独立的 Tilemap，设置正确的 Sorting Order |
