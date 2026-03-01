# 实施计划：批量生成物品工具 Sprite 选择优化

## 概述

基于已审核的需求和设计文档，将 `Tool_BatchItemSOGenerator` 的 Sprite 选择区域从单按钮+截断列表升级为双按钮+批量选择弹窗+拖拽排序+右键菜单的完整交互体验。

## 任务

- [x] 1. 创建 SpriteBatchSelectWindow 批量选择弹窗
  - [x] 1.1 创建 `Assets/Editor/SpriteBatchSelectWindow.cs`，实现独立 EditorWindow
    - 静态 `ShowWindow(Action<List<Sprite>> callback)` 方法
    - 从 `Selection.objects` 加载 Sprite（支持 Texture2D 和文件夹）
    - 网格布局展示 Sprite（勾选框 + 图标 + 名称）
    - 搜索框筛选、全选/取消全选按钮
    - 底部「取消」和「添加(N)」按钮
    - 未选中有效资源时显示提示信息
    - _需求: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7_

- [x] 2. 重写 DrawSpriteSelection 方法和新增拖拽排序
  - [x] 2.1 重写 `Tool_BatchItemSOGenerator.DrawSpriteSelection()` 方法
    - 替换原有单按钮为「获取选中项」+「批量选择」两个并排按钮
    - 「获取选中项」保留原有 `GetSelectedSprites()` 逻辑
    - 「批量选择」调用 `SpriteBatchSelectWindow.ShowWindow`
    - _需求: 1.1, 1.2, 1.3_
  - [x] 2.2 实现已选列表完整展示（ScrollView，最大高度 400px）
    - 每项显示：拖拽手柄 ≡ + 序号 + Sprite 图标 + 名称 + 预测 ID
    - 移除截断逻辑（不再显示"还有 XX 项"）
    - _需求: 3.1, 3.2, 3.3_
  - [x] 2.3 实现拖拽排序功能
    - 新增拖拽状态字段（`_isDraggingSprite`, `_dragSourceIndex`, `_dragTargetIndex`）
    - 在手柄区域处理 MouseDown/MouseDrag/MouseUp 事件
    - 拖拽目标行高亮显示
    - 拖拽完成后执行 `List.RemoveAt` + `List.Insert`
    - 参考 `InventoryBootstrapEditor.HandleItemDrag` 模式
    - _需求: 4.1, 4.2_
  - [x] 2.4 实现右键菜单功能
    - 使用 `GenericMenu` 构建上下文菜单
    - 菜单项：上移、下移、移到顶部、移到底部、删除
    - 边界条件处理（首项禁用上移，末项禁用下移）
    - 操作后重新计算预测 ID
    - _需求: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

- [x] 3. 检查点 - 确保编译通过
  - 确保所有代码编译通过，请用户验证功能。

- [x] 4. 数据兼容性验证
  - [x] 4.1 确认 `selectedSprites` 数据类型保持 `List<Sprite>` 不变
    - 确认生成逻辑按列表索引分配 ID
    - 确认拖拽和右键操作直接操作 `selectedSprites` 列表
    - _需求: 6.1, 6.2, 6.3_

- [x] 5. 最终检查点 - 确保所有功能完整
  - 确保所有代码编译通过，请用户验证功能。

## 说明

- 所有任务都是必须完成的
- 每个任务关联到具体需求以确保可追溯性
- 检查点用于增量验证
