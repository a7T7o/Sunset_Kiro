# 9.0.4 智能交互升级 - 任务列表

## 任务概览

| 阶段 | 任务数 | 预估时间 |
|------|--------|---------|
| 0. 基础设施核验（锐评002补充） | 2 | 15 分钟 |
| 1. FarmToolPreview 修改 | 4 | 30 分钟 |
| 2. GameInputManager 重构 | 7 | 60 分钟 |
| 3. 边界情况处理 | 3 | 20 分钟 |
| 4. 测试与验证 | 3 | 30 分钟 |

---

## 阶段 0：基础设施核验（锐评002补充）

**优先级**：最高，必须在其他任务之前完成

- [x] 0.1 验证 PlayerAutoNavigator 回调机制
  - 检查是否支持 `MoveTo(Vector3 target, Action onComplete)` 或等效方法
  - 如果不支持，需要先添加回调支持
  - 验证 `onComplete` 在 `Stop()` 之后、下一帧逻辑开始之前被调用

- [x] 0.2 验证回调时序
  - 添加帧号日志验证 T1-T3 同帧执行
  - 验证代码：
    ```csharp
    // 在 onComplete 回调中
    Debug.Log($"[FarmNav] onComplete at frame {Time.frameCount}");
    // 在 ExecuteAction 中
    Debug.Log($"[FarmNav] ExecuteAction at frame {Time.frameCount}");
    ```
  - 两者必须相同，否则需要调整实现

---

## 阶段 1：FarmToolPreview 修改

- [x] 1.1 新增公开属性
  - 添加 `IsInRange` 属性（bool）
  - 添加 `CurrentCursorPos` 属性（Vector3）
  - 添加 `CurrentCellPos` 属性（Vector3Int）
  - 添加 `CurrentLayerIndex` 属性（int）

- [x] 1.2 修改 UpdateHoePreview
  - 从 `isValid` 判定中移除 `&& withinReach`
  - 单独设置 `IsInRange = withinReach`
  - 记录 `CurrentCursorPos`、`CurrentCellPos`、`CurrentLayerIndex`

- [x] 1.3 修改 UpdateWateringPreview
  - 从 `isValid` 判定中移除 `&& withinReach`
  - 单独设置 `IsInRange = withinReach`
  - 记录位置信息

- [x] 1.4 修改 UpdateSeedPreview
  - 从 `isValid` 判定中移除 `&& withinReach`
  - 单独设置 `IsInRange = withinReach`
  - 记录位置信息

---

## 阶段 2：GameInputManager 重构

- [x] 2.1 新增状态机变量（锐评002补充）
  - 新增 `FarmNavState` 枚举（Idle, Navigating, Executing）
  - 新增 `_farmNavState` 字段
  - 新增 `_farmNavigationAction` 字段
  - 新增 `_cachedSeedData` 字段

- [x] 2.2 新增 StartFarmingNavigation 方法
  - 接收目标位置和回调
  - 设置 `_farmNavState = Navigating`
  - 调用 `autoNavigator.SetDestination`
  - 启动监控协程

- [x] 2.3 新增 WaitForNavigationComplete 协程
  - 监控导航状态
  - 检测到达条件
  - 设置 `_farmNavState = Executing`
  - 执行回调
  - 设置 `_farmNavState = Idle`

- [x] 2.4 重构 TryTillSoil
  - 使用 `FarmToolPreview.IsValid` 检查有效性
  - 使用 `FarmToolPreview.IsInRange` 分流
  - 近距离：调用 `ExecuteTillSoil`
  - 远距离：调用 `StartFarmingNavigation`

- [x] 2.5 重构 TryWaterTile
  - 使用 `FarmToolPreview.IsValid` 检查有效性
  - 使用 `FarmToolPreview.IsInRange` 分流
  - 近距离：调用 `ExecuteWaterTile`
  - 远距离：调用 `StartFarmingNavigation`

- [x] 2.6 重构 TryPlantSeed
  - 使用 `FarmToolPreview.IsValid` 检查有效性
  - 使用 `FarmToolPreview.IsInRange` 分流
  - 缓存 seedData 到 `_cachedSeedData`
  - 新增 `IsHoldingSameSeed` 检查

- [x] 2.7 新增 ExecuteTillSoil / ExecuteWaterTile / ExecutePlantSeed
  - 提取纯执行逻辑
  - 不含距离检查
  - 供直接调用和回调使用

- [x] 2.8 新增 FarmingSnapshot 快照机制（锐评006补充）
  - 新增 `FarmingSnapshot` 结构体
  - 新增 `_farmingSnapshot` 字段
  - 新增 `ValidateSnapshot()` 方法
  - 新增 `ClearSnapshot()` 方法

- [x] 2.9 在 TryPlantSeed 中集成快照（锐评006补充）
  - 远距离种植时创建快照
  - 到达后校验快照
  - 校验失败时播放取消音效

---

## 阶段 3：边界情况处理

- [x] 3.1 新增 CancelFarmingNavigation 方法（锐评002补充）
  - 检查 `_farmNavState != Idle`
  - 重置 `_farmNavState = Idle`
  - 清理 `_farmNavigationAction = null`
  - 清理 `_cachedSeedData = null`
  - 停止导航 `_playerNavigator.Stop()`
  - 停止协程 `StopCoroutine(_farmingNavigationCoroutine)`
  - 输出 Debug Log

- [x] 3.2 切换工具时取消导航
  - 在 `HandleHotbarSelection` 中添加 `CancelFarmingNavigation` 调用
  - 新增 `_farmingNavigationCoroutine` 字段

- [x] 3.3 打开面板时取消导航
  - 在 `HandlePanelHotkeys` 中添加 `CancelFarmingNavigation` 调用

- [x] 3.4 手动移动时取消导航
  - 在 `HandleMovement` 中添加 `CancelFarmingNavigation` 调用
  - 仅在有输入时取消

---

## 阶段 4：测试与验证

- [x] 4.1 编译验证
  - 确保无编译错误
  - 确保无警告

- [x] 4.2 功能测试
  - 近距离锄地/浇水/种植
  - 远距离导航→锄地/浇水/种植
  - 导航中断场景

- [x] 4.3 更新 memory.md
  - 记录本次会话
  - 记录修改的文件
  - 记录遗留问题

---

## 依赖关系

```
0.1 ──── 0.2  ← Phase 0 必须先完成
           │
           ▼
1.1 ──┬── 1.2
      ├── 1.3
      └── 1.4
           │
           ▼
      2.1 ──── 2.2 ──── 2.3
           │
           ▼
      2.4 ──┬── 2.5 ──── 2.6
            │
            ▼
           2.7
           │
           ▼
      3.1 ──── 3.2 ──── 3.3 ──── 3.4
           │
           ▼
      4.1 ──── 4.2 ──── 4.3
```

---

## 验收标准

### 必须通过
- [ ] 远处可耕作地块显示绿色光标
- [ ] 点击远处有效地块触发导航
- [ ] 导航到达后自动执行动作
- [ ] 近距离操作与 9.0.3 行为一致
- [ ] 无效地块显示红色光标且点击无效

### 边界情况
- [ ] 导航过程中切换工具 → 取消导航
- [ ] 导航过程中打开面板 → 取消导航
- [ ] 导航过程中手动移动 → 取消导航
- [ ] 导航到达但目标已失效 → 不执行动作

### 锐评002 补充验收
- [ ] Phase 0 基础设施核验通过
- [ ] 状态机正确转换（Idle → Navigating → Executing → Idle）
- [ ] 中断时状态正确重置为 Idle
- [ ] 回调时序验证通过（T1-T3 同帧）

### 锐评006 补充验收
- [ ] 种子远距离种植时创建快照
- [ ] 快照校验失败时不执行动作
- [ ] 快照校验失败时播放取消音效
