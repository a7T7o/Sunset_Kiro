# 9.0.3 预览农田 - 任务列表

## P0：动态 Sorting Layer（US-1）

- [x] 1. PlacementValidator 新增 HasFarmingObstacle 静态方法
  - [x] 1.1 定义 farmingObstacleTags 数组（Tree, Rock, Building，不含 Player）
  - [x] 1.2 实现 HasFarmingObstacle(Vector3 cellCenter) 静态方法
  - [x] 1.3 复用现有的 HasTreeAtPosition 和 HasChestAtPosition 检测

- [x] 2. FarmToolPreview 新增 UpdateSortingLayer 方法
  - [x] 2.1 新增 UpdateSortingLayer(Transform playerTransform) 方法
  - [x] 2.2 使用 PlacementLayerDetector.GetPlayerSortingLayer() 获取层级
  - [x] 2.3 同步更新 ghostTilemapRenderer 和 cursorRenderer 的 sortingLayerName

## P1：障碍物检测 + 距离检测（US-2, US-4）

- [x] 3. FarmToolPreview 集成障碍物检测
  - [x] 3.1 在 UpdateHoePreview 中调用 HasFarmingObstacle
  - [x] 3.2 在 UpdateWateringPreview 中调用 HasFarmingObstacle
  - [x] 3.3 有障碍物时设置 currentState = Invalid

- [x] 4. FarmToolPreview 集成距离检测
  - [x] 4.1 新增 farmToolReach 参数（从 GameInputManager 传入）
  - [x] 4.2 新增 IsWithinReach 方法
  - [x] 4.3 超出距离时设置 currentState = Invalid

- [x] 5. GameInputManager 修改 UpdateFarmToolPreview 调用
  - [x] 5.1 传递 playerTransform 参数
  - [x] 5.2 传递 farmToolReach 参数

## P2：种子预览支持（US-3）

- [x] 6. FarmToolPreview 新增 UpdateSeedPreview 方法
  - [x] 6.1 实现 UpdateSeedPreview(layerIndex, cellPos, seedData, playerTransform, reach)
  - [x] 6.2 检查障碍物、距离、是否可种植
  - [x] 6.3 只显示光标，不显示 1+8 预览

- [x] 7. GameInputManager 新增种子路由
  - [x] 7.1 在 UpdatePreviews 中检测 SeedData
  - [x] 7.2 新增 UpdateSeedPreview 方法
  - [x] 7.3 路由到 FarmToolPreview.UpdateSeedPreview

## 验证

- [ ] 8. 功能验证
  - [ ] 8.1 验证跨楼层时预览 Sorting Layer 正确
  - [ ] 8.2 验证石头/树木上显示红色预览
  - [ ] 8.3 验证玩家站在目标格子上时显示绿色（不被自己阻挡）
  - [ ] 8.4 验证超出距离时显示红色
  - [ ] 8.5 验证手持种子时显示预览
