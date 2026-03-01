# 补丁003 设计文档

## 概述

修复002验收后的6个严重问题 + 预览系统全面改造。核心改动涉及：作物位置架构、动画帧触发机制、队列执行逻辑、预览视觉系统。

---

## 一、P1/P5 作物位置修复：视觉子物体架构

### 1.1 问题根因

CropController 的 SpriteRenderer 在自身上（`GetComponent<SpriteRenderer>`），`AlignSpriteBottom` 修改 `transform.localPosition` 直接覆盖了 `Instantiate` 设置的正确世界坐标。

### 1.2 设计方案

学习 TreeController 的架构，为 CropController 创建视觉子物体：

```
改造前：
  Crop_xxx (自身) ← SpriteRenderer + CropController
  
改造后：
  Crop_xxx (自身) ← CropController（位置 = 格子中心 = 预览中心）
  └─ Visual (子物体) ← SpriteRenderer
```

### 1.3 修改清单

| 文件 | 修改内容 |
|------|---------|
| CropController.cs | 移除 `RequireComponent(typeof(SpriteRenderer))`，`Awake` 中创建 Visual 子物体并添加 SpriteRenderer，`spriteRenderer` 指向子物体 |
| CropController.cs | `AlignSpriteBottom` 无需修改（修改的是 `spriteRenderer.transform.localPosition`，现在指向子物体） |

### 1.4 兼容性考虑

- 存档加载：`DynamicObjectFactory.TryReconstructCrop` 通过 `Instantiate(cropPrefab)` 创建，`Awake` 会自动创建子物体，无需额外处理
- Prefab 结构：现有 Prefab 上的 SpriteRenderer 组件需要移除（改为运行时创建），或保留但在 Awake 中检测并迁移到子物体
- 碰撞体：CropController 如果有碰撞体，保持在父物体上不受影响

### 1.5 推荐实现：运行时创建子物体

在 `Awake` 中检测 SpriteRenderer 是否在自身上，如果是则创建子物体并迁移：

```
Awake():
  var sr = GetComponent<SpriteRenderer>();
  if (sr != null) {
    // SpriteRenderer 在自身上 → 创建子物体迁移
    var visual = new GameObject("Visual");
    visual.transform.SetParent(transform, false);
    var newSR = visual.AddComponent<SpriteRenderer>();
    // 复制 sr 的关键属性到 newSR
    // 销毁自身的 sr
    spriteRenderer = newSR;
  } else {
    // 已经是子物体结构（兼容未来 Prefab 更新）
    spriteRenderer = GetComponentInChildren<SpriteRenderer>();
  }
```

这样不需要修改现有 Prefab，运行时自动迁移。

---

## 二、P3 队列逻辑修复：动画帧触发 tile 更新

### 2.1 问题根因

`ExecuteFarmAction` 的 Till/Water 分支中，`ExecuteTillSoil`/`ExecuteWaterTile` 与 `RequestAction`（播放动画）在同一帧同步执行，导致 tile 在动画开始瞬间就完成变化。

### 2.2 设计方案：延迟执行机制

引入"待执行逻辑"概念，tile 更新延迟到动画进度达到指定比例时触发。

核心流程：
```
ExecuteFarmAction(Till):
  ① FaceTarget → ② RequestAction(Crush) → ③ 记录待执行请求（不立即执行）

Update 或协程监听:
  当 animController.GetAnimationProgress() >= tileUpdateTriggerProgress 时:
    → 执行 ExecuteTillSoil / ExecuteWaterTile
    → 标记已触发（防止重复）

OnActionComplete:
  → 正常走队列流程（OnFarmActionAnimationComplete）
```

### 2.3 关键参数

| 参数 | 值 | 说明 |
|------|---|------|
| `tileUpdateTriggerProgress` | 0.5f | 动画进度50%时触发（第四帧/共八帧） |
| `toolAnimationDuration` | 0.8f | 工具动画总时长（已有） |

### 2.4 修改清单

| 文件 | 修改内容 |
|------|---------|
| GameInputManager.cs | `ExecuteFarmAction` Till/Water 分支移除同步执行逻辑，改为记录待执行请求 |
| GameInputManager.cs | 新增 `_pendingTileUpdate` 字段和 `Update` 中的进度监听逻辑 |
| GameInputManager.cs | 新增 `tileUpdateTriggerProgress` 配置字段（Inspector 可调） |

### 2.5 松开分支时序修复

`PlayerInteraction.OnActionComplete` 松开分支中，调整执行顺序：

```
改造前（错误）：
  gimRelease.OnFarmActionAnimationComplete();  // ← 触发 ProcessNextAction
  // ... 其他清理 ...
  isPerformingAction = false;                  // ← 太晚！新动画被守卫拦截

改造后（正确）：
  isPerformingAction = false;                  // ← 先解除守卫
  // ... 其他清理 ...
  gimRelease.OnFarmActionAnimationComplete();  // ← 再触发下一个
```

| 文件 | 修改内容 |
|------|---------|
| PlayerInteraction.cs | 松开分支中 `isPerformingAction = false` 移到 `OnFarmActionAnimationComplete()` 之前 |


---

## 三、P2 长按退化修复

### 3.1 问题根因

`HandleUseCurrentTool` 第703行用 `GetMouseButtonDown(0)` 而非 `GetMouseButton(0)`，长按只触发一帧。

### 3.2 设计方案

将入口检测改为 `GetMouseButton(0)`，同时增加防重复入队逻辑：

```
HandleUseCurrentTool:
  bool isHolding = Input.GetMouseButton(0);
  bool isFirstPress = Input.GetMouseButtonDown(0);
  
  if (!isHolding) return;
  
  // 首次按下：正常入队
  // 持续按住：检查当前鼠标位置是否已在队列中，未入队则入队
  // 持续按住同一位置：不重复入队（由 OnActionComplete 长按分支处理重复执行）
```

### 3.3 长按同一位置的连续执行

`OnActionComplete` 长按分支中，动画完成后如果队列为空且仍在长按，重新检测当前鼠标位置并入队：

```
OnActionComplete 长按分支:
  isPerformingAction = false;
  OnFarmActionAnimationComplete();
  // 如果队列为空且仍在长按 → 重新入队当前位置
```

### 3.4 修改清单

| 文件 | 修改内容 |
|------|---------|
| GameInputManager.cs | `HandleUseCurrentTool` 入口检测改为 `GetMouseButton` + 防重复入队 |
| GameInputManager.cs | 长按分支增加队列为空时的重新入队逻辑 |

---

## 四、P4 导航入队统一

### 4.1 设计方案

简化 `HandleUseCurrentTool` 中的导航状态分支：无论当前是否在导航，所有左键点击都走入队逻辑。

```
改造前：
  if (导航中) {
    if (队列处理中) → 入队
    else → 中断导航重新开始
  }

改造后：
  if (导航中 || 执行中) {
    → 统一入队（TryEnqueueFromCurrentInput）
    return;
  }
```

### 4.2 修改清单

| 文件 | 修改内容 |
|------|---------|
| GameInputManager.cs | `HandleUseCurrentTool` 导航状态分支简化为统一入队 |

---

## 五、P6 预览系统全面改造

### 5.1 架构设计

预览系统改造为三层结构：

```
FarmToolPreview
├─ 鼠标跟随预览层（实时更新）
│   ├─ 耕地：整个tile颜色叠加（绿=可交互，红=不可交互）
│   ├─ 浇水：显示浇水后图案 + 颜色叠加
│   ├─ 种子：显示作物第一阶段sprite + 颜色叠加
│   └─ 收获：无预览
│
├─ 队列锁定预览层（入队时添加，执行完移除）
│   └─ 无颜色叠加的带透明度tile（当前样式）
│
└─ 光标层
    └─ 仅种子模式保留光标框（耕地/浇水/收获去除）
```

### 5.2 颜色叠加实现

使用 GhostTilemap 的 Tile 颜色属性：

- 可交互：`ghostTilemap.SetColor(pos, new Color(0, 1, 0, 0.3f))` 绿色半透明叠加
- 不可交互：`ghostTilemap.SetColor(pos, new Color(1, 0, 0, 0.3f))` 红色半透明叠加
- 队列锁定：`ghostTilemap.SetColor(pos, new Color(1, 1, 1, 0.5f))` 白色半透明（当前样式）

### 5.3 浇水图案预览

从 `FarmlandBorderManager` 或 `LayerTilemaps.waterPuddleTilemapNew` 获取水渍 Tile，在 GhostTilemap 上预览显示。

### 5.4 种子作物预览

从 `SeedData.cropPrefab` 获取 CropController 的 `stages[0].normalSprite`，创建临时 SpriteRenderer 显示在预览位置。需要新增一个 `_seedPreviewSpriteRenderer` 字段。

### 5.5 多位置队列预览

扩展 `FarmToolPreview`：

- 新增 `_queuedPreviewPositions` 列表，记录所有队列中的预览位置
- `AddQueuePreview(cellPos, layerIndex)`：入队时调用，在该位置显示锁定预览
- `RemoveQueuePreview(cellPos)`：执行完成时调用，移除该位置预览
- `ClearAllQueuePreviews()`：清空队列时调用

### 5.6 修改清单

| 文件 | 修改内容 |
|------|---------|
| FarmToolPreview.cs | 新增颜色叠加逻辑（替代光标方框） |
| FarmToolPreview.cs | 新增浇水图案预览 |
| FarmToolPreview.cs | 新增种子作物sprite预览（`_seedPreviewSpriteRenderer`） |
| FarmToolPreview.cs | 新增多位置队列预览管理（`_queuedPreviewPositions`） |
| FarmToolPreview.cs | 去除耕地/浇水/收获的光标方框 |
| GameInputManager.cs | 入队成功时调用 `AddQueuePreview`，执行完成时调用 `RemoveQueuePreview` |

---

## 六、修改文件总览

| 文件 | 涉及问题 | 修改范围 |
|------|---------|---------|
| CropController.cs | P1/P5 | Awake 创建视觉子物体，移除 RequireComponent |
| GameInputManager.cs | P2/P3/P4/P6 | HandleUseCurrentTool、ExecuteFarmAction、Update（延迟执行）、入队/出队预览调用 |
| PlayerInteraction.cs | P3 | OnActionComplete 松开分支时序调整 |
| FarmToolPreview.cs | P6 | 颜色叠加、图案预览、多位置队列预览 |

---

## 七、风险评估

| 风险 | 等级 | 缓解措施 |
|------|------|---------|
| CropController 子物体迁移影响存档加载 | 中 | Awake 中自动检测并迁移，兼容新旧结构 |
| 动画帧触发时机不精确 | 低 | 使用时间比例计算，`tileUpdateTriggerProgress` 可在 Inspector 调整 |
| 多位置预览性能 | 低 | 队列最大长度有限（通常不超过10个），GhostTilemap 操作轻量 |
| 长按防重复入队逻辑 | 中 | 使用 `_queuedPositions` HashSet 已有去重机制 |
