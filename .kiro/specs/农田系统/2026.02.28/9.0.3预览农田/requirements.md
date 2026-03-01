# 9.0.3 预览农田 - 需求文档

## 概述

完善农田工具预览系统，解决以下核心问题：
1. 预览 Sorting Layer 硬编码，跨楼层时显示异常
2. 障碍物检测缺失，可在石头/树木上显示绿色预览
3. 种子预览缺席，手持种子时无任何反馈
4. 距离检测缺失，远处显示绿色但实际无法操作

---

## 用户故事

### US-1：动态空间感知（P0）

**作为** 玩家
**我希望** 农田工具预览能跟随我所在的楼层
**以便** 在多楼层场景中预览显示正确

**验收标准**：
- AC-1.1：预览 GhostTilemap 的 Sorting Layer 实时跟随玩家
- AC-1.2：使用 `PlacementLayerDetector.GetPlayerSortingLayer()` 获取玩家层级
- AC-1.3：玩家切换楼层时，预览自动切换到对应层级

---

### US-2：分级避障规则（P1）

**作为** 玩家
**我希望** 农田工具预览能正确检测障碍物
**以便** 知道哪些位置可以锄地/浇水

**验收标准**：
- AC-2.1：检测 Tree、Rock、Building 标签的障碍物
- AC-2.2：**不检测 Player 标签**（玩家站在地里应该能锄地）
- AC-2.3：有障碍物时显示红色预览
- AC-2.4：使用新方法 `HasFarmingObstacle()` 而非 `HasObstacle()`

**关键区别**：
| 方法 | 检测 Player | 用途 |
|------|-------------|------|
| `HasObstacle()` | ✅ 是 | 放置物品（箱子、树苗） |
| `HasFarmingObstacle()` | ❌ 否 | 农田工具（锄头、水壶） |

---

### US-3：种子预览支持（P2）

**作为** 玩家
**我希望** 手持种子时能看到种植预览
**以便** 知道哪些位置可以种植

**验收标准**：
- AC-3.1：手持 SeedData 时显示光标预览
- AC-3.2：绿色 = 可种植（已耕地、无作物）
- AC-3.3：红色 = 不可种植
- AC-3.4：GameInputManager 正确路由 SeedData 到 FarmToolPreview

---

### US-4：距离限制（P1）

**作为** 玩家
**我希望** 预览能反映实际操作距离
**以便** 不会看到绿色预览却无法操作

**验收标准**：
- AC-4.1：超出 `farmToolReach` 距离时显示红色
- AC-4.2：距离计算使用玩家 Collider 中心
- AC-4.3：预览与实际操作判定完全一致（WYSIWYG）

---

## 非功能需求

### NFR-1：性能
- 预览更新不应造成明显卡顿
- 避免每帧重复创建对象

### NFR-2：兼容性
- 与现有 PlacementPreview 系统互不干扰
- 与 FarmTileManager、FarmlandBorderManager 正确集成

---

## 依赖关系

| 依赖 | 说明 |
|------|------|
| `PlacementLayerDetector` | 获取玩家 Sorting Layer |
| `PlacementValidator` | 新增 `HasFarmingObstacle()` 方法 |
| `FarmTileManager` | 耕地状态查询 |
| `FarmlandBorderManager` | 1+8 预览 Tile 生成 |
