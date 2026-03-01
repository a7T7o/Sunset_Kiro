# Phase 3.5 终极技术债务清算行动 - 需求文档

## 概述

基于与 Code Reaper 的技术辩论达成的最终共识，执行三个战役的技术债务清算。

---

## 用户故事

### US-1: 数据层净化（战役 A）

**作为** 策划人员  
**我希望** Inspector 面板根据物品类型动态显示相关字段  
**以便** 我不会被无关的配置项干扰（如装备不显示农田配置）

#### 验收标准

- AC-1.1: `ItemData.cs` 保留 `OnValidate()` 中的核心验证逻辑（ID 范围、名称非空、图标非空）
- AC-1.2: 创建 `ItemDataEditor.cs`，根据 `category` 动态隐藏/显示字段
- AC-1.3: 当 `category == Equipment` 时，隐藏 `Placement Config`、`Farm Config`
- AC-1.4: 当 `category == Seed` 时，隐藏 `Equipment Config`、`Placement Config`
- AC-1.5: 当 `category == Furniture/Placeable` 时，显示 `Placement Config`
- AC-1.6: 批量工具（如 `Tool_BatchItemSOGenerator`）修改 SO 时，`OnValidate` 仍能触发验证

---

### US-2: 存档系统补完（战役 B）

**作为** 玩家  
**我希望** 游戏能正确保存和恢复世界状态  
**以便** 我的游戏进度不会丢失

#### 验收标准

- AC-2.1: `SaveManager.SaveGame()` 排除玩家子物体中名为 "Tool" 的对象
- AC-2.2: `TreeController` 实现 `IPersistentObject` 接口
- AC-2.3: 树木存档数据包含：GUID、当前阶段、当前血量、当前季节、是否激活
- AC-2.4: 读档后树木视觉状态立即恢复（不等待下一帧）
- AC-2.5: `SaveManager.LoadGame()` 结束时强制刷新 UI（`InventoryPanelUI.Refresh()`）

---

### US-3: 农田工具距离检测（战役 C）

**作为** 玩家  
**我希望** 使用锄头和水壶时有合理的距离限制  
**以便** 我不能隔着很远的距离耕地或浇水

#### 验收标准

- AC-3.1: `TryTillSoil()` 添加距离检测，超出 `toolReach` 时返回 false
- AC-3.2: `TryWaterTile()` 添加距离检测，超出 `toolReach` 时返回 false
- AC-3.3: 距离计算基于玩家 Collider 中心到目标格子中心的平面距离
- AC-3.4: `toolReach` 可配置（默认值 1.5f）
- AC-3.5: 保留 `HandleRightClickAutoNav()` 中用于检测可交互物体的 Raycast 逻辑

---

## 非功能需求

### NFR-1: 代码质量
- 所有修改必须通过编译（0 错误）
- 保持现有功能不受影响（箱子交互、NPC 对话等）

### NFR-2: 性能
- 距离检测使用纯数学计算，不使用 Physics2D
- 不增加额外的 Update 调用

### NFR-3: 可维护性
- Custom Editor 代码与运行时代码分离
- 使用 `SerializedProperty` 支持 Undo/Redo

---

## 约束条件

1. **严禁删除** `HandleRightClickAutoNav()` 中的 Raycast 逻辑
2. **保留** `ItemData.OnValidate()` 的核心验证
3. **使用** `PersistentObjectRegistry` 管理持久化对象
4. **遵循** 项目现有的命名规范和代码风格
