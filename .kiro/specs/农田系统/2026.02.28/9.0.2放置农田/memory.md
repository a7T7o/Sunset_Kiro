# 9.0.2 放置农田 - 子工作区记忆

## 模块概述

实现农田工具的预览系统，采用联邦制架构：
- `FarmToolPreview` 独立于 `PlacementPreview`
- 共享 `PlacementGridCalculator` 坐标计算
- 通过断言模式实现 1+8 动态预览

## 当前状态

- **完成度**: 80%（代码实现完成，待验收）
- **最后更新**: 2026-02-07
- **状态**: ✅ Phase 1-3 完成，Phase 4 待验收

---

## 核心设计决策

### 联邦制 vs 大一统

| 方案 | 说明 | 结论 |
|------|------|------|
| 大一统 | 复用 PlacementPreview | ❌ 不适合 1+8 模式 |
| 联邦制 | 独立 FarmToolPreview | ✅ 采纳 |

### 断言模式（核心技术）

```csharp
Func<Vector3Int, bool> previewPredicate = pos =>
{
    if (pos == centerPos) return true; // 假装这里已耕作
    var tileData = FarmTileManager.Instance?.GetTileData(layerIndex, pos);
    return tileData != null && tileData.isTilled;
};
```

---

## 涉及的代码文件

### 核心文件（已修改）
| 文件 | 操作 | 说明 |
|------|------|------|
| `FarmlandBorderManager.cs` | 重构 | 添加 GetPreviewTiles 接口、断言模式支持 |
| `GameInputManager.cs` | 修改 | 接入 FarmToolPreview、UpdatePreviews() |

### 新增文件（已创建）
| 文件 | 说明 |
|------|------|
| `FarmToolPreview.cs` | 农田工具预览组件（自生能力） |

### 相关文件（引用）
| 文件 | 关系 |
|------|------|
| `PlacementGridCalculator.cs` | 坐标计算（复用） |
| `PlacementPreview.cs` | 参考实现 |
| `FarmTileManager.cs` | 数据查询 |

---

## 会话记录

### 会话 1 - 2026-02-06（锐评审视）

**用户需求**: 审核锐评001，验证指控真实性

**完成任务**:
1. ✅ 验证锐评指控（部分成立、部分不成立）
2. ✅ 识别 1+8 模式与 NxM 模式的本质差异
3. ✅ 创建 `锐评001审视报告.md`

**核心结论**: 农田预览不应强行寄生在放置系统上

---

### 会话 2 - 2026-02-06（设计路线对比）

**用户需求**: 对比 Kiro 和 Gemini 的设计路线

**完成任务**:
1. ✅ 分析 Kiro 路线（防御性思维）
2. ✅ 分析 Gemini 路线（架构重构）
3. ✅ 创建 `Kiro与Gemini设计路线对比分析.md`

**核心结论**: Gemini 方向正确，采纳联邦制 + 断言模拟

---

### 会话 3 - 2026-02-06（生成三件套）

**用户需求**: 生成 requirements.md、design.md、tasks.md

**完成任务**:
1. ✅ 创建 `requirements.md` - 4 个用户故事
2. ✅ 创建 `design.md` - 架构设计、重构方案
3. ✅ 创建 `tasks.md` - 4 阶段 45 个任务
4. ✅ 创建 `memory.md` - 本文件

**新建文件**:
- `requirements.md`
- `design.md`
- `tasks.md`
- `memory.md`

---

## 遗留问题

- [x] 等待用户审核三件套
- [x] 执行 Phase 1: FarmlandBorderManager 重构
- [x] 执行 Phase 2: FarmToolPreview 实现
- [x] 执行 Phase 3: GameInputManager 接入
- [ ] 执行 Phase 4: 集成测试（待验收）
- [ ] 场景配置：添加 FarmToolPreview 组件
- [ ] 配置 cursorSprite（光标 Sprite）

---

## 文档索引

| 文档 | 说明 |
|------|------|
| `requirements.md` | 需求文档（4 个用户故事） |
| `design.md` | 设计文档（架构、重构方案） |
| `tasks.md` | 任务列表（45 个任务） |
| `Kiro与Gemini设计路线对比分析.md` | 设计路线对比 |
| `锐评001审视报告.md` | 锐评审视 |
| `0文档锐评001.md` | Gemini 锐评 |
| `1设计锐评0.5.md` | Gemini 设计分析 |
| `1设计锐评1.0.md` | Gemini 详细设计 |
| `1设计执行001.md` | Gemini 执行指令 |


---

### 会话 4 - 2026-02-07（锐评002执行）

**锐评来源**: `2执行锐评002.md`

**用户需求**: 执行锐评002的最终修正指令

**完成任务**:
1. ✅ 重构 `FarmlandBorderManager.cs`
   - 新增 `IsCenterBlock(layerIndex, pos, predicate)` 重载
   - 新增 `CheckNeighborCenters(layerIndex, pos, predicate)` 重载
   - 新增 `CalculateBorderTileAt(layerIndex, pos, predicate)` 纯计算方法
   - 新增 `ApplyBorderTile(tilemaps, pos, tile)` 副作用方法
   - 新增 `GetPreviewTiles(layerIndex, centerPos)` 预览接口
   - 新增 `GetCenterTile(isFertilized)` 辅助方法

2. ✅ 创建 `FarmToolPreview.cs`
   - 单例模式
   - `FarmPreviewState` 枚举（Hidden, Valid, Invalid）
   - 自生能力：Awake 自动创建 GhostTilemap（Sorting Layer=Ground, Order=100）
   - 自生能力：Awake 自动创建 CursorRenderer（Sorting Layer=Effects, Order=0）
   - `UpdateHoePreview(layerIndex, cellPos)` - 调用 GetPreviewTiles 实现 1+8 预览
   - `UpdateWateringPreview(layerIndex, cellPos)` - 光标预览（无 1+8）
   - 颜色反馈：绿色=有效，红色=无效，蓝色=浇水

3. ✅ 修改 `GameInputManager.cs`
   - 新增 `UpdatePreviews()` 方法，在 Update 中调用
   - 根据手持物品类型路由到 FarmToolPreview 或 PlacementPreview
   - 修改 `TryTillSoil` 使用 `FarmToolPreview.IsValid()` 防抖
   - 修改 `TryWaterTile` 使用 `FarmToolPreview.IsValid()` 防抖
   - 使用 `PlacementGridCalculator.GetCellCenter()` 对齐坐标
   - 保留 `GetMouseWorldPosition()` 在 GameInputManager 中（不移到 PlacementGridCalculator）

**修改文件**:
- `Assets/YYY_Scripts/Farm/FarmlandBorderManager.cs` - 重构，新增预览接口
- `Assets/YYY_Scripts/Farm/FarmToolPreview.cs` - 新建
- `Assets/YYY_Scripts/Controller/Input/GameInputManager.cs` - 集成预览分发

**关键技术决策**:
1. `GetMouseWorldPosition()` 保留在 `GameInputManager`（PlacementGridCalculator 是纯数学类，不应耦合 Camera）
2. 坐标对齐使用 `PlacementGridCalculator.GetCellCenter(rawPos)`
3. FarmToolPreview 具备自生能力，无需手动配置场景

**遗留问题**:
- [ ] 需要在场景中添加 FarmToolPreview 组件（或挂载到 PersistentManagers）
- [ ] 需要配置 cursorSprite（光标 Sprite）
- [ ] Phase 4 集成测试待验收

---

### 会话 5 - 2026-02-07（预览不显示问题审视）

**用户需求**: 
> 手持锄头时看不到任何耕地预览，预览不会跟随鼠标移动，即使配置了图片也看不到光标跟随。FarmTileManager 的 Inspector 参数配置很奇怪，需要审视参数是否符合最新设计。

**审视结论**:

#### 问题 1：FarmToolPreview.Instance 为 null（根本原因）

`FarmToolPreview` 是 MonoBehaviour 单例，需要场景中有 GameObject 挂载该组件。

**调用链分析**:
```
GameInputManager.Update()
  → UpdatePreviews()
    → UpdateFarmToolPreview()
      → var farmPreview = FarmToolPreview.Instance;
      → if (farmPreview == null) return;  // ← 这里直接返回了！
```

场景中没有 FarmToolPreview 组件，所以 `Instance` 永远是 `null`，预览逻辑根本没有执行。

#### 问题 2：cursorSprite 设计问题

`cursorSprite` 是 `[SerializeField]` 字段，需要手动配置：
```csharp
[Header("光标配置")]
[SerializeField] private Sprite cursorSprite;
```

如果没有配置，光标框不显示。但 GhostTilemap 预览 Tiles 应该仍能显示（如果 FarmToolPreview 存在）。

#### 问题 3：LayerTilemaps 参数冗余

`LayerTilemaps.cs` 中存在新旧字段并存的问题：
```csharp
// 旧版字段（应该移除）
public Tilemap farmlandTilemap;

// 新版字段（当前使用）
public Tilemap farmlandCenterTilemap;
public Tilemap farmlandBorderTilemap;
```

从用户截图看，旧版字段 `farmlandTilemap` 被配置成了新版 Tilemap，造成冗余和混淆。

#### 设计缺陷总结

| 问题 | 严重程度 | 说明 |
|------|---------|------|
| FarmToolPreview 单例不自动创建 | 🔴 严重 | 导致预览完全不工作 |
| cursorSprite 强依赖外部资源 | 🟡 中等 | 没有默认值，需要手动配置 |
| 代码静默失败 | 🟡 中等 | 没有错误提示，难以排查 |
| LayerTilemaps 新旧字段并存 | 🟠 轻微 | Inspector 混乱，但不影响功能 |

**涉及的代码文件**:

| 文件 | 问题 |
|------|------|
| `FarmToolPreview.cs` | 单例需要场景组件，cursorSprite 无默认值 |
| `GameInputManager.cs` | 静默失败，无错误提示 |
| `LayerTilemaps.cs` | 新旧字段并存，应清理 `farmlandTilemap` |
| `FarmTileManager.cs` | Inspector 显示冗余字段 |

**本次会话性质**: 仅审视分析，未修改任何代码

**后续建议**:
1. 在场景中添加 FarmToolPreview 组件（挂载到 PersistentManagers 或独立 GameObject）
2. 清理 LayerTilemaps 中的旧版 `farmlandTilemap` 字段
3. 为 cursorSprite 提供默认值或运行时生成
4. 在 GameInputManager 中添加 Debug.LogWarning 提示 FarmToolPreview 缺失

---

### 会话 6 - 2026-02-07（锐评003执行）

**锐评来源**: `2执行锐评003.md`

**用户需求**: 执行锐评003的终极修正指令

**完成任务**:
1. ✅ 改造 `FarmToolPreview.cs` 为 Lazy Singleton
   - Instance getter 中自动创建实例（如果场景中没有）
   - 新增 `CreateProceduralCursorSprite()` 方法，程序化生成 16x16 白色方框光标
   - 无需手动配置 cursorSprite，无需手动添加场景组件

2. ✅ 清理 `LayerTilemaps.cs` 旧字段
   - 为 `farmlandTilemap` 添加 `[Obsolete]` 和 `[HideInInspector]` 特性
   - 为 `waterPuddleTilemap` 添加 `[Obsolete]` 和 `[HideInInspector]` 特性
   - Inspector 中不再显示这些旧字段

3. ✅ 修改 `GameInputManager.cs` 移除静默失败
   - `UpdateFarmToolPreview()` 不再检查 `farmPreview == null`
   - `HideFarmToolPreview()` 直接调用 `Instance.Hide()`
   - 信任 Lazy Singleton 保证返回有效实例

**修改文件**:
- `Assets/YYY_Scripts/Farm/FarmToolPreview.cs` - Lazy Singleton + 程序化 Sprite
- `Assets/YYY_Scripts/Farm/Data/LayerTilemaps.cs` - 清理旧字段
- `Assets/YYY_Scripts/Controller/Input/GameInputManager.cs` - 移除静默失败

**编译状态**: ✅ 成功（27 个警告，0 个错误）

**关键技术决策**:
1. Lazy Singleton 在 getter 中自动创建实例，真正实现"挂上去就能跑"
2. 程序化生成 16x16 白色方框光标，无需外部资源
3. `[Obsolete]` + `[HideInInspector]` 双重标记旧字段，既提醒迁移又清理 Inspector

**遗留问题**:
- [x] FarmToolPreview 单例自动创建 ✅ 已解决
- [x] cursorSprite 程序化生成 ✅ 已解决
- [x] LayerTilemaps 旧字段清理 ✅ 已解决
- [ ] Phase 4 集成测试待验收（用户手动测试）

---

### 会话 7 - 2026-02-08（预览不显示问题排查）

**用户需求**: 
> 执行锐评003后，运行游戏仍然看不到任何预览。日志疯狂输出。

**问题分析**:

从上下文摘要看，用户测试后反馈：
1. `UpdateFarmToolPreview` 被正确调用（每帧都在输出）
2. `farmPreview=FarmToolPreview` 实例存在
3. `layerIndex=0, cellPos=(15, 12, 0)` 坐标正确
4. 但预览仍然不显示

**可能的问题点**:
1. **FarmlandBorderManager.Instance 为 null** - 如果场景中没有这个组件，`GetPreviewTiles()` 不会被调用
2. **centerTileUnfertilized 未配置** - 如果 Tile 资源未配置，SetTile 无效
3. **Sorting Layer 不存在** - "Ground" 和 "Effects" 如果不存在，渲染会失败
4. **GhostTilemap 没有正确挂载到 Grid 下** - Tilemap 必须在 Grid 下才能正确显示

**本次修改**:

1. ✅ 添加一次性诊断日志 `LogDiagnosticsOnce()`
   - 检查 GhostTilemap 是否存在及其状态
   - 检查 GhostTilemapRenderer 的 Sorting Layer 配置
   - 检查 CursorRenderer 的 Sprite 和颜色
   - 检查 FarmlandBorderManager.Instance 是否存在
   - 检查 FarmTileManager.Instance 是否存在
   - 检查 Sorting Layer "Ground" 和 "Effects" 是否存在
   - 调用 `FarmlandBorderManager.ValidateTileResources()` 验证 Tile 资源配置

2. ✅ 添加 `GetPreviewTiles` 返回值诊断
   - 输出返回的 Tile 数量
   - 输出每个位置对应的 Tile 名称

3. ✅ 诊断日志不依赖 `showDebugInfo`
   - 一次性输出，用于排查问题
   - 不会每帧输出，避免日志爆炸

**修改文件**:
- `Assets/YYY_Scripts/Farm/FarmToolPreview.cs` - 添加诊断日志

**编译状态**: ✅ 成功（27 个警告，0 个错误）

**下一步**:
用户运行游戏后，查看 Console 中的诊断日志，根据输出定位问题：
- 如果 `FarmlandBorderManager.Instance` 为 null → 需要在场景中添加该组件
- 如果 `centerTileUnfertilized` 未配置 → 需要配置 Tile 资源
- 如果 Sorting Layer 不存在 → 需要在 Project Settings 中添加
- 如果 `GetPreviewTiles` 返回空 → 检查 `CanTillAt` 逻辑



---

### 会话 8 - 2026-02-08（Sorting Layer 修复）

**用户需求**: 
> 继续排查预览不显示问题，查看 Console 日志

**问题根因**:

从诊断日志发现：
```
  - Ground: ❌ 不存在
  - Effects: ❌ 不存在
```

**项目中的 Sorting Layers**（来自 `TagManager.asset`）：
- `Default`
- `Layer 1`
- `Layer 2`
- `Layer 3`
- `Building`
- `CloudShadow`

代码中设置的 `"Ground"` 和 `"Effects"` 在项目中根本不存在！Unity 静默回退到 `Default`，导致预览被地面 Tilemap 遮挡。

**修复内容**:

1. ✅ 修改 `FarmToolPreview.cs` 的 `EnsureComponents()` 方法
   - `ghostTilemapRenderer.sortingLayerName` 从 `"Ground"` 改为 `"Layer 1"`
   - `ghostTilemapRenderer.sortingOrder` 从 `100` 改为 `9999`（极高值确保在地面之上）
   - `cursorRenderer.sortingLayerName` 从 `"Effects"` 改为 `"Layer 1"`
   - `cursorRenderer.sortingOrder` 从 `0` 改为 `10000`（比 ghostTilemap 更高）

**修改文件**:
- `Assets/YYY_Scripts/Farm/FarmToolPreview.cs` - 修复 Sorting Layer 设置

**编译状态**: ✅ 成功（27 个警告，0 个错误）

**技术说明**:
- 使用 `"Layer 1"` 是因为它在项目中存在且位于 `Default` 之上
- 使用极高的 `sortingOrder`（9999/10000）确保预览在所有地面 Tile 之上
- 这是一个配置错误，不是逻辑错误

**待验收**:
- [ ] 用户运行游戏，确认预览是否正常显示
- [ ] 确认光标颜色反馈（绿色=可用，红色=不可用）
- [ ] 确认 1+8 预览 Tiles 是否正确显示

