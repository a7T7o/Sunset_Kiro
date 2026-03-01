# 9.0.2 农田放置系统 - 任务清单

**创建时间**: 2026-02-06
**工作区**: 9.0.2放置农田

---

## Phase 1: 核心重构 (FarmlandBorderManager)

### 1.1 提取纯计算逻辑

- [x] 1.1.1 重构 `IsCenterBlock` 方法，支持传入 `Func<Vector3Int, bool> isTilledPredicate` 参数
- [x] 1.1.2 重构 `CheckNeighborCenters` 方法，支持断言参数
- [x] 1.1.3 新增 `CalculateBorderTileAt(layer, pos, predicate)` 纯计算方法
- [x] 1.1.4 新增 `ApplyBorderTile(layer, pos, tile)` 应用方法
- [x] 1.1.5 重构 `UpdateBorderAt` 内部调用新方法，保持原有行为不变

### 1.2 实现预览接口

- [x] 1.2.1 新增 `GetPreviewTiles(layerIndex, centerPos)` 方法
- [x] 1.2.2 实现断言构造逻辑（假装 centerPos 已耕作）
- [x] 1.2.3 实现中心块计算
- [x] 1.2.4 实现周围 8 格边界变化计算
- [x] 1.2.5 返回 `Dictionary<Vector3Int, TileBase>` 结果

### 1.3 单元测试

- [ ] 1.3.1 测试 `CalculateBorderTileAt` 在不同邻接情况下的返回值
- [ ] 1.3.2 测试 `GetPreviewTiles` 返回的数据完整性（包含中心块 + 边界）
- [ ] 1.3.3 测试预览计算不修改真实数据（无副作用验证）

---

## Phase 2: 预览组件实现 (FarmToolPreview)

### 2.1 创建组件基础

- [x] 2.1.1 创建 `FarmToolPreview.cs` 脚本
- [x] 2.1.2 定义 `FarmPreviewState` 枚举（Hidden, Valid, Invalid）
- [x] 2.1.3 添加 `cursorRenderer` (SpriteRenderer) 引用
- [x] 2.1.4 添加 `ghostTilemap` (Tilemap) 引用
- [x] 2.1.5 实现单例模式或服务定位

### 2.2 实现锄头预览

- [x] 2.2.1 实现 `UpdateHoePreview(layerIndex, cellPos)` 方法
- [x] 2.2.2 调用 `FarmTileManager.CanTillAt` 检查是否可锄
- [x] 2.2.3 调用 `FarmlandBorderManager.GetPreviewTiles` 获取预览数据
- [x] 2.2.4 将预览数据渲染到 `ghostTilemap`
- [x] 2.2.5 设置光标颜色（绿色=可锄，红色=不可锄）

### 2.3 实现水壶预览

- [x] 2.3.1 实现 `UpdateWateringPreview(layerIndex, cellPos)` 方法
- [x] 2.3.2 检查目标格子是否已耕作且干燥
- [x] 2.3.3 设置光标颜色（蓝色=可浇水，红色=不可浇水）
- [x] 2.3.4 水壶预览不显示 1+8 边界变化

### 2.4 创建预制体

- [ ] 2.4.1 创建 `FarmToolPreview` 预制体
- [ ] 2.4.2 配置 `ghostTilemap` 的 Sorting Layer 和 Order
- [ ] 2.4.3 配置 `cursorRenderer` 的 Sprite 和颜色
- [ ] 2.4.4 设置预览透明度（Alpha=0.5）

---

## Phase 3: 输入系统接入 (GameInputManager)

### 3.1 统一坐标系

- [x] 3.1.1 移除 `GameInputManager` 中私有的坐标转换逻辑
- [x] 3.1.2 统一使用 `PlacementGridCalculator.GetMouseWorldPosition()`
- [x] 3.1.3 统一使用 `PlacementGridCalculator.WorldToCell()`

### 3.2 预览分发逻辑

- [x] 3.2.1 在 Update 中判断手持物品类型
- [x] 3.2.2 手持可放置物品 → 调用 `PlacementPreview`
- [x] 3.2.3 手持农具 → 调用 `FarmToolPreview`
- [x] 3.2.4 其他情况 → 隐藏所有预览

### 3.3 点击处理重构

- [x] 3.3.1 点击时检查 `FarmToolPreview.IsValid()`
- [x] 3.3.2 只有预览有效时才执行农田操作
- [x] 3.3.3 移除旧的 Raycast 判定逻辑

---

## Phase 4: 集成测试与验收

### 4.1 功能验证

- [ ] 4.1.1 验证锄头预览显示 1+8 边界变化
- [ ] 4.1.2 验证水壶预览显示正确的光标颜色
- [ ] 4.1.3 验证预览与实际执行结果一致
- [ ] 4.1.4 验证坐标对齐与放置系统一致

### 4.2 边界情况测试

- [ ] 4.2.1 测试在已耕作格子上锄地（应显示红色）
- [ ] 4.2.2 测试在地图边缘锄地（边界计算正确）
- [ ] 4.2.3 测试快速移动鼠标时预览更新流畅
- [ ] 4.2.4 测试切换工具时预览正确切换

### 4.3 性能验证

- [ ] 4.3.1 验证预览更新不产生明显卡顿
- [ ] 4.3.2 验证无 GC Alloc（或在可接受范围内）

---

## 任务依赖关系

```
Phase 1 (核心重构)
    │
    ├── 1.1 提取纯计算逻辑
    │       │
    │       └── 1.2 实现预览接口
    │               │
    │               └── 1.3 单元测试
    │
    ▼
Phase 2 (预览组件)
    │
    ├── 2.1 创建组件基础
    │       │
    │       ├── 2.2 实现锄头预览
    │       │
    │       └── 2.3 实现水壶预览
    │
    └── 2.4 创建预制体
    │
    ▼
Phase 3 (输入接入)
    │
    ├── 3.1 统一坐标系
    │       │
    │       └── 3.2 预览分发逻辑
    │               │
    │               └── 3.3 点击处理重构
    │
    ▼
Phase 4 (集成测试)
```

---

## 预计工时

| Phase | 任务数 | 预计时间 |
|-------|--------|---------|
| Phase 1 | 13 | 2-3 小时 |
| Phase 2 | 14 | 2-3 小时 |
| Phase 3 | 9 | 1-2 小时 |
| Phase 4 | 9 | 1-2 小时 |
| **总计** | **45** | **6-10 小时** |
