# 10.1.0 全面持久化前夕 — 需求文档

> 来源：全面审查报告 V1 + V2 + 代码事实验证
> 创建时间：2026-02-17
> 工作区：`.kiro/specs/农田系统/10.1.0全面持久化前夕/`
> 覆盖范围：动作生命周期/输入缓存（P0）、存档字段补全（P0）、干燥/湿润耕地视觉（P1）

---

## 一、用户故事

### US-1：动作生命周期与输入缓存（P0）

> 作为玩家，我希望在使用农田工具（锄头/水壶/种子）时，动画播放期间的新输入不会丢失，而是被缓存，动画结束后自动消费缓存执行新的完整动作链，这样我可以流畅地连续操作农田。

#### 验收标准

**AC-1.1 输入缓存基础机制**
- 动画播放期间（`IsPerformingAction() == true`），农田工具的新点击输入被缓存而非丢弃
- 缓存内容：鼠标世界坐标（不缓存 cellPos，因为到执行时鼠标可能已移动）
- 只保留最后一次输入（后来的覆盖前面的）
- 动画结束时检查缓存，有缓存则启动新的完整动作链（判断有效性 → 距离判断 → 导航/执行 → 动画 → 完成）

**AC-1.2 长按左键处理**
- 长按左键（`GetMouseButton(0)`）时，动画结束后自动以当前鼠标位置作为缓存输入
- 如果当前鼠标位置有效（`IsValid() == true`），启动新的完整动作链
- 如果当前鼠标位置无效，不执行任何操作，回到 Idle 状态
- 长按不应对同一块已耕作的地重复播放空动画（`CanTillAt` 返回 false 时不播放动画）

**AC-1.3 导航中重新输入**
- 导航途中点击新位置时：
  - 中断当前导航
  - 更新预览到新位置（视觉必须同步刷新）
  - 重新锁定预览到新位置
  - 重新启动导航到新位置
- 预览视觉必须与锁定位置一致，不允许出现"预览在 A 点但实际耕的是 B 点"的情况

**AC-1.4 `LockPosition` 视觉同步**
- `FarmToolPreview.LockPosition()` 锁定时，必须自动刷新一次视觉到锁定位置
- 保证锁定位置和视觉显示一致，消除同帧解锁+重锁导致的视觉跳变

**AC-1.5 `IsPerformingAction()` 与农田工具的关系**
- 农田工具（Hoe/WateringCan）的输入处理不被 `IsPerformingAction()` 一刀切阻断
- 农田工具使用独立的执行保护（`_isExecutingFarming`），允许动画期间缓存新输入
- 非农田工具（武器/其他）保持原有 `IsPerformingAction()` 保护逻辑不变

**AC-1.6 种子种植的输入缓存**
- 种子种植同样支持输入缓存
- 缓存消费时需要重新验证：快照匹配 + 预览有效 + 距离判断 + 手持物品一致
- 种子袋剩余数量不足时，不执行缓存的种植操作

---

### US-2：存档字段补全（P0）

> 作为玩家，我希望存档和读档后，耕地的浇水状态（昨日浇水、浇水时间、水渍变体）能完整保留，这样读档后的农田视觉和逻辑与存档前完全一致。

#### 背景（代码事实）

当前 `FarmTileSaveData` 只有 5 个字段：
```csharp
public Vector3Int cellPosition;
public bool isWatered;
public bool hasCrop;
public int cropInstanceId;
public int moistureStateInt; // SoilMoistureState 枚举
```

运行时 `FarmTileData` 还有以下字段未被存档覆盖：
- `wateredYesterday`（bool）— 昨日是否浇水，影响日结逻辑
- `waterTime`（float）— 浇水时刻，影响水渍消退计时
- `puddleVariant`（int）— 水渍 Tile 变体索引（A/B/C），影响视觉一致性

`FarmTileManager.Save()` 只写入 5 个字段，`CreateTileFromSaveData()` 只恢复 5 个字段。

#### 验收标准

**AC-2.1 FarmTileSaveData 字段补全**
- 新增 `wateredYesterday`（bool）字段
- 新增 `waterTime`（float）字段
- 新增 `puddleVariant`（int）字段
- 字段类型与 `FarmTileData` 中的运行时字段一一对应

**AC-2.2 Save 链路适配**
- `FarmTileManager.Save()` 写入新增的 3 个字段
- 写入值直接取自对应 `FarmTileData` 的运行时值
- 不改变现有 5 个字段的写入逻辑

**AC-2.3 Load 链路适配**
- `FarmTileManager.CreateTileFromSaveData()` 恢复新增的 3 个字段
- 恢复后 `FarmTileData` 的运行时状态与存档前一致
- 恢复后视觉（水渍 Tile 变体）与存档前一致

**AC-2.4 旧存档兼容**
- 旧存档中不存在新增字段时，使用安全默认值：
  - `wateredYesterday` → `false`
  - `waterTime` → `0f`
  - `puddleVariant` → `0`
- 旧存档读取不报错、不崩溃
- 默认值不会导致异常的视觉或逻辑行为

**AC-2.5 存档完整性验证**
- 存档 → 读档后，所有 `FarmTileData` 字段值与存档前完全一致
- 特别验证：浇水状态 + 水渍变体 + 昨日浇水标记在存读档后保持不变

---

### US-3：干燥/湿润耕地视觉渲染（P1）

> 作为玩家，我希望耕地在不同含水状态下有不同的视觉表现（干燥→浇水→水渍消退→湿润渐变），并且这些视觉状态能被持久化，这样农田看起来更真实、更有生命力。

#### 背景（代码事实）

- `FarmVisualManager` 中存在 `dryFarmlandTile` 和 `wetDarkTile` 两个 `TileBase` 字段，但 `UpdateTileVisual()` 从未使用它们
- 当前 `UpdateTileVisual()` 只操作水渍叠加层（`waterPuddleTilemapNew`），不修改耕地 Tile 本身
- `SoilMoistureState` 枚举已有三个状态：`Dry`、`WetWithPuddle`、`WetDark`
- 耕地 Tile 的渲染层与水渍叠加层是分离的

#### 验收标准

**AC-3.1 锄地 → 干燥耕地 Tile**
- 锄地成功后，耕地位置渲染 `dryFarmlandTile`
- 干燥耕地 Tile 替换原始地面 Tile（或在耕地专用层渲染）
- 对应 `SoilMoistureState.Dry`

**AC-3.2 浇水 → 水渍叠加（耕地保持干燥）**
- 浇水后，水渍叠加层显示水渍 Tile（现有逻辑，已正常工作）
- 耕地 Tile 本身保持 `dryFarmlandTile` 不变
- 对应 `SoilMoistureState.WetWithPuddle`

**AC-3.3 水渍消退 → 湿润耕地渐变**
- 水渍消退后（现有逻辑移除水渍叠加层），耕地进入渐变阶段
- 在 30 游戏分钟内，耕地 Tile 从 `dryFarmlandTile` 渐变为 `wetDarkTile`
- 渐变方式要"灵动"，不能瞬间切换（可用 Shader 渐变、透明度过渡、或分阶段 Tile 替换）
- 对应 `SoilMoistureState.WetDark`

**AC-3.4 湿润耕地的持续与回退**
- 湿润耕地（`WetDark`）在下一次日结时，如果当天未浇水，回退为 `Dry`
- 如果当天浇水了，保持 `WetDark` 状态
- 回退为 `Dry` 时，视觉同步切换为 `dryFarmlandTile`

**AC-3.5 视觉状态持久化**
- `SoilMoistureState` 已通过 `moistureStateInt` 存档（现有逻辑）
- 读档后根据 `SoilMoistureState` 恢复正确的耕地 Tile 视觉：
  - `Dry` → `dryFarmlandTile`
  - `WetWithPuddle` → `dryFarmlandTile` + 水渍叠加
  - `WetDark` → `wetDarkTile`
- 渐变进度不需要持久化（读档后直接显示目标状态即可）

**AC-3.6 与现有水渍系统兼容**
- 不修改现有水渍叠加层的逻辑（水渍显示/消退已正常工作）
- 耕地 Tile 渲染与水渍叠加层独立，互不干扰
- `dryFarmlandTile` 和 `wetDarkTile` 通过 Inspector 配置，不硬编码

---

## 二、边界情况

| 编号 | 场景 | 预期行为 |
|------|------|----------|
| E-1 | 动画播放中快速连续点击 5 次不同位置 | 只缓存最后一次点击，动画结束后对最后位置执行完整动作链 |
| E-2 | 动画播放中切换手持物品（锄头→水壶） | 缓存失效，不执行缓存的操作（手持物品不一致） |
| E-3 | 导航途中目标耕地被其他系统移除 | 到达后 `CanTillAt`/`CanWaterAt` 返回 false，取消操作，回到 Idle |
| E-4 | 长按左键但鼠标悬停在无效区域 | 动画结束后检测到无效位置，不执行操作，回到 Idle |
| E-5 | 长按左键在同一块已耕作的地上 | `CanTillAt` 返回 false，不播放空动画，不执行操作 |
| E-6 | 旧存档缺少新增的 3 个字段 | 使用默认值（`wateredYesterday=false`, `waterTime=0f`, `puddleVariant=0`），正常加载 |
| E-7 | 存档时耕地处于渐变中途（Dry→WetDark） | 存档 `moistureStateInt` 为当前状态，读档后直接显示目标状态（不恢复渐变进度） |
| E-8 | 日结时耕地处于 `WetWithPuddle` 状态 | 水渍消退逻辑正常执行，进入渐变阶段 |
| E-9 | 多块耕地同时处于不同含水状态 | 每块耕地独立管理自己的 `SoilMoistureState`，互不影响 |
| E-10 | 导航中重新点击同一位置 | 不中断导航，继续前往原目标（目标未变化） |
| E-11 | 种子袋在缓存期间被消耗完 | 缓存消费时重新验证种子数量，不足则取消操作 |
| E-12 | 浇水后立即存档再读档 | 水渍变体（puddleVariant）保持一致，视觉不跳变 |

---

## 三、非功能需求

| 编号 | 需求 | 说明 |
|------|------|------|
| NF-1 | 输入缓存零感知延迟 | 缓存消费到动作链启动的间隔 ≤ 1 帧，玩家感受不到卡顿 |
| NF-2 | 存档体积增长可控 | 新增 3 个字段（bool + float + int）每块耕地增加约 20 字节，100 块耕地增加约 2KB，可忽略 |
| NF-3 | 渐变效果性能 | 耕地渐变（Dry→WetDark）不使用逐帧 Update，优先使用 Shader 参数插值或协程定时更新 |
| NF-4 | 向后兼容 | 旧存档（无新增字段）必须能正常加载，不报错不崩溃 |

---

## 四、设计约束

| 编号 | 约束 | 说明 |
|------|------|------|
| DC-1 | 新增并兼容 | 所有修改不能破坏现有功能，必须在现有代码基础上新增 |
| DC-2 | 先设计后实现 | 所有修改必须先提交设计方案，用户审核通过后才能修改代码 |
| DC-3 | 玩家位置 = Collider 中心 | 所有涉及玩家位置的计算使用 `playerCollider.bounds.center` |
| DC-4 | Unity 6 API | 使用 `compositeOperation` 替代 `usedByComposite`，使用 `FindObjectsByType` 替代 `FindObjectsOfType` |
| DC-5 | 农田工具独立保护 | 农田工具使用 `_isExecutingFarming` 保护，不依赖 `IsPerformingAction()` 的一刀切阻断 |
| DC-6 | 水渍叠加层独立 | 耕地 Tile 渲染与水渍叠加层分离，互不干扰 |
| DC-7 | Inspector 配置优先 | `dryFarmlandTile`、`wetDarkTile`、水渍 Tile 变体均通过 Inspector 配置，不硬编码 |

---

## 五、正确性属性

以下属性用于验证实现的正确性，可作为 Property-Based Testing 的基础。

| 编号 | 属性 | 关联验收标准 |
|------|------|-------------|
| CP-1 | 对于任意输入序列，动画播放期间的输入永远不会丢失——要么被缓存，要么被后续输入覆盖 | AC-1.1 |
| CP-2 | 缓存中最多只有一个输入，后来的总是覆盖前面的 | AC-1.1 |
| CP-3 | 缓存消费时启动的动作链与直接点击启动的动作链完全一致（判断有效性→距离→导航/执行→动画→完成） | AC-1.1 |
| CP-4 | 长按左键时，动画结束后的行为等价于在当前鼠标位置单击一次 | AC-1.2 |
| CP-5 | 导航中重新输入后，预览位置 === 锁定位置 === 导航目的地，三者始终一致 | AC-1.3, AC-1.4 |
| CP-6 | `LockPosition()` 调用后，视觉显示的位置 === 锁定的 cellPos | AC-1.4 |
| CP-7 | 非农田工具的 `IsPerformingAction()` 保护逻辑不受任何修改影响 | AC-1.5 |
| CP-8 | 对于任意 `FarmTileData`，`Save()` → `CreateTileFromSaveData()` 后所有 8 个字段值不变 | AC-2.1~AC-2.5 |
| CP-9 | 旧存档（5 字段）加载后，新增 3 个字段取默认值，不报错 | AC-2.4 |
| CP-10 | 对于任意 `SoilMoistureState`，存档→读档后耕地 Tile 视觉与该状态对应的 Tile 一致 | AC-3.5 |
| CP-11 | 水渍消退后 30 游戏分钟内，耕地从 `Dry` 渐变为 `WetDark`，不存在瞬间跳变 | AC-3.3 |
| CP-12 | 每块耕地的 `SoilMoistureState` 独立管理，修改一块不影响其他块 | E-9 |

---

## 六、术语表

| 术语 | 说明 |
|------|------|
| 动作链 | 一次完整的农田操作流程：判断有效性 → 距离判断 → 导航（如需）→ 执行 → 动画 → 完成 |
| 输入缓存 | 动画播放期间暂存的玩家输入，动画结束后消费 |
| 水渍叠加层 | `waterPuddleTilemapNew` 上的水渍 Tile，覆盖在耕地 Tile 之上 |
| 干燥耕地 | `dryFarmlandTile`，锄地后的初始视觉状态 |
| 湿润耕地 | `wetDarkTile`，水渍消退后渐变的最终视觉状态 |
| 渐变 | 从干燥到湿润的视觉过渡，持续 30 游戏分钟，要求灵动不瞬切 |
| `SoilMoistureState` | 土壤含水状态枚举：`Dry`（干燥）、`WetWithPuddle`（有水渍）、`WetDark`（湿润） |
| `_isExecutingFarming` | 农田工具专用的执行保护标志，替代 `IsPerformingAction()` 的一刀切阻断 |
