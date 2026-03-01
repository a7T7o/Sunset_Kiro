# 批量 SO 生成器分类优化 - 开发记忆

## 模块概述

将 `Tool_BatchItemSOGenerator` 的物品类型选择从扁平按钮布局改为大类+小类的层级结构（V2 版本），支持更多物品类型并提升用户体验。

## 当前状态

- **完成度**: 100% ✅
- **最后更新**: 2026-01-04
- **状态**: ✅ 已完成

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（大类分类系统、小类动态显示） |
| `design.md` | 设计文档 |
| `tasks.md` | 任务列表 |

## 完成内容

1. ✅ 添加 ItemMainCategory 大类枚举（6 个大类：工具装备/种植类/可放置/消耗品/材料/其他）
2. ✅ 扩展 ItemSOType 枚举（15 种小类）
3. ✅ 使用静态 Dictionary 实现大类→小类映射
4. ✅ UI 采用两行按钮布局：大类 + 小类
5. ✅ 切换大类时自动选中第一个小类并更新 ID/路径
6. ✅ 新增 4 种可放置物品类型（WorkstationData/StorageData/InteractiveDisplayData/SimpleEventData）

## 涉及的代码文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `Tool_BatchItemSOGenerator.cs` | 重构 | V2 版本，大类+小类层级结构 |
