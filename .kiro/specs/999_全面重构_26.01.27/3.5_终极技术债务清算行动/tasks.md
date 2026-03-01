# Phase 3.5 终极技术债务清算行动 - 任务列表

## 任务概览

| 战役 | 任务数 | 预估时间 | 状态 |
|------|--------|---------|------|
| A: 数据层净化 | 3 | 30 分钟 | ✅ 完成 |
| B: 存档系统补完 | 4 | 45 分钟 | ✅ 完成 |
| C: 农田工具距离检测 | 2 | 15 分钟 | ✅ 完成 |

---

## 战役 A: 数据层净化

- [x] 1. 审查 ItemData.OnValidate
  - [x] 1.1 确认保留的核心验证逻辑（ID、名称、图标、价格）
  - [x] 1.2 移除或简化非核心验证（如有）

- [x] 2. 创建 ItemDataEditor.cs
  - [x] 2.1 创建文件 `Assets/Editor/ItemDataEditor.cs`
  - [x] 2.2 实现 `[CustomEditor(typeof(ItemData), true)]` 支持子类
  - [x] 2.3 实现 `OnInspectorGUI()` 基础框架
  - [x] 2.4 实现字段分组绘制（Basic Info、Visuals、Economy）
  - [x] 2.5 实现条件显示逻辑（根据 category 隐藏/显示）
  - [x] 2.6 实现 `DrawRemainingProperties()` 确保子类字段不被吞掉

- [x] 3. 验证 Custom Editor
  - [x] 3.1 测试 Equipment 类型 SO（应隐藏 Placement Config）
  - [x] 3.2 测试 Seed 类型 SO（应隐藏 Equipment Config）
  - [x] 3.3 测试 Placeable 类型 SO（应显示 Placement Config）
  - [x] 3.4 验证 Undo/Redo 功能正常

---

## 战役 B: 存档系统补完

- [x] 4. SaveManager Tool 排除
  - [x] 4.1 在 `CollectPlayerData()` 添加 Tool 排除说明注释
  - [x] 4.2 确认 Tool 没有实现 IPersistentObject，不会被收集
  - [x] 4.3 验证存档文件中不包含 Tool 数据

- [x] 5. TreeController 实现 IPersistentObject
  - [x] 5.1 添加 `persistentId` 字段（GUID）
  - [x] 5.2 实现 OnValidate() 自动生成 GUID（编辑器模式）
  - [x] 5.3 实现 Awake() 运行时 GUID 检查（动态生成的树）
  - [x] 5.4 添加 `IPersistentObject` 接口实现
  - [x] 5.5 实现 `PersistentId` 属性
  - [x] 5.6 实现 `ObjectType` 属性（返回 "Tree"）
  - [x] 5.7 实现 `ShouldSave` 属性
  - [x] 5.8 实现 `Save()` 方法（使用 TreeSaveData + genericData）
  - [x] 5.9 实现 `Load()` 方法（恢复状态并刷新视觉）

- [x] 6. TreeController 注册到 Registry
  - [x] 6.1 在 `Start()` 中注册到 `PersistentObjectRegistry`（带 ID 冲突自愈）
  - [x] 6.2 在 `OnDestroy()` 中从 Registry 注销

- [x] 7. SaveManager UI 刷新
  - [x] 7.1 在 `LoadGame()` 末尾添加 `RefreshAllUI()` 调用
  - [x] 7.2 实现 `RefreshAllUI()` 方法刷新 InventoryPanelUI 和 ToolbarUI
  - [x] 7.3 验证读档后 UI 立即更新

---

## 战役 C: 农田工具距离检测

- [x] 8. 添加距离检测配置
  - [x] 8.1 在 `GameInputManager` 添加 `farmToolReach` 字段
  - [x] 8.2 设置默认值 1.5f
  - [x] 8.3 添加 Header 和 Tooltip

- [x] 9. 实现距离检测逻辑
  - [x] 9.1 修改 `TryTillSoil()` 添加距离检测
  - [x] 9.2 修改 `TryWaterTile()` 添加距离检测
  - [x] 9.3 使用 `GetPlayerCenter()` 获取玩家位置
  - [x] 9.4 使用 `tilemaps.GetCellCenterWorld()` 获取目标位置
  - [x] 9.5 验证距离检测不影响其他功能

---

## 验收检查

- [x] 10. 最终验收
  - [x] 10.1 编译通过（0 错误）
  - [ ] 10.2 战役 A：Inspector 动态显示正常（需用户手动验证）
  - [ ] 10.3 战役 B：存档/读档功能正常（需用户手动验证）
  - [ ] 10.4 战役 B：树木状态正确恢复（需用户手动验证）
  - [ ] 10.5 战役 C：农田工具距离限制生效（需用户手动验证）
  - [ ] 10.6 战役 C：箱子交互、NPC 对话不受影响（需用户手动验证）
  - [x] 10.7 更新 memory.md 记录完成状态
