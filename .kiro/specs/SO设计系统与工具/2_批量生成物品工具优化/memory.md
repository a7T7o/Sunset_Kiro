# 批量生成物品工具优化 - 开发记忆

## 模块概述

优化 `Tool_BatchItemSOGenerator` 编辑器工具的 Sprite 选择区域交互体验，包括：
- 双按钮布局（获取选中项 + 批量选择弹窗）
- 已选列表完整展示（ScrollView，不截断）
- 拖拽排序（参考 InventoryBootstrapEditor 模式）
- 右键菜单（上移/下移/移到顶部/移到底部/删除）

## 当前状态

- **完成度**: 100% ✅
- **最后更新**: 2026-02-15
- **状态**: ✅ 已完成

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（6 个需求，Sprite 选择区域优化） |
| `design.md` | 设计文档（UI 布局、交互流程、组件设计） |
| `tasks.md` | 任务列表 |

## 完成内容

1. ✅ 创建 `SpriteBatchSelectWindow.cs` — Sprite 批量选择弹窗
2. ✅ 重写 `DrawSpriteSelection()` — 双按钮布局
3. ✅ 实现已选列表完整展示（ScrollView，最大 400px）
4. ✅ 实现拖拽排序（MouseDown/Drag/Up 事件处理）
5. ✅ 实现右键菜单（GenericMenu）
6. ✅ 数据兼容性验证通过

## 涉及的代码文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `SpriteBatchSelectWindow.cs` | 新增 | Sprite 批量选择弹窗 |
| `Tool_BatchItemSOGenerator.cs` | 修改 | 重写 DrawSpriteSelection，新增拖拽排序和右键菜单 |

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 独立 EditorWindow 做批量选择 | 避免主窗口过于复杂 | 2026-02-15 |
| 参考 InventoryBootstrapEditor 拖拽模式 | 项目内已有成熟实现 | 2026-02-15 |
