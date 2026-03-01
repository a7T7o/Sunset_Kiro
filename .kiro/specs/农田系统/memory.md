# 农田系统 - 开发记忆

> 本卷起始于 memory_0.md 之后。历史详细记录请查阅 memory_0.md。

## 模块概述

农田系统，包括：
- 耕地状态管理（锄地、浇水）
- 作物生长周期
- 水分系统
- 季节适应性
- **种子种植**（SeedData 专属，与树苗分离）

## 当前状态

- **完成度**: 65%
- **最后更新**: 2026-02-25
- **状态**: 🚧 进行中
- **当前焦点**: 10.1.5补丁005（种子并入放置系统）+ 10.1.4补丁004（三层预览架构）

---

## 🔴 重要架构决策 🔴

### 1. 种子归属农田系统（2025-12-30）
种子（SeedData）由农田系统管理，与树苗（SaplingData）分离。

### 2. 彻底解耦架构（2026-01-27）
FarmingManagerNew 已废弃，FarmTileManager/CropController/CropManager 各自自治。

### 3. 联邦制预览架构（2026-02-06）
FarmToolPreview 独立于 PlacementPreview，共享 PlacementGridCalculator。

### 4. 种子应并入放置系统（2026-02-25，待执行）
种子走 FIFO 队列是架构错位，放置系统是正确归宿（树苗是成功案例）。

---

## 关键文件索引

### 核心脚本
| 文件 | 说明 |
|------|------|
| `FarmTileManager.cs` | 耕地状态管理，自治时间事件 |
| `FarmlandBorderManager.cs` | 边界视觉管理，预览接口 |
| `FarmToolPreview.cs` | 农田工具预览组件 |
| `CropController.cs` | 作物控制器，自治时间事件 |
| `CropManager.cs` | 作物工厂 |
| `GameInputManager.cs` | 锄地/浇水/种植/收获调用入口 |
| `PlacementGridCalculator.cs` | 坐标对齐（共享） |

---

## 子工作区索引

| 编号 | 名称 | 状态 | 说明 |
|------|------|------|------|
| 0~7 | 早期工作区 | ✅ 完成 | 详见 memory_0.md |
| 8 | 修复 | 🚧 进行中 | P0 bug 修复 |
| 9.0.1 | 重生之我在种田 | ✅ 完成 | 进度整理 |
| 9.0.2 | 放置农田 | 🚧 进行中 | 预览系统设计 |
| 9.0.5 | 智能交互bug修复 | ✅ 代码完成 | 锁定机制+5状态机，待验收 |
| 10.0.1 | 农作物设计与完善 | ✅ 代码完成 | 种子袋+作物7样式+CropController重构，待验收 |
| 10.0.2.1 | 纠正（废弃CropManager） | ✅ 完成 | 全面对齐树木Prefab驱动模式 |
| 10.1.0 | 全面持久化前夕 | 🚧 进行中 | 全面审查报告完成 |
| 10.1.1补丁001 | 交互漏洞修补 | ✅ 完成 | 二次修复完成，待二次验收 |
| 10.1.1补丁002 | 交互体验优化 | 🚧 进行中 | design V2 + tasks V2 完成，待审核 |
| 10.1.1补丁003 | 预览系统改造 | ✅ 代码完成 | shader覆盖层+双Tilemap分离+队列预览，待验收 |
| 10.1.4补丁004 | 三层预览架构重构 | 🚧 进行中 | 005锐评系列完成，种子剥离方向确定 |
| 10.1.5补丁005 | 种子并入放置系统 | ✅ 代码完成 | 全部代码任务完成，待手动验收 |

---

## 上一卷最后记录回顾

memory_0.md 最后记录（会话2续53，2026-02-25）：
- 10.1.5补丁005 完成锐评001审核 + design.md + tasks.md 生成
- 种子并入放置系统方案确定，15个子任务待执行

---

## 会话记录

（新卷起始，后续会话从此处追加）

### 元讨论 - 2026-02-25（Memory 分卷制改造方案讨论与实施）

**用户需求**：主 memory 膨胀（3267行）导致上下文压力，讨论并实施分卷制改造方案。

**完成任务**：
1. ✅ 确定分卷参数：主 memory 200行/卷，子 memory 300行/卷
2. ✅ 确定命名方案：活跃卷始终 memory.md，归档为 memory_0/1/2...
3. ✅ memory-update-check.kiro.hook 更新为 v9.0（分卷检查 + 全对话记录 + 主memory摘要3-5行限制）
4. ✅ rules.md "工作区记忆"部分重写（融入 workspace-memory 关键规则：分卷制、更新规则、禁令）
5. ✅ 000-context-recovery.md 加入分卷感知规则
6. ✅ 农田系统主 memory 归档（3267行 → memory_0.md，新建精简 memory.md）

**未完成**：
- ❌ 10.1.4补丁004 子 memory 归档（1774行，待归档）
- ❌ 其他超限子 memory 检查

**关键决策**：workspace-memory.md 降级为参考文档，关键规则融入 rules.md 常驻；所有对话都必须记录到 memory。

**修改文件**：memory-update-check.kiro.hook、rules.md、000-context-recovery.md、memory_0.md（归档）、memory.md（重建）

### 元讨论续1 - 2026-02-25（10.1.4补丁004 子 memory 归档）

**完成任务**：10.1.4补丁004 子 memory 归档（1774行 → memory_0.md + 新建82行精简 memory.md）。

**发现**：其他超限子 memory（10.1.3补丁003: 920行、10.1.0全面持久化前夕: 763行、9.0.5智能交互bug修复: 555行、10.0.1农作物设计与完善: 507行、10.1.2补丁002: 447行），用户未明确要求处理。

**修改文件**：10.1.4补丁004/memory_0.md（归档）、10.1.4补丁004/memory.md（重建）

### 会话2续55 - 2026-02-25（锐评002审核 + 两件套完备性检查）

**完成**：锐评002事实核查（layerIndex断层⚠️部分正确但高估）、10.1.4遗留分析（不影响005）、两件套完备性检查通过并补充 layerIndex 获取方式。

**修改文件**：design.md、tasks.md（补充 layerIndex 获取明确化）

### 会话3续1~续2 - 2026-02-25（补丁005一条龙执行完成）

**完成**：补丁005全部代码任务完成。任务A修复房子不标红（FarmToolPreview.UpdateHoePreview条件修正）；任务B种子从FIFO剥离并接入放置系统（SeedData标记isPlaceable、PlacementValidator/Manager/Preview种子分支、GameInputManager+FarmToolPreview FIFO清理）。编译通过（0错误2警告，均为已有）。

**修改文件**：SeedData.cs、PlacementManager.cs、PlacementValidator.cs、PlacementPreview.cs、FarmToolPreview.cs、GameInputManager.cs

**状态**：✅ 代码完成，待手动验收（验收指南已创建）

### 会话3续8 - 2026-02-28（补丁005验收反馈：种子预览+耕地预览方向纠正）

**内容**：用户批评续4~续7方向错误。种子预览应保留方框（和箱子一致），hideCellVisuals 需回滚；耕地不可交互未耕地应显示红色1+8 tile预览（shader红色叠加）而非只有红方框。方案已确认待执行。
**待定**：障碍物检测标签白名单（稻草人/洒水器豁免）、树苗不能种在耕地上。

### 会话4续1~续2 - 2026-02-28（补丁005验收修复执行完成）

**完成**：PlacementPreview.cs hideCellVisuals 回滚彻底完成（UpdateGridCellPositions 残留清理）；FarmToolPreview.cs UpdateHoePreview else 分支拆分为"未耕地不可交互→红色1+8预览+shader红色叠加"和"已耕地不可交互→红方框"。编译通过。
**修改文件**：PlacementPreview.cs、FarmToolPreview.cs
