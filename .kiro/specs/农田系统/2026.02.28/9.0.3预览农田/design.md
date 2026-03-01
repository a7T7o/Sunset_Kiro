# 9.0.3 预览农田 - 设计文档

## 架构概述

```
GameInputManager.UpdatePreviews()
        │
        ├─ ToolData (Hoe/WateringCan) ──► FarmToolPreview
        │                                      │
        │                                      ├─ UpdateSortingLayer()  [P0]
        │                                      ├─ CheckObstacle()       [P1]
        │                                      ├─ CheckDistance()       [P1]
        │                                      └─ UpdateCursor()
        │
        ├─ SeedData ────────────────────► FarmToolPreview.UpdateSeedPreview()  [P2]
        │
        └─ PlaceableItemData ───────────► PlacementPreview (现有)
```

---

## Logic Flow（调用链）

### 1. 预览更新流程

```
Update()
  └─ UpdatePreviews()
       ├─ 检查面板/UI状态 → 隐藏预览
       ├─ 获取手持物品
       │
       ├─ if (ToolData: Hoe/WateringCan)
       │    └─ UpdateFarmToolPreview(alignedPos, isHoe, playerTransform)
       │         ├─ UpdateSortingLayer(playerTransform)     ← P0 新增
       │         ├─ CheckFarmingObstacle(cellPos)           ← P1 新增
       │         ├─ CheckDistance(playerCenter, cellPos)    ← P1 新增
       │         └─ UpdateHoePreview() / UpdateWateringPreview()
       │
       ├─ if (SeedData)                                     ← P2 新增
       │    └─ UpdateSeedPreview(alignedPos, seedData, playerTransform)
       │         ├─ UpdateSortingLayer(playerTransform)
       │         ├─ CheckFarmingObstacle(cellPos)
       │         ├─ CheckDistance(playerCenter, cellPos)
       │         └─ CheckCanPlant(cellPos, seedData)
       │
       └─ else → HideAllPreviews()
```

### 2. 障碍物检测流程

```
FarmToolPreview.CheckFarmingObstacle(cellCenter)
  └─ PlacementValidator.HasFarmingObstacle(cellCenter)
       ├─ Physics2D.OverlapBoxAll(cellCenter, 0.9x0.9)
       │    └─ 检测 Tree, Rock, Building 标签
       │       ⚠️ 不检测 Player 标签！
       │
       ├─ HasTreeAtPosition(cellCenter)  ← 检测无碰撞体的树苗
       │
       └─ HasChestAtPosition(cellCenter) ← 检测无碰撞体的箱子
```

---

## 组件设计

### FarmToolPreview 修改

```csharp
public class FarmToolPreview : MonoBehaviour
{
    // ========== P0: 动态 Sorting Layer ==========
    
    /// <summary>
    /// 更新 GhostTilemap 和 CursorRenderer 的 Sorting Layer
    /// </summary>
    public void UpdateSortingLayer(Transform playerTransform)
    {
        string sortingLayer = PlacementLayerDetector.GetPlayerSortingLayer(playerTransform);
        
        if (ghostTilemapRenderer != null)
            ghostTilemapRenderer.sortingLayerName = sortingLayer;
        
        if (cursorRenderer != null)
            cursorRenderer.sortingLayerName = sortingLayer;
    }
    
    // ========== P1: 障碍物检测 ==========
    
    /// <summary>
    /// 检查是否有农田障碍物（排除 Player）
    /// </summary>
    private bool HasFarmingObstacle(Vector3 cellCenter)
    {
        return PlacementValidator.HasFarmingObstacle(cellCenter);
    }
    
    // ========== P1: 距离检测 ==========
    
    /// <summary>
    /// 检查是否在操作距离内
    /// </summary>
    private bool IsWithinReach(Vector2 playerCenter, Vector3 cellCenter, float reach)
    {
        return Vector2.Distance(playerCenter, cellCenter) <= reach;
    }
    
    // ========== P2: 种子预览 ==========
    
    /// <summary>
    /// 更新种子预览
    /// </summary>
    public void UpdateSeedPreview(int layerIndex, Vector3Int cellPos, 
                                   SeedData seedData, Transform playerTransform, 
                                   float reach)
    {
        // 1. 更新 Sorting Layer
        UpdateSortingLayer(playerTransform);
        
        // 2. 检查障碍物
        Vector3 cellCenter = GetCellCenterWorld(layerIndex, cellPos);
        if (HasFarmingObstacle(cellCenter))
        {
            currentState = FarmPreviewState.Invalid;
            UpdateCursor(layerIndex, cellPos);
            Show();
            return;
        }
        
        // 3. 检查距离
        Vector2 playerCenter = GetPlayerCenter(playerTransform);
        if (!IsWithinReach(playerCenter, cellCenter, reach))
        {
            currentState = FarmPreviewState.Invalid;
            UpdateCursor(layerIndex, cellPos);
            Show();
            return;
        }
        
        // 4. 检查是否可种植
        bool canPlant = FarmTileManager.Instance?.CanPlantAt(layerIndex, cellPos) ?? false;
        currentState = canPlant ? FarmPreviewState.Valid : FarmPreviewState.Invalid;
        
        // 5. 更新光标（种子模式不显示 1+8 预览）
        ClearGhostTilemap();
        UpdateCursor(layerIndex, cellPos);
        Show();
    }
}
```

### PlacementValidator 新增方法

```csharp
public class PlacementValidator
{
    /// <summary>农田障碍物标签（不含 Player）</summary>
    private static readonly string[] farmingObstacleTags = new string[] 
    { 
        "Tree", "Rock", "Building" 
        // ⚠️ 注意：不包含 "Player"
    };
    
    /// <summary>
    /// 检查是否有农田障碍物（排除 Player）
    /// 用于锄头、水壶、种子等农田工具
    /// </summary>
    public static bool HasFarmingObstacle(Vector3 cellCenter)
    {
        // 1. 碰撞体检测（排除 Player）
        Vector2 boxSize = new Vector2(0.9f, 0.9f);
        Collider2D[] hits = Physics2D.OverlapBoxAll(cellCenter, boxSize, 0f);
        
        foreach (var hit in hits)
        {
            if (HasAnyTag(hit.transform, farmingObstacleTags))
                return true;
        }
        
        // 2. 无碰撞体的树苗检测
        if (HasTreeAtPosition(cellCenter, 0.5f))
            return true;
        
        // 3. 无碰撞体的箱子检测
        if (HasChestAtPosition(cellCenter, 0.5f))
            return true;
        
        return false;
    }
}
```

### GameInputManager 路由修改

```csharp
private void UpdatePreviews()
{
    // ... 现有检查 ...
    
    if (itemData is ToolData tool)
    {
        switch (tool.toolType)
        {
            case ToolType.Hoe:
            case ToolType.WateringCan:
                HidePlacementPreview();
                // ★ 传递 playerTransform 用于 Layer 检测
                UpdateFarmToolPreview(alignedPos, tool.toolType == ToolType.Hoe, 
                                      playerMovement?.transform);
                return;
        }
    }
    else if (itemData is SeedData seedData)
    {
        // ★ P2 新增：种子路由到 FarmToolPreview
        HidePlacementPreview();
        UpdateSeedPreview(alignedPos, seedData, playerMovement?.transform);
        return;
    }
    // ...
}

/// <summary>
/// ★ P2 新增：更新种子预览
/// </summary>
private void UpdateSeedPreview(Vector3 alignedPos, SeedData seedData, Transform playerTransform)
{
    var farmPreview = FarmGame.Farm.FarmToolPreview.Instance;
    var farmTileManager = FarmGame.Farm.FarmTileManager.Instance;
    
    if (farmTileManager == null)
    {
        farmPreview.Hide();
        return;
    }
    
    int layerIndex = farmTileManager.GetCurrentLayerIndex(alignedPos);
    var tilemaps = farmTileManager.GetLayerTilemaps(layerIndex);
    
    if (tilemaps == null)
    {
        farmPreview.Hide();
        return;
    }
    
    Vector3Int cellPos = tilemaps.WorldToCell(alignedPos);
    farmPreview.UpdateSeedPreview(layerIndex, cellPos, seedData, playerTransform, farmToolReach);
}
```

---

## 关键设计决策

| 决策 | 原因 |
|------|------|
| 使用 `HasFarmingObstacle()` 而非 `HasObstacle()` | 农田工具不应被玩家自身阻挡 |
| 种子预览只显示光标，不显示 1+8 | 种子是单格操作，无需边界预览 |
| 距离检测使用玩家 Collider 中心 | 与实际操作判定一致（WYSIWYG） |
| Sorting Layer 实时跟随玩家 | 支持多楼层场景 |

---

## 文件修改清单

| 文件 | 修改内容 |
|------|---------|
| `FarmToolPreview.cs` | 新增 UpdateSortingLayer、障碍物检测、距离检测、种子预览 |
| `PlacementValidator.cs` | 新增静态方法 `HasFarmingObstacle()` |
| `GameInputManager.cs` | 修改 UpdateFarmToolPreview 传参、新增 UpdateSeedPreview |
