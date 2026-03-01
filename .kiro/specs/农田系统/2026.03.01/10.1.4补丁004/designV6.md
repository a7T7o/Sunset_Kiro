# 补丁004 设计文档 V6 — 农田预览系统最终修复

> 来源：`补丁004V6全面审查报告.md`（15章完整报告）
> 基础：V1~V4 已实施代码 + V5/V5.1/V5.2/锐评001~005 全部审查结论
> 用户确认：4个待确认事项全部通过（002执行001.md）
> 排除：模块R（种子放置化）→ 记代办，不在V6中实施

---

## 目录

1. [模块Q：极限操作绝对锁定](#一模块q极限操作绝对锁定)
2. [模块S：锄头全状态农作物清除](#二模块s锄头全状态农作物清除)
3. [模块V：Ghost增量差集合并c+b方向](#三模块vghost增量差集合并cb方向)
4. [模块N'：b层统一拦截（含hasCrop）](#四模块nb层统一拦截含hascrop)
5. [模块T：浇水随机重写](#五模块t浇水随机重写)
6. [模块U：浇水b层纳入](#六模块u浇水b层纳入)
7. [模块P'：水渍tile移出isValid保护圈](#七模块p水渍tile移出isvalid保护圈)
8. [模块O''：Shader染色配合三分支](#八模块oshader染色配合三分支)
9. [模块P''：播种b层拦截](#九模块p播种b层拦截)
10. [正确性属性汇总](#十正确性属性汇总)
11. [涉及文件汇总](#十一涉及文件汇总)

---

## 一、模块Q：极限操作绝对锁定（🔴最高优先级）

> 涉及文件：`GameInputManager.cs`
> 涉及方法：`HandleMovement`、`CancelFarmingNavigation`

### 1.1 问题

玩家左键耕地动画开始瞬间按WASD → 动画播放但耕地不出现、预览不消失。

### 1.2 根因

`HandleMovement` WASD中断分支无条件调用 `CancelFarmingNavigation` → 无条件清除 `_isExecutingFarming` → `ForceUnlock` → 玩家移动 → 动画中断 → 回调链断裂。

### 1.3 方案

核心原则：`_isExecutingFarming = true` 时，WASD只清空等待队列，不取消执行，不解锁移动。

Q1：HandleMovement WASD中断分支增加 `wasExecuting` 检查
Q2：CancelFarmingNavigation 增加执行保护（`_isExecutingFarming` 时只清理导航状态，不清除执行标志）

### 1.4 当前代码状态：✅ 已实施

代码审查确认 HandleMovement（第457行）已有 `wasExecuting` 检查，CancelFarmingNavigation（第2008行）已有执行保护。


### 1.5 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-Q1 | `_isExecutingFarming=true` 时 WASD 不调用 `CancelFarmingNavigation` | 代码审查 |
| CP-Q2 | `_isExecutingFarming=true` 时 WASD 不调用 `ForceUnlock` | 代码审查 |
| CP-Q3 | `CancelFarmingNavigation` 中执行中保持执行状态不变 | 代码审查 |

---

## 二、模块S：锄头全状态农作物清除（🔴高优先级）

> 涉及文件：`FarmToolPreview.cs`、`GameInputManager.cs`、`CropController.cs`
> 涉及方法：`UpdateHoePreview`、`ExecuteFarmAction`、`OnFarmActionAnimationComplete`、`TryEnqueueFarmTool`、`ExecuteTillSoil`（Update中_pendingTileUpdate监听）

### 2.1 问题

锄头对所有阶段农作物都能除去。有农作物的耕地不显示任何预览，无农作物的已有耕地显示放置系统红方框。

### 2.2 方案

S1：UpdateHoePreview 新增 `hasCrop` 判断（检测任何状态的 cropController）
S2：isValid 扩展为 `!hasObstacle && (canTill || hasCrop)`
S3：三分支视觉：canTill→绿色1+8 / hasCrop→空白 / 其他→放置系统红方框
S4：新增 `FarmActionType.RemoveCrop` 枚举值
S5：CropController 新增 `public void RemoveCrop()` 方法
S6：ExecuteFarmAction 新增 RemoveCrop 分支（Crush动画 + _pendingTileUpdate）
S7：Update 中 _pendingTileUpdate 监听新增 RemoveCrop 分支
S8：OnFarmActionAnimationComplete 兜底新增 RemoveCrop 分支
S9：TryEnqueueFarmTool 根据 `_hasCrop` 判断入队类型（Till vs RemoveCrop）
S10：暴露 `_canTill` / `_hasCrop` 属性供 TryEnqueueFarmTool 使用

### 2.3 当前代码状态

- S1/S2/S3/S10：✅ 已实施（UpdateHoePreview 有 hasCrop 三分支 + _canTill/_hasCrop 属性）
- S4：✅ 已实施（FarmActionType 枚举已有 RemoveCrop）
- S5：✅ 已实施（CropController.RemoveCrop() 已存在，第557行）
- S6：❌ 未实施（ExecuteFarmAction 缺少 RemoveCrop case）
- S7：❌ 未实施（Update 中 _pendingTileUpdate switch 缺少 RemoveCrop case）
- S8：❌ 未实施（OnFarmActionAnimationComplete 兜底 switch 缺少 RemoveCrop case）
- S9：❌ 未实施（TryEnqueueFarmTool 只判断 Hoe→Till / Water，缺少 RemoveCrop 入队逻辑）

### 2.4 待实施改动

S6 — ExecuteFarmAction 新增：
```csharp
case FarmActionType.RemoveCrop:
    FaceTarget(request.worldPos);
    playerInteraction?.RequestAction(PlayerAnimController.AnimState.Crush);
    _pendingTileUpdate = request;
    _tileUpdateTriggered = false;
    break;
```

S7 — Update 中 _pendingTileUpdate 监听新增：
```csharp
case FarmActionType.RemoveCrop:
    ExecuteRemoveCrop(req.layerIndex, req.cellPos);
    break;
```

S8 — OnFarmActionAnimationComplete 兜底新增：
```csharp
case FarmActionType.RemoveCrop:
    ExecuteRemoveCrop(req.layerIndex, req.cellPos);
    break;
```

S9 — TryEnqueueFarmTool 入队类型判断：
```csharp
FarmActionType type;
if (tool.toolType == ToolType.Hoe)
    type = farmPreview.HasCrop ? FarmActionType.RemoveCrop : FarmActionType.Till;
else
    type = FarmActionType.Water;
```

新增 ExecuteRemoveCrop 方法：
```csharp
private bool ExecuteRemoveCrop(int layerIndex, Vector3Int cellPos)
{
    var tileData = FarmTileManager.Instance?.GetTileData(layerIndex, cellPos);
    if (tileData?.cropController != null)
    {
        tileData.cropController.RemoveCrop();
        return true;
    }
    return false;
}
```

### 2.5 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-S1 | 有农作物的耕地 `isValid=true` | 代码审查 |
| CP-S2 | 有农作物的耕地不显示任何预览 | 游戏内验收 |
| CP-S3 | 无农作物的已有耕地显示放置系统红方框 | 游戏内验收 |
| CP-S4 | 锄头点击有农作物的耕地 → 清除农作物，耕地保留 | 游戏内验收 |
| CP-S5 | 所有作物状态都可被锄头清除 | 游戏内验收 |
| CP-S6 | RemoveCrop 使用 Crush 动画 | 代码审查 |

---

## 三、模块V：Ghost增量差集合并c+b方向（🔴高优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`

### 3.1 问题

c层附近ghost增量差集只取c层方向做差集，忽略b层（队列预览）的方向贡献。

### 3.2 方案

增量差集对比时，合并c层和b层的方向取并集后再做差集。

### 3.3 当前代码状态：✅ 已实施

代码审查确认 UpdateHoePreview 第520-530行已有 mergedU/D/L/R 合并 c+b 方向的逻辑。

### 3.4 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-V1 | 增量差集合并c+b方向后再做差集 | 代码审查 |

---

## 四、模块N'：b层统一拦截（含hasCrop）（🔴高优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`

### 4.1 问题

已在队列中的格子不应再次操作（canTill/hasCrop 都应被拦截）。

### 4.2 当前代码状态：✅ 已实施

代码审查确认 UpdateHoePreview 第453-457行已有 `queuePreviewPositions.Contains(cellPos)` 拦截 canTill + hasCrop。

### 4.3 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-N1 | 已在队列中的格子 canTill=false 且 hasCrop=false | 代码审查 |


---

## 五、模块T：浇水随机重写（🟡中等优先级）

> 涉及文件：`FarmToolPreview.cs`、`GameInputManager.cs`
> 涉及方法：`UpdateWateringPreview`、`UpdateHoePreview`（重置标志）、`UpdateSeedPreview`（重置标志）、`ExecuteWaterTile`

### 5.1 问题

当前浇水随机逻辑是"进入新格子就随机"，用户要求：
1. 切换到水壶时随机默认样式
2. 鼠标移动不切换样式
3. 执行浇水 + 移出当前格子才触发随机
4. 锁定新样式直到下次浇水+移出

### 5.2 方案

T1：新增字段 `_needsNewPuddleVariant`（浇水执行后设为true）+ `_wateringModeInitialized`（检测切换）
T2：UpdateWateringPreview 开头检测切换到水壶 → 随机默认样式
T3：移除"进入新格子就随机"逻辑
T4：新增"浇水后移出才随机"逻辑：`_needsNewPuddleVariant && cellPos != _lastWateringCellPos`
T5：ExecuteWaterTile 成功后设置 `_needsNewPuddleVariant = true` + 记录 `_lastWateringCellPos`
T6：UpdateHoePreview / UpdateSeedPreview 开头重置 `_wateringModeInitialized = false`

### 5.3 当前代码状态：❌ 未实施

UpdateWateringPreview 第703行仍是 `if (cellPos != _lastWateringCellPos)` 随机逻辑。

### 5.4 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-T1 | 切换到水壶时随机一次默认样式 | 游戏内验收 |
| CP-T2 | 鼠标在耕地间移动时样式不变 | 游戏内验收 |
| CP-T3 | 浇水后鼠标仍在同一格子 → 样式不变 | 游戏内验收 |
| CP-T4 | 浇水后移出格子 → 触发随机 → 锁定新样式 | 游戏内验收 |
| CP-T5 | ghost/队列/执行三层使用相同 puddleVariant | 代码审查 |

---

## 六、模块U：浇水b层纳入（🟡中等优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateWateringPreview`

### 6.1 问题

canWater 判断不知道 b 层（queuePreviewPositions），已入队的格子仍显示可浇水。

### 6.2 方案

canWater 计算后增加：
```csharp
if (canWater && queuePreviewPositions.Contains(cellPos))
    canWater = false;
```

### 6.3 当前代码状态：❌ 未实施

### 6.4 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-U1 | 已在队列中的格子 canWater=false | 代码审查 |

---

## 七、模块P'：水渍tile移出isValid保护圈（🟡中等优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateWateringPreview`

### 7.1 问题

当前水渍tile放置在 `if (isValid)` 内部，`!isValid` 时 ghostTilemap 为空，shader 染红色但无载体（看不到红色效果）。

### 7.2 方案

将水渍tile放置逻辑移出 `if (isValid)` 保护圈，无论 isValid 都放置水渍tile作为 shader 载体。

### 7.3 当前代码状态：❌ 未实施

### 7.4 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-P1 | 无论 isValid，ghostTilemap 都放置水渍tile | 代码审查 |
| CP-P2 | isValid=false 时水渍tile被 shader 染红色 | 游戏内验收 |

---

## 八、模块O''：Shader染色配合三分支（🟡中等优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateHoePreview`

### 8.1 问题

Shader 染色需要配合模块S的三分支：canTill时绿色，hasCrop时无载体设Color.clear，其他也无载体设Color.clear。

### 8.2 当前代码状态：✅ 已实施

代码审查确认 UpdateHoePreview 末尾已有 canTill→validColor / else→Color.clear 的逻辑。

### 8.3 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-O1 | canTill 时 shader 绿色，其他时 Color.clear | 代码审查 |

---

## 九、模块P''：播种b层拦截（🟡低优先级）

> 涉及文件：`FarmToolPreview.cs`
> 涉及方法：`UpdateSeedPreview`

### 9.1 问题

canPlant 判断不知道 b 层，已入队的格子仍显示可播种。

### 9.2 方案

canPlant 计算后增加：
```csharp
if (canPlant && queuePreviewPositions.Contains(cellPos))
    canPlant = false;
```

### 9.3 当前代码状态：❌ 未实施

### 9.4 正确性属性

| 编号 | 属性 | 验证方式 |
|------|------|---------|
| CP-P3 | 已在队列中的格子 canPlant=false | 代码审查 |

---

## 十、正确性属性汇总

| 编号 | 属性 | 模块 | 状态 |
|------|------|------|------|
| CP-Q1~Q3 | 绝对锁定三条 | Q | ✅ 已实施 |
| CP-S1~S6 | 锄头农作物六条 | S | 🔶 部分实施 |
| CP-V1 | 增量差集合并 | V | ✅ 已实施 |
| CP-N1 | b层统一拦截 | N' | ✅ 已实施 |
| CP-T1~T5 | 浇水随机五条 | T | ❌ 未实施 |
| CP-U1 | 浇水b层 | U | ❌ 未实施 |
| CP-P1~P2 | 水渍移出isValid | P' | ❌ 未实施 |
| CP-O1 | Shader三分支 | O'' | ✅ 已实施 |
| CP-P3 | 播种b层 | P'' | ❌ 未实施 |

---

## 十一、涉及文件汇总

| 文件 | 涉及模块 | 修改内容 |
|------|---------|---------|
| GameInputManager.cs | S, T | ExecuteFarmAction RemoveCrop分支 + Update _pendingTileUpdate RemoveCrop + OnFarmActionAnimationComplete RemoveCrop + TryEnqueueFarmTool 入队类型 + ExecuteRemoveCrop新方法 + ExecuteWaterTile设置标志 |
| FarmToolPreview.cs | T, U, P', P'' | UpdateWateringPreview 随机重写+b层+水渍移出isValid + UpdateSeedPreview b层拦截 + 新增浇水模式字段 + UpdateHoePreview/UpdateSeedPreview 重置标志 |
| CropController.cs | — | 无需修改（RemoveCrop已存在） |

