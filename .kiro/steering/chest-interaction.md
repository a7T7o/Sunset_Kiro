---
inclusion: manual
priority: P1
keywords: [箱子, 交互, ChestController, BoxPanel, 锁, 钥匙, Chest]
lastUpdated: 2026-02-12
---

# 箱子/交互系统规则

> 本文件基于箱子系统 22 个会话的开发经验编写，所有规则均经过代码验证。

---

## 一、箱子层级结构

箱子采用父子层级结构：

```
箱子父物体（PlaceableObject + PersistentObject）
  └── Box 子物体（ChestController + SpriteRenderer + Collider）
```

| 组件 | 位置 | 职责 |
|------|------|------|
| PlaceableObject | 父物体 | 放置系统管理（位置、旋转、Sorting Layer） |
| PersistentObject | 父物体 | 存档系统注册（GUID、注册/注销） |
| ChestController | Box 子物体 | 箱子核心逻辑（库存、交互、UI） |
| SpriteRenderer | Box 子物体 | 箱子外观渲染 |
| Collider | Box 子物体 | 物理碰撞和交互检测 |

### 🔴 易错点
获取 ChestController 时不能在父物体上 `GetComponent`，必须用 `GetComponentInChildren` 或直接引用子物体。

---

## 二、4 种箱子类型

| 类型 | 列数 | 行数 | 容量 |
|------|------|------|------|
| 木质小箱 | 12 | 1 | 12 |
| 木质大箱 | 12 | 2 | 24 |
| 铁质小箱 | 12 | 1 | 12 |
| 铁质大箱 | 12 | 2 | 24 |

所有箱子固定 12 列布局（与背包一致），大箱 = 2 行，小箱 = 1 行。

---

## 三、UI 互斥机制

BoxPanel（箱子面板）与 PackagePanel（背包面板）的互斥关系：

| 操作 | Main 区域 | Top 区域 |
|------|-----------|---------|
| 打开箱子 | BoxPanel 显示 | PackagePanel 切换到 Top 模式 |
| 关闭箱子 | BoxPanel 隐藏 | PackagePanel 恢复 Main 模式 |

### 关键约束
- BoxPanelUI.Open() 禁止调用 ClosePackagePanel()
- 所有互斥逻辑必须集中在 PackagePanelTabsUI 中
- panelRoot 在箱子打开时必须保持 active

---

## 四、存档集成

### Initialize/Load 顺序陷阱

```
放置新箱子：Initialize() → 创建空 inventory → 正常
读档恢复：DynamicObjectFactory.TryReconstruct() → Load(data) → [下一帧] Start()/Initialize()
```

- `Initialize()` 中必须检查 `_inventory != null`，避免覆盖 `Load()` 已恢复的数据
- 这是箱子系统实际踩过的坑（3.7.6 会话中修复）
- 详见 `save-system.md` 的"执行顺序陷阱"章节

### 存档数据
- `ChestSaveData`：箱子类型、容量、物品列表
- 嵌套在 `WorldObjectSaveData` 中，通过 `objectType = "Chest"` 标识

---

## 五、交互系统

### 距离检测
- 使用 Collider 底部中心（不是 bounds.center）作为交互点
- 玩家位置使用 `playerCollider.bounds.center`（遵守最高优先级规则）
- Sprite Bounds 用于可视化检测（高亮、鼠标悬停）

### 导航集成
- 点击箱子 → 自动导航到交互距离 → 到达后自动打开
- 导航目标点 = Collider 底部中心偏移

---

## 六、锁系统

| 规则 | 说明 |
|------|------|
| 锁/钥匙配对 | 每把锁对应特定钥匙 |
| 归属转移 | 箱子可以转移归属权 |
| 玩家箱子 | 即使上锁也可直接打开（不需要钥匙） |
| 其他玩家箱子 | 必须有对应钥匙才能打开 |

---

## 七、已知未修复问题

1. Up 区域未接入 InventoryInteractionManager — 拖拽操作在 Up 区域不生效
2. 拖拽与修饰键两套系统不互通 — Shift+点击和拖拽是独立的交互路径
