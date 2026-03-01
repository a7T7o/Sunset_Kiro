# 补丁003 设计文档 V2

> V2 重组原则：学习补丁002 design.md，按「修改目标（文件 × 方法/区域）」为主轴。每个版块自包含——现状代码摘要、所有需要的改动、改动后完整伪代码、该版块保证的正确性属性。执行时只需看对应版块。
>
> 来源：`补丁003全面分析与修复方案V2.md`（6 个问题：P1/P5/P2/P3/P4/P6 + 7 个漏洞修补）
> 所有方案已锁定：V2/V3 审视报告全面确认，无待确认项。

---

## 目录

1. [模块 A：作物 Prefab 结构改造（P1/P5）](#一模块-a作物-prefab-结构改造p1p5)
2. [模块 B：CropController — 接口暴露与 transform 引用审计（V5/V7）](#二模块-bcropcontroller--接口暴露与-transform-引用审计v5v7)
3. [模块 C：GameInputManager.ExecuteFarmAction — 延迟执行机制（P3）](#三模块-cgameinputmanagerexecutefarmaction--延迟执行机制p3)
4. [模块 D：GameInputManager.ClearActionQueue — 清理完整性（V1）](#四模块-dgameinputmanagerclearactionqueue--清理完整性v1)
5. [模块 E：PlayerInteraction.OnActionComplete — 时序修复与长按分支（P3/P2/V2）](#五模块-eplayerinteractiononactioncomplete--时序修复与长按分支p3p2v2)
6. [模块 F：GameInputManager.HandleUseCurrentTool — 导航入队统一（P4/V6）](#六模块-fgameinputmanagerhandleuseCurrenttool--导航入队统一p4v6)
7. [模块 G：FarmVisualManager — 水渍 tile 接口暴露（P6 前置）](#七模块-gfarmvisualmanager--水渍-tile-接口暴露p6-前置)
8. [模块 H：FarmToolPreview — 预览系统全面改造（P6）](#八模块-hfarmtoolpreview--预览系统全面改造p6)
9. [模块 I：GameInputManager — 队列预览联动（P6）](#九模块-igameinputmanager--队列预览联动p6)
10. [交互矩阵](#十交互矩阵)
11. [正确性属性汇总](#十一正确性属性汇总)
12. [涉及文件汇总](#十二涉及文件汇总)

---

## 一、模块 A：作物 Prefab 结构改造（P1/P5）

> 涉及问题：P1（作物位置错误）、P5（种子位置错误）
> 修改方式：手动修改 Prefab + Editor 批量工具（非代码运行时改造）

### 1.1 现状

CropController Prefab 结构：
```
Crop_xxx (自身) ← SpriteRenderer + CropController + BoxCollider2D
```
- `Awake` 第175行：`spriteRenderer = GetComponent<SpriteRenderer>()` — 获取自身 SR
- `AlignSpriteBottom` 第723行：修改 `spriteRenderer.transform.localPosition` = 修改自身 localPosition = 覆盖世界坐标

### 1.2 改动

手动修改所有作物 Prefab 为父子结构：
```
Crop_xxx_Root (父物体，空 Transform) ← 位置 = 格子中心
└─ Crop_xxx (子物体) ← SpriteRenderer + CropController + BoxCollider2D（原有 Prefab 全部内容）
```

需要 Editor 批量工具辅助：
- 遍历所有作物 Prefab
- 为每个 Prefab 创建父物体
- 将原有内容移到子物体下
- 保存 Prefab

### 1.3 改动后效果

- `Instantiate(cropPrefab, cropWorldPos, ...)` → 父物体在格子中心
- `AlignSpriteBottom` 修改子物体 localPosition → 父物体世界坐标不受影响
- `GetComponent<SpriteRenderer>()` 仍然能找到（SR 在 CropController 同一 GameObject）
- `[RequireComponent(typeof(SpriteRenderer))]` 保留

### 1.4 CropController 代码无需修改

AlignSpriteBottom 现状代码：
```csharp
private void AlignSpriteBottom()
{
    if (spriteRenderer == null || spriteRenderer.sprite == null) return;
    Bounds spriteBounds = spriteRenderer.sprite.bounds;
    Vector3 localPos = spriteRenderer.transform.localPosition;
    localPos.y = -spriteBounds.min.y;
    spriteRenderer.transform.localPosition = localPos;
}
```
改造后 `spriteRenderer.transform` 是子物体，`localPosition` 相对于父物体，不影响父物体世界坐标。代码无需修改。

### 1.5 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-A1 | 作物 GameObject 世界坐标 = 格子中心 = 预览中心 | 父物体位置由 Instantiate 设置，AlignSpriteBottom 只影响子物体 |
| CP-A2 | 每个生长阶段 Sprite 底部中心对齐格子中心 | AlignSpriteBottom 在 UpdateVisuals 末尾调用 |

---

## 二、模块 B：CropController — 接口暴露与 transform 引用审计（V5/V7）

> 涉及漏洞：V5（stages 是 private）、V7（外部代码依赖 transform.position）
> 文件：`Assets/YYY_Scripts/Farm/CropController.cs`

### 2.1 现状

```csharp
[SerializeField] private CropStageConfig[] stages;  // 第41行，private
```
外部无法获取第一阶段 Sprite，种子预览需要此数据。

### 2.2 改动1：添加公开方法

```csharp
/// <summary>
/// 获取第一阶段的普通 Sprite（种子预览用）
/// </summary>
public Sprite GetFirstStageSprite()
{
    if (stages == null || stages.Length == 0) return null;
    return stages[0].normalSprite;
}
```

### 2.3 改动2：transform 引用审计

套父物体后，CropController 的 `transform` 是子物体（经过 AlignSpriteBottom 偏移）。如果外部代码通过 `cropController.transform.position` 获取作物位置，得到的是偏移后的位置而非格子中心。

需要全局搜索以下模式：
- `cropController.transform.position`
- `crop.transform.position`
- `GetComponent<CropController>().transform`

如果发现外部引用，需要改为 `cropController.transform.parent.position`（父物体 = 格子中心），或在 CropController 中添加：
```csharp
/// <summary>
/// 获取作物的格子中心世界坐标（父物体位置）
/// </summary>
public Vector3 GetCellCenterPosition()
{
    return transform.parent != null ? transform.parent.position : transform.position;
}
```

### 2.4 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-B1 | 外部代码获取作物位置时得到格子中心而非偏移后位置 | GetCellCenterPosition 方法 + 全局搜索替换 |
| CP-B2 | 种子预览能获取第一阶段 Sprite | GetFirstStageSprite 公开方法 |

---

## 三、模块 C：GameInputManager.ExecuteFarmAction — 延迟执行机制（P3）

> 涉及问题：P3（tile 更新与动画同帧执行）
> 文件：`Assets/YYY_Scripts/Controller/Input/GameInputManager.cs`

### 3.1 现状代码

```csharp
// ExecuteFarmAction 第2495行
case FarmActionType.Till:
    FaceTarget(request.worldPos);
    playerInteraction?.RequestAction(AnimState.Crush);
    ExecuteTillSoil(request.layerIndex, request.cellPos);  // ← 同帧立即执行！
    break;

case FarmActionType.Water:
    FaceTarget(request.worldPos);
    playerInteraction?.RequestAction(AnimState.Watering);
    ExecuteWaterTile(request.layerIndex, request.cellPos);  // ← 同帧立即执行！
    break;
```

### 3.2 改动：引入 `_pendingTileUpdate`

新增字段：
```csharp
// 延迟 tile 更新
private FarmActionRequest? _pendingTileUpdate = null;
private bool _tileUpdateTriggered = false;

[Header("动画帧触发")]
[SerializeField] private float tileUpdateTriggerProgress = 0.5f;  // 动画50%进度触发
```

### 3.3 改动后 ExecuteFarmAction 伪代码

```
case Till:
    FaceTarget(request.worldPos)
    playerInteraction?.RequestAction(AnimState.Crush)
    // 🔴 不再同步执行 ExecuteTillSoil
    _pendingTileUpdate = request
    _tileUpdateTriggered = false
    break

case Water:
    FaceTarget(request.worldPos)
    playerInteraction?.RequestAction(AnimState.Watering)
    // 🔴 不再同步执行 ExecuteWaterTile
    _pendingTileUpdate = request
    _tileUpdateTriggered = false
    break
```

### 3.4 Update 中新增延迟执行监听

```
// 在 Update() 中，HandleUseCurrentTool 之前或之后
if (_pendingTileUpdate != null && !_tileUpdateTriggered)
{
    float progress = playerInteraction?.GetAnimationProgress() ?? 1f
    if (progress >= tileUpdateTriggerProgress)
    {
        var req = _pendingTileUpdate.Value
        switch (req.type)
        {
            case Till: ExecuteTillSoil(req.layerIndex, req.cellPos); break
            case Water: ExecuteWaterTile(req.layerIndex, req.cellPos); break
        }
        _tileUpdateTriggered = true
        // 注意：不清空 _pendingTileUpdate，等动画完成回调时清空
    }
}
```

### 3.5 动画完成回调中清空

OnFarmActionAnimationComplete 中：
```csharp
public void OnFarmActionAnimationComplete()
{
    // 如果 tile 更新还没触发（异常情况，如动画被跳过），强制执行
    if (_pendingTileUpdate != null && !_tileUpdateTriggered)
    {
        var req = _pendingTileUpdate.Value;
        switch (req.type)
        {
            case FarmActionType.Till: ExecuteTillSoil(req.layerIndex, req.cellPos); break;
            case FarmActionType.Water: ExecuteWaterTile(req.layerIndex, req.cellPos); break;
        }
    }
    _pendingTileUpdate = null;
    _tileUpdateTriggered = false;
    
    _isExecutingFarming = false;
    _queuedPositions.Remove((_currentProcessingRequest.layerIndex, _currentProcessingRequest.cellPos));
    ProcessNextAction();
}
```

### 3.6 GetAnimationProgress 确认

PlayerInteraction 或 PlayerAnimController 需要暴露动画进度查询方法。当前代码中 `toolAnimationDuration = 0.8f` 和时间计时器已存在，需要确认 `GetAnimationProgress()` 是否已有或需要新增。

### 3.7 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-C1 | tile 更新在动画进度 >= 50% 时触发，不在动画开始瞬间 | Update 中监听进度 + `_pendingTileUpdate` 机制 |
| CP-C2 | 动画完成时 tile 更新一定已执行（兜底） | OnFarmActionAnimationComplete 中强制执行未触发的更新 |
| CP-C3 | 连续操作 A→B 流程正确：A 第四帧触发 → A 完成 → 改朝向 B → B 动画 → B 第四帧触发 | 每次 ExecuteFarmAction 重置 `_pendingTileUpdate` |

---

## 四、模块 D：GameInputManager.ClearActionQueue — 清理完整性（V1）

> 涉及漏洞：V1（ClearActionQueue 未清理 _pendingTileUpdate）
> 文件：`Assets/YYY_Scripts/Controller/Input/GameInputManager.cs`

### 4.1 现状代码

```csharp
public void ClearActionQueue()
{
    _farmActionQueue.Clear();
    _queuedPositions.Clear();
    _isProcessingQueue = false;
    _isExecutingFarming = false;
    _currentHarvestTarget = null;
    _currentProcessingRequest = default;
}
```
缺少 `_pendingTileUpdate` 清理。

### 4.2 改动后完整伪代码

```csharp
public void ClearActionQueue()
{
    _farmActionQueue.Clear();
    _queuedPositions.Clear();
    _isProcessingQueue = false;
    _isExecutingFarming = false;
    _currentHarvestTarget = null;
    _currentProcessingRequest = default;
    
    // 🔴 V1 漏洞修补：清理待执行的 tile 更新
    _pendingTileUpdate = null;
    _tileUpdateTriggered = false;
    
    // 🔴 P6：清理所有队列预览
    FarmToolPreview.Instance?.ClearAllQueuePreviews();
}
```

### 4.3 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-D1 | ClearActionQueue 后无残留状态：队列空、防重复集合空、_pendingTileUpdate 空、队列预览清空 | ClearActionQueue 方法完整清理 |

---

## 五、模块 E：PlayerInteraction.OnActionComplete — 时序修复与长按分支（P3/P2/V2）

> 涉及问题：P3（松开分支时序）、P2（长按连续执行）、V2（清理顺序不一致）
> 文件：`Assets/YYY_Scripts/Service/Player/PlayerInteraction.cs`

### 5.1 现状代码（松开分支，第247行起）

```csharp
else  // 松开分支
{
    var gimRelease = GameInputManager.Instance;
    if (gimRelease != null)
    {
        if (lockManager != null && lockManager.HasCachedHotbarInput)
            gimRelease.ClearActionQueue();
        else
            gimRelease.OnFarmActionAnimationComplete();  // ← 触发 ProcessNextAction
    }
    layerAnimSync?.ForceHideTool();
    animController?.StopAnimationTracking();
    isPerformingAction = false;  // ← 太晚！在 OnFarmActionAnimationComplete 之后
    lockManager?.EndAction(false);
    ApplyCachedHotbarSwitch();
    lockManager?.ClearAllCache();
}
```

### 5.2 改动1：松开分支时序修复

`isPerformingAction = false` 必须在 `OnFarmActionAnimationComplete()` 之前，否则 ProcessNextAction → ExecuteFarmAction → RequestAction → PerformAction 被 `isPerformingAction` 守卫拦截。

### 5.3 改动2：长按分支增加重新入队逻辑（P2）

长按分支现状（第224行起）：
```csharp
if (isFarmTool)
{
    animController?.StopAnimationTracking();
    lockManager?.EndAction(false);
    lockManager?.ClearAllCache();
    isPerformingAction = false;
    gimContinue.OnFarmActionAnimationComplete();
}
```
当前逻辑：长按分支直接调用 `OnFarmActionAnimationComplete` → `ProcessNextAction`。如果队列为空，ProcessNextAction 结束，长按连续执行就断了。

需要改为：长按分支中，如果队列为空且仍在长按，获取当前鼠标位置重新入队。

### 5.4 改动后完整伪代码

```
OnActionComplete():
  // ===== Collect 专用分支（保持不变）=====
  if (currentAction == Collect):
    animController?.StopAnimationTracking()
    lockManager?.EndAction(false)
    lockManager?.ClearAllCache()
    isPerformingAction = false
    GameInputManager.Instance?.OnCollectAnimationComplete()
    return

  if (currentAction == Death): isCarrying = false
  ApplyCachedDirectionToFacing()

  bool isCurrentlyHolding = Input.GetMouseButton(0)
  bool shouldContinue = isCurrentlyHolding && IsToolAction(currentAction)
  var actionToRepeat = currentAction

  if (shouldContinue):
    var gim = GameInputManager.Instance
    bool isFarmTool = gim != null && gim.IsHoldingFarmTool()

    if (isFarmTool):
      // 🔴 V2 漏洞修补：统一清理顺序（与松开分支一致）
      animController?.StopAnimationTracking()
      isPerformingAction = false          // 🔴 先解除守卫
      lockManager?.EndAction(false)
      lockManager?.ClearAllCache()
      // 🔴 P2：长按分支 — 队列为空时重新入队当前鼠标位置
      if (gim.IsQueueEmpty()):
        gim.TryEnqueueFromCurrentInput()  // 获取当前鼠标位置入队
      gim.OnFarmActionAnimationComplete() // 取队列下一个
    else:
      // 通用工具（镐子/斧头）：保持原有长按行为不变
      animController?.StopAnimationTracking()
      lockManager?.EndAction(true)
      StartAction(actionToRepeat, true)

  else:  // 松开分支
    var gimRelease = GameInputManager.Instance
    // 🔴 P3 时序修复：先解除守卫，再触发下一个
    layerAnimSync?.ForceHideTool()
    animController?.StopAnimationTracking()
    isPerformingAction = false            // 🔴 移到 OnFarmActionAnimationComplete 之前
    lockManager?.EndAction(false)

    if (gimRelease != null):
      if (lockManager != null && lockManager.HasCachedHotbarInput):
        gimRelease.ClearActionQueue()     // 动画期间切换工具栏 → 清空队列
      else:
        gimRelease.OnFarmActionAnimationComplete()  // 取队列下一个

    ApplyCachedHotbarSwitch()
    lockManager?.ClearAllCache()
```

### 5.5 GameInputManager 需要新增的辅助方法

```csharp
/// <summary>
/// 队列是否为空（供 OnActionComplete 长按分支查询）
/// </summary>
public bool IsQueueEmpty() => _farmActionQueue.Count == 0;
```

### 5.6 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-E1 | 松开分支：isPerformingAction = false 在 OnFarmActionAnimationComplete 之前 | 时序调整 |
| CP-E2 | 长按分支：队列为空且仍在长按时，获取当前鼠标位置重新入队 | IsQueueEmpty + TryEnqueueFromCurrentInput |
| CP-E3 | 长按/松开分支清理顺序一致 | 统一为 StopTracking → isPerformingAction=false → EndAction → ClearAllCache |
| CP-E4 | Collect 专用分支不进入 IsToolAction 长按逻辑 | currentAction == Collect 在 shouldContinue 之前拦截 |
| CP-E5 | 通用工具（Slice/Pierce）保持原有长按行为不变 | else 分支完全不变 |

---

## 六、模块 F：GameInputManager.HandleUseCurrentTool — 导航入队统一（P4/V6）

> 涉及问题：P4（导航期间预览变更）、V6（任务4方向错误）
> 文件：`Assets/YYY_Scripts/Controller/Input/GameInputManager.cs`

### 6.1 现状代码（导航状态分支，第710行起）

```csharp
if (_farmNavState == FarmNavState.Navigating || _farmNavState == FarmNavState.Locked)
{
    if (_isProcessingQueue)
    {
        TryEnqueueFromCurrentInput();
        return;
    }
    // 读取实时数据判断新位置有效性
    var farmPreview = FarmToolPreview.Instance;
    if (farmPreview != null && farmPreview.IsValid())
    {
        if (farmPreview.IsLocked && farmPreview.CurrentCellPos == farmPreview.LockedCellPos)
            return;  // 点击同一位置
        CancelFarmingNavigation();
        // 继续往下走，重新进入工具入队逻辑
    }
    else
    {
        CancelFarmingNavigation();
        return;
    }
}
```

问题：队列未处理时（`_isProcessingQueue = false`），导航中的点击会中断导航重新开始。用户要求所有点击统一入队。

### 6.2 改动后伪代码

```
// 导航中/执行中的左键点击 → 统一入队
if (_farmNavState == FarmNavState.Navigating || _farmNavState == FarmNavState.Locked
    || _farmNavState == FarmNavState.Executing)
{
    TryEnqueueFromCurrentInput();
    return;
}
```

简化为一行：无论导航状态如何，所有左键点击都走入队。导航期间的点击不再中断导航。

### 6.3 V6 漏洞修补：不改入口检测

HandleUseCurrentTool 的入口检测保持 `GetMouseButtonDown(0)`，不改为 `GetMouseButton(0)`。长按连续执行由模块 E（OnActionComplete 长按分支）处理。

### 6.4 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-F1 | 导航期间的左键点击统一入队，不中断当前导航 | 导航状态分支简化为 TryEnqueueFromCurrentInput |
| CP-F2 | HandleUseCurrentTool 入口保持 GetMouseButtonDown | 不修改入口检测 |

---

## 七、模块 G：FarmVisualManager — 水渍 tile 接口暴露（P6 前置）

> 涉及问题：P6（浇水预览需要水渍 tile 资源）
> 文件：`Assets/YYY_Scripts/Farm/FarmVisualManager.cs`

### 7.1 现状

```csharp
[SerializeField] private TileBase[] wetPuddleTiles;  // 3种水渍变体，private
```
外部无法访问水渍 tile 资源。

### 7.2 改动：添加公开方法

```csharp
/// <summary>
/// 获取随机水渍 Tile（浇水预览用）
/// </summary>
public TileBase GetRandomPuddleTile()
{
    if (wetPuddleTiles == null || wetPuddleTiles.Length == 0) return null;
    return wetPuddleTiles[Random.Range(0, wetPuddleTiles.Length)];
}

/// <summary>
/// 获取所有水渍 Tile 变体
/// </summary>
public TileBase[] GetPuddleTiles() => wetPuddleTiles;
```

### 7.3 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-G1 | 浇水预览能获取随机水渍 tile | GetRandomPuddleTile 公开方法 |

---

## 八、模块 H：FarmToolPreview — 预览系统全面改造（P6）

> 涉及问题：P6（预览系统全面改造）、V3（颜色乘法混合）、V4（ClearGhostTilemap 冲突）
> 文件：`Assets/YYY_Scripts/Farm/FarmToolPreview.cs`
> 这是改动量最大的模块，涉及预览架构重构

### 8.1 现状架构

```
FarmToolPreview
├─ ghostTilemap（GhostTilemap）— 耕地 1+8 预览 tile
├─ cursorRenderer（SpriteRenderer）— 方框光标
├─ currentPreviewPositions（HashSet）— 当前预览位置缓存
└─ 锁定机制（_isLocked / _lockedCellPos / _lockedLayerIndex）
```

问题：
- cursorRenderer 方框光标用于所有模式，用户要求耕地/浇水取消方框
- `Tilemap.SetColor` 是乘法混合（V3），不能实现颜色覆盖
- `ClearGhostTilemap` 清除所有预览位置，包括队列预览（V4）
- 无多位置队列预览支持

### 8.2 改造后架构

```
FarmToolPreview
├─ ghostTilemap（现有）— 鼠标跟随预览专用
│   └─ 耕地 1+8 预览 tile（保持不变）
│
├─ queuePreviewTilemap（新增）— 队列锁定预览专用
│   └─ 耕地/浇水队列预览 tile（原始 tile + 透明度）
│
├─ 鼠标跟随颜色覆盖层（新增，方案C）
│   ├─ hoeOverlayRenderer（SpriteRenderer）— 耕地颜色覆盖
│   └─ waterOverlayRenderer（SpriteRenderer）— 浇水颜色覆盖
│
├─ 种子预览（新增，复刻放置系统）
│   ├─ seedGridCell（程序化格子方框）— 底部格子
│   └─ seedPreviewRenderer（SpriteRenderer）— 作物第一阶段 sprite
│
├─ 种子队列预览对象池（新增）
│   └─ List<SpriteRenderer> seedQueuePool — 对象池
│
├─ cursorRenderer（现有）— 仅种子模式保留
│
└─ 锁定机制（保持不变）
```

### 8.3 新增字段

```csharp
// === 双 Tilemap 分离 ===
private Tilemap queuePreviewTilemap;           // 队列预览专用 Tilemap
private TilemapRenderer queuePreviewTilemapRenderer;

// === 方案C：程序化 SpriteRenderer 颜色覆盖 ===
private SpriteRenderer hoeOverlayRenderer;     // 耕地颜色覆盖
private SpriteRenderer waterOverlayRenderer;   // 浇水颜色覆盖
private Sprite overlaySprite;                  // 程序化纯色方块 sprite（共用）

// === 种子预览（复刻放置系统）===
private SpriteRenderer seedPreviewRenderer;    // 作物第一阶段 sprite
private SpriteRenderer seedGridRenderer;       // 底部格子方框
private Sprite gridSprite;                     // 程序化格子方框 sprite

// === 种子队列预览对象池 ===
private List<SpriteRenderer> seedQueuePool = new();
private List<(Vector3Int cellPos, SpriteRenderer renderer)> activeSeedQueuePreviews = new();

// === 队列预览位置缓存 ===
private HashSet<Vector3Int> queuePreviewPositions = new();

// === 颜色配置 ===
[Header("覆盖层颜色")]
[SerializeField] private Color overlayValidColor = new Color(0f, 1f, 0f, 0.3f);
[SerializeField] private Color overlayInvalidColor = new Color(1f, 0f, 0f, 0.3f);
[SerializeField] private float queuePreviewAlpha = 0.5f;
```

### 8.4 初始化（EnsureComponents 扩展）

```
EnsureComponents():
  // ... 现有 ghostTilemap 和 cursorRenderer 创建逻辑 ...

  // 新增：queuePreviewTilemap
  if (queuePreviewTilemap == null):
    var queueGo = new GameObject("QueuePreviewTilemap")
    queueGo.transform.SetParent(transform, false)
    queuePreviewTilemap = queueGo.AddComponent<Tilemap>()
    queuePreviewTilemapRenderer = queueGo.AddComponent<TilemapRenderer>()
    // 设置 Sorting Layer 与 ghostTilemap 一致

  // 新增：颜色覆盖 SpriteRenderer
  overlaySprite = CreateOverlaySprite()  // 程序化纯色方块
  hoeOverlayRenderer = CreateOverlayRenderer("HoeOverlay")
  waterOverlayRenderer = CreateOverlayRenderer("WaterOverlay")

  // 新增：种子预览
  gridSprite = CreateGridSprite()  // 复刻 PlacementGridCell 的程序化格子
  seedGridRenderer = CreateSeedGridRenderer("SeedGrid")
  seedPreviewRenderer = CreateSeedPreviewRenderer("SeedPreview")
```

### 8.5 程序化 Sprite 生成

覆盖层 Sprite（纯色半透明方块，无边框）：
```
CreateOverlaySprite():
  // 生成 32x32 纯白色填充 Texture2D（无边框）
  // 运行时通过 SpriteRenderer.color 控制颜色和透明度
  var tex = new Texture2D(32, 32)
  填充所有像素为 Color.white
  return Sprite.Create(tex, new Rect(0,0,32,32), new Vector2(0.5f,0.5f), 32)
```

格子方框 Sprite（复刻 PlacementGridCell）：
```
CreateGridSprite():
  // 复刻 PlacementGridCell.CreateGridSprite 的逻辑
  // 32x32，边框 alpha=0.8，内部填充 alpha=0.3
  // 运行时通过 SpriteRenderer.color 控制颜色
```

### 8.6 耕地预览改造（UpdateHoePreview）

```
UpdateHoePreview(layerIndex, cellPos, playerTransform, reach):
  if (_isLocked) return  // 锁定时不更新

  UpdateRealtimeData(...)
  bool isValid = ...  // 现有有效性判断

  // 🔴 去除 cursorRenderer 方框（耕地不需要方框）
  cursorRenderer.enabled = false

  // 🔴 方案C：颜色覆盖
  hoeOverlayRenderer.enabled = true
  hoeOverlayRenderer.transform.position = GetCellCenterWorld(layerIndex, cellPos)
  hoeOverlayRenderer.color = isValid ? overlayValidColor : overlayInvalidColor

  // GhostTilemap 1+8 预览保持不变（只在有效时显示）
  ClearGhostTilemap()
  if (isValid && isHoeMode):
    // ... 现有 GetPreviewTiles 逻辑 ...
```

### 8.7 浇水预览改造（UpdateWateringPreview）

```
UpdateWateringPreview(layerIndex, cellPos, playerTransform, reach):
  if (_isLocked) return

  UpdateRealtimeData(...)
  bool isValid = ...

  // 🔴 去除 cursorRenderer 方框
  cursorRenderer.enabled = false

  // 🔴 方案C：颜色覆盖
  waterOverlayRenderer.enabled = true
  waterOverlayRenderer.transform.position = GetCellCenterWorld(layerIndex, cellPos)
  waterOverlayRenderer.color = isValid ? overlayValidColor : overlayInvalidColor

  // 🔴 水渍 tile 预览（在 ghostTilemap 上显示）
  ClearGhostTilemap()
  if (isValid):
    var puddleTile = FarmVisualManager.Instance?.GetRandomPuddleTile()
    if (puddleTile != null):
      ghostTilemap.SetTile(cellPos, puddleTile)
      currentPreviewPositions.Add(cellPos)
```

### 8.8 种子预览改造（UpdateSeedPreview）

```
UpdateSeedPreview(alignedPos, seedData, playerTransform, reach):
  if (_isLocked) return

  // 🔴 隐藏耕地/浇水覆盖层
  hoeOverlayRenderer.enabled = false
  waterOverlayRenderer.enabled = false

  bool isValid = ...  // isFarmland && !hasCrop

  // 🔴 底部格子方框（复刻 PlacementGridCell）
  seedGridRenderer.enabled = true
  seedGridRenderer.transform.position = GetCellCenterWorld(...)
  seedGridRenderer.color = isValid
    ? new Color(0f, 1f, 0f, 0.4f)   // 绿色
    : new Color(1f, 0f, 0f, 0.4f)   // 红色

  // 🔴 作物第一阶段 sprite
  seedPreviewRenderer.enabled = true
  var cropSprite = seedData.cropPrefab?.GetComponent<CropController>()?.GetFirstStageSprite()
  seedPreviewRenderer.sprite = cropSprite
  seedPreviewRenderer.transform.position = GetCellCenterWorld(...)
  seedPreviewRenderer.color = isValid
    ? new Color(1f, 1f, 1f, 0.7f)              // 原色 + alpha
    : new Color(1f, 0.5f, 0.5f, 0.7f)          // 偏红 + alpha

  // 🔴 保留 cursorRenderer（种子模式保留光标框）— 实际上底部格子已替代，cursorRenderer 可隐藏
  cursorRenderer.enabled = false  // 底部格子方框已替代
```

### 8.9 队列预览管理

```
// 入队时调用：在指定位置显示队列锁定预览
AddQueuePreview(cellPos, layerIndex, FarmActionType type):
  if (queuePreviewPositions.Contains(cellPos)) return  // 防重复

  if (type == PlantSeed):
    // 种子队列预览：SpriteRenderer 对象池
    var renderer = GetOrCreateSeedQueueRenderer()
    renderer.transform.position = GetCellCenterWorld(layerIndex, cellPos)
    renderer.sprite = 当前种子的第一阶段 sprite
    renderer.color = new Color(1f, 1f, 1f, queuePreviewAlpha)  // 原色 + 降低 alpha
    renderer.enabled = true
    activeSeedQueuePreviews.Add((cellPos, renderer))
  else:
    // 耕地/浇水队列预览：queuePreviewTilemap
    var tile = 获取对应预览 tile（耕地用 farmland tile，浇水用 puddle tile）
    queuePreviewTilemap.SetTile(cellPos, tile)
    // 设置透明度
    queuePreviewTilemap.SetColor(cellPos, new Color(1f, 1f, 1f, queuePreviewAlpha))

  queuePreviewPositions.Add(cellPos)

// 执行完成时调用：移除指定位置的队列预览
RemoveQueuePreview(cellPos):
  if (!queuePreviewPositions.Contains(cellPos)) return

  // 检查是否是种子队列预览
  var seedEntry = activeSeedQueuePreviews.Find(x => x.cellPos == cellPos)
  if (seedEntry.renderer != null):
    seedEntry.renderer.enabled = false
    seedQueuePool.Add(seedEntry.renderer)  // 回收到对象池
    activeSeedQueuePreviews.Remove(seedEntry)
  else:
    queuePreviewTilemap.SetTile(cellPos, null)

  queuePreviewPositions.Remove(cellPos)

// 清空所有队列预览（WASD 中断 / 切换工具 / ESC）
ClearAllQueuePreviews():
  // 清空 queuePreviewTilemap
  foreach (var pos in queuePreviewPositions):
    queuePreviewTilemap.SetTile(pos, null)

  // 回收所有种子队列预览
  foreach (var (cellPos, renderer) in activeSeedQueuePreviews):
    renderer.enabled = false
    seedQueuePool.Add(renderer)
  activeSeedQueuePreviews.Clear()

  queuePreviewPositions.Clear()
```

### 8.10 Hide 方法扩展

```
Hide():
  // 现有逻辑
  cursorRenderer.enabled = false
  ClearGhostTilemap()

  // 新增：隐藏所有覆盖层和种子预览
  hoeOverlayRenderer.enabled = false
  waterOverlayRenderer.enabled = false
  seedGridRenderer.enabled = false
  seedPreviewRenderer.enabled = false
  // 注意：不清空队列预览（Hide 只隐藏鼠标跟随预览）
```

### 8.11 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-H1 | 耕地预览显示方案C颜色覆盖，不显示方框光标 | UpdateHoePreview 中 cursorRenderer.enabled = false + hoeOverlayRenderer |
| CP-H2 | 浇水预览显示水渍 tile + 方案C颜色覆盖 | UpdateWateringPreview 中 ghostTilemap + waterOverlayRenderer |
| CP-H3 | 种子预览复刻放置系统效果（底部格子 + 物品 sprite + 颜色切换） | UpdateSeedPreview 中 seedGridRenderer + seedPreviewRenderer |
| CP-H4 | ClearGhostTilemap 只清除 ghostTilemap，不影响 queuePreviewTilemap | 双 Tilemap 分离 |
| CP-H5 | 队列预览在入队时添加，执行完移除，中断时全部清空 | AddQueuePreview / RemoveQueuePreview / ClearAllQueuePreviews |
| CP-H6 | 种子队列预览使用 SpriteRenderer 对象池，不使用 Tilemap | activeSeedQueuePreviews + seedQueuePool |

---

## 九、模块 I：GameInputManager — 队列预览联动（P6）

> 涉及问题：P6（入队/出队时调用预览管理）
> 文件：`Assets/YYY_Scripts/Controller/Input/GameInputManager.cs`

### 9.1 EnqueueAction 中添加队列预览

```
EnqueueAction(FarmActionRequest request):
  // ... 现有防重复和入队逻辑 ...
  _farmActionQueue.Enqueue(request)

  // 🔴 P6：入队成功时添加队列预览
  FarmToolPreview.Instance?.AddQueuePreview(request.cellPos, request.layerIndex, request.type)

  // ... 现有启动处理逻辑 ...
```

### 9.2 OnFarmActionAnimationComplete 中移除队列预览

```
OnFarmActionAnimationComplete():
  // ... 现有逻辑 ...
  _queuedPositions.Remove(...)

  // 🔴 P6：执行完成时移除队列预览
  FarmToolPreview.Instance?.RemoveQueuePreview(_currentProcessingRequest.cellPos)

  ProcessNextAction()
```

### 9.3 OnCollectAnimationComplete 中移除队列预览

```
OnCollectAnimationComplete():
  // ... 现有逻辑 ...
  _queuedPositions.Remove(...)

  // 🔴 P6：执行完成时移除队列预览
  FarmToolPreview.Instance?.RemoveQueuePreview(_currentProcessingRequest.cellPos)

  ProcessNextAction()
```

### 9.4 ClearActionQueue 中已包含清空（模块 D 已处理）

模块 D 的 ClearActionQueue 改动已包含 `FarmToolPreview.Instance?.ClearAllQueuePreviews()`。

### 9.5 正确性属性

| 属性 | 描述 | 保证方式 |
|------|------|---------|
| CP-I1 | 入队成功时对应位置显示队列预览 | EnqueueAction 中调用 AddQueuePreview |
| CP-I2 | 执行完成时对应位置队列预览消失 | OnFarmActionAnimationComplete / OnCollectAnimationComplete 中调用 RemoveQueuePreview |
| CP-I3 | 队列清空时所有队列预览消失 | ClearActionQueue 中调用 ClearAllQueuePreviews |


---

## 十、交互矩阵

> 本章汇总补丁003涉及的所有交互场景，确保每个场景都有明确的行为定义和关联属性。
> 补丁003不改变补丁002已建立的队列基础架构（FIFO、防重复、暂停/恢复），只修复其中的时序错误、补充缺失功能。

### 10.1 左键点击入队矩阵（补丁003修改点标注）

| 编号 | 手持物品 | 鼠标位置 | 预览状态 | 行为 | 补丁003变更 | 关联属性 |
|------|---------|---------|---------|------|-----------|---------|
| S1 | 任意 | 成熟/枯萎作物（同层级） | - | 入队 Harvest | 无变更 | - |
| S2 | 锄头 | 可耕地 | Valid | 入队 Till | 无变更 | CP-F1 |
| S3 | 锄头 | 不可耕地 | Invalid | 无操作 | 无变更 | - |
| S4 | 浇水壶 | 可浇水地 | Valid | 入队 Water | 无变更 | CP-F1 |
| S5 | 浇水壶 | 不可浇水地 | Invalid | 无操作 | 无变更 | - |
| S6 | 种子 | 可种植地 | Valid | 入队 PlantSeed | 无变更 | CP-F1 |
| S7 | 种子 | 不可种植地 | Invalid | 无操作 | 无变更 | - |
| S8 | 任意农田工具 | 导航中点击新位置 | Valid | 直接入队（不中断导航） | P4：统一走 TryEnqueueFromCurrentInput | CP-F1 |
| S9 | 镐子/斧头/武器 | 无成熟作物 | - | 走原有工具逻辑（不入队） | 无变更 | CP-E5 |

### 10.2 队列执行期间点击矩阵

| 编号 | 触发条件 | 新点击目标 | 行为 | 补丁003变更 | 关联属性 |
|------|---------|-----------|------|-----------|---------|
| E1 | 动画执行中 | 有效农田目标 | 入队 | 无变更 | - |
| E2 | 导航中 | 同一格子 | 忽略（防重复） | 无变更 | - |
| E3 | 导航中 | 不同有效格子 | 入队 | P4：统一入队路径 | CP-F1 |
| E4 | 面板打开 | 任意 | 不处理 | 无变更 | - |

### 10.3 动画执行时序矩阵（P3 核心改动）

| 阶段 | 时间点 | 行为 | 关联属性 |
|------|--------|------|---------|
| 动画开始 | 帧 0 | ExecuteFarmAction 设置 `_pendingTileUpdate`，不立即执行 tile 更新 | CP-C1 |
| 动画中期 | 进度 >= 50% | Update 检测到进度达标，执行 `_pendingTileUpdate` 中的 tile 更新 | CP-C1 |
| 动画完成 | 最后帧 | OnFarmActionAnimationComplete 兜底：若 `_pendingTileUpdate` 未执行则强制执行 | CP-C2 |
| 连续操作 | A完成→B开始 | 改朝向 B → 新动画 → 重置 `_pendingTileUpdate` | CP-C3 |

### 10.4 OnActionComplete 分支矩阵（P3/P2/V2 改动）

| currentAction | IsToolAction | 鼠标按住 | 队列状态 | 行为 | 补丁003变更 | 关联属性 |
|--------------|-------------|---------|---------|------|-----------|---------|
| Collect | false | - | - | 专用分支：执行 Harvest → ProcessNextAction | 无变更 | CP-E4 |
| Crush（锄地） | true | true | 队列空 | 获取当前鼠标位置重新入队 | P2：长按分支改为重新入队 | CP-E2 |
| Crush（锄地） | true | true | 队列非空 | OnFarmActionAnimationComplete → ProcessNextAction | 无变更 | - |
| Crush（锄地） | true | false | - | OnFarmActionAnimationComplete → ProcessNextAction | P3：isPerformingAction 前移 | CP-E1 |
| Watering | true | true | 队列空 | 获取当前鼠标位置重新入队 | P2：同锄地 | CP-E2 |
| Watering | true | false | - | OnFarmActionAnimationComplete → ProcessNextAction | P3：同锄地 | CP-E1 |
| Slice（斧头） | true | true/false | - | 保持原有长按行为 | 无变更 | CP-E5 |
| Pierce（镐子） | true | true/false | - | 保持原有长按行为 | 无变更 | CP-E5 |

### 10.5 中断矩阵

| 触发 | 队列行为 | 导航行为 | 预览行为 | _pendingTileUpdate | 队列预览 | 关联属性 |
|------|---------|---------|---------|-------------------|---------|---------|
| WASD 移动 | 清空 | 取消 | 恢复跟随 | 清空 | 全部清空 | CP-D1 |
| ESC | 清空 | 取消 | 恢复跟随 | 清空 | 全部清空 | CP-D1 |
| 切换快捷栏 | 清空 | 取消 | 切换模式 | 清空 | 全部清空 | CP-D1 |
| 打开面板 | 暂停（不清空） | 取消当前导航 | 保持锁定 | 保留 | 保留 | - |
| 关闭面板 | 恢复执行 | 重新开始 | 重新锁定 | 保留 | 保留 | - |

### 10.6 预览状态矩阵（P6 改动）

| 场景 | 耕地预览 | 浇水预览 | 种子预览 | 补丁003变更 | 关联属性 |
|------|---------|---------|---------|-----------|---------|
| 鼠标跟随（有效） | 方案C覆盖层（绿色）+ 1+8 ghostTilemap | 方案C覆盖层（绿色）+ 水渍 tile | 底部格子（绿色）+ 作物第一阶段 sprite | P6：全面改造 | CP-H1/H2/H3 |
| 鼠标跟随（无效） | 方案C覆盖层（红色） | 方案C覆盖层（红色） | 底部格子（红色）+ 作物 sprite 偏红 | P6：全面改造 | CP-H1/H2/H3 |
| 队列锁定预览 | queuePreviewTilemap 上的 tile | queuePreviewTilemap 上的水渍 tile | SpriteRenderer 对象池（降低 alpha） | P6：新增 | CP-H5/H6 |
| 锁定到当前目标 | 锁定位置覆盖层 + 1+8 | 锁定位置覆盖层 + 水渍 | 锁定位置格子 + sprite | 无变更（锁定机制不变） | - |



---

## 十一、正确性属性汇总

| 编号 | 属性 | 描述 | 保证模块 |
|------|------|------|---------|
| CP-A1 | 作物世界坐标 = 格子中心 | 父物体位置由 Instantiate 设置，AlignSpriteBottom 只影响子物体 | 模块 A |
| CP-A2 | 每个生长阶段 Sprite 底部中心对齐格子中心 | AlignSpriteBottom 在 UpdateVisuals 末尾调用 | 模块 A |
| CP-B1 | 外部代码获取作物位置时得到格子中心 | GetCellCenterPosition 方法 + 全局搜索替换 | 模块 B |
| CP-B2 | 种子预览能获取第一阶段 Sprite | GetFirstStageSprite 公开方法 | 模块 B |
| CP-C1 | tile 更新在动画进度 >= 50% 时触发 | Update 中监听进度 + `_pendingTileUpdate` 机制 | 模块 C |
| CP-C2 | 动画完成时 tile 更新一定已执行（兜底） | OnFarmActionAnimationComplete 中强制执行 | 模块 C |
| CP-C3 | 连续操作 A→B 流程正确 | 每次 ExecuteFarmAction 重置 `_pendingTileUpdate` | 模块 C |
| CP-D1 | ClearActionQueue 后无残留状态 | 队列空 + 防重复集合空 + _pendingTileUpdate 空 + 队列预览清空 | 模块 D |
| CP-E1 | 松开分支：isPerformingAction = false 在回调之前 | 时序调整 | 模块 E |
| CP-E2 | 长按分支：队列空且仍在长按时重新入队 | IsQueueEmpty + TryEnqueueFromCurrentInput | 模块 E |
| CP-E3 | 长按/松开分支清理顺序一致 | 统一为 StopTracking → isPerformingAction=false → EndAction → ClearAllCache | 模块 E |
| CP-E4 | Collect 专用分支不进入 IsToolAction 长按逻辑 | currentAction == Collect 在 shouldContinue 之前拦截 | 模块 E |
| CP-E5 | 通用工具（Slice/Pierce）保持原有长按行为不变 | else 分支完全不变 | 模块 E |
| CP-F1 | 导航期间左键点击统一入队 | 导航状态分支简化为 TryEnqueueFromCurrentInput | 模块 F |
| CP-F2 | HandleUseCurrentTool 入口保持 GetMouseButtonDown | 不修改入口检测 | 模块 F |
| CP-G1 | 浇水预览能获取随机水渍 tile | GetRandomPuddleTile 公开方法 | 模块 G |
| CP-H1 | 耕地预览显示方案C颜色覆盖，不显示方框光标 | UpdateHoePreview 中 cursorRenderer.enabled = false + hoeOverlayRenderer | 模块 H |
| CP-H2 | 浇水预览显示水渍 tile + 方案C颜色覆盖 | UpdateWateringPreview 中 ghostTilemap + waterOverlayRenderer | 模块 H |
| CP-H3 | 种子预览复刻放置系统效果 | UpdateSeedPreview 中 seedGridRenderer + seedPreviewRenderer | 模块 H |
| CP-H4 | ClearGhostTilemap 只清除 ghostTilemap | 双 Tilemap 分离 | 模块 H |
| CP-H5 | 队列预览入队添加、执行完移除、中断全清 | AddQueuePreview / RemoveQueuePreview / ClearAllQueuePreviews | 模块 H |
| CP-H6 | 种子队列预览使用 SpriteRenderer 对象池 | activeSeedQueuePreviews + seedQueuePool | 模块 H |
| CP-I1 | 入队成功时显示队列预览 | EnqueueAction 中调用 AddQueuePreview | 模块 I |
| CP-I2 | 执行完成时队列预览消失 | OnFarmActionAnimationComplete / OnCollectAnimationComplete 中调用 RemoveQueuePreview | 模块 I |
| CP-I3 | 队列清空时所有队列预览消失 | ClearActionQueue 中调用 ClearAllQueuePreviews | 模块 I |



---

## 十二、涉及文件汇总

| 文件 | 修改内容 | 关联模块 | 关联问题 |
|------|---------|---------|---------|
| 作物 Prefab（所有） | 手动套父物体，原有内容移到子物体 | A | P1/P5 |
| `CropController.cs` | 新增 GetFirstStageSprite()、GetCellCenterPosition()；全局搜索 transform.position 引用并替换 | B | V5/V7 |
| `GameInputManager.cs` | ExecuteFarmAction 延迟执行（_pendingTileUpdate + Update 监听）；ClearActionQueue 补充清理；HandleUseCurrentTool 导航入队统一；EnqueueAction/OnFarmActionAnimationComplete/OnCollectAnimationComplete 队列预览联动 | C/D/F/I | P3/V1/P4/V6/P6 |
| `PlayerInteraction.cs` | OnActionComplete 松开分支 isPerformingAction 前移；长按分支队列空时重新入队；清理顺序统一 | E | P3/P2/V2 |
| `FarmVisualManager.cs` | 新增 GetRandomPuddleTile()、GetPuddleTiles() 公开方法 | G | P6 前置 |
| `FarmToolPreview.cs` | 新增双 Tilemap（queuePreviewTilemap）；新增方案C覆盖层（hoeOverlayRenderer/waterOverlayRenderer）；新增种子预览组件（seedGridRenderer/seedPreviewRenderer）；新增 SpriteRenderer 对象池；改造 UpdateHoePreview/UpdateWateringPreview/UpdateSeedPreview；新增 AddQueuePreview/RemoveQueuePreview/ClearAllQueuePreviews | H | P6 |
| Editor 批量工具（新建） | 批量为作物 Prefab 套父物体的 Editor 脚本 | A | P1/P5 |

### 测试框架

本补丁为 Unity C# 项目，核心逻辑涉及 MonoBehaviour 和 Unity API（Physics2D、Tilemap 等），不适合纯单元测试。

验证策略：
1. 编译验证：`getDiagnostics` 检查所有修改文件，0 错误 0 警告
2. 逻辑审查：代码审查确认每个正确性属性（CP-A1~CP-I3）在实现中被满足
3. 运行时验证：用户在 Unity Editor 中按交互矩阵（第十章）逐项手动测试

