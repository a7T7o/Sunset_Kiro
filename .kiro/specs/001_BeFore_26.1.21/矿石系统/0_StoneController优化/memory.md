# StoneController 优化 - 开发记忆

## 模块概述

优化 StoneController 的编辑器界面和 Sprite 管理系统，使其与 TreeControllerV2 保持一致的交互体验。支持动态参数调节和自动 Sprite 加载。

核心内容：
- 4 种石头阶段（M1/M2/M3/M4），各有不同血量和石料总量
- 3 种矿物类型（C1 铜矿/C2 铁矿/C3 金矿）+ None（纯石头）
- OreIndex 矿物含量指数，对应不同 Sprite 变体
- Inspector 界面与 TreeControllerV2 保持一致

## 当前状态

- **完成度**: 100% ✅
- **最后更新**: 2026-01-21 之前
- **状态**: ✅ 已完成

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（编辑器界面优化、Sprite 自动加载） |
| `design.md` | 设计文档 |
| `tasks.md` | 任务列表 |

## 涉及的代码文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `StoneController.cs` | 修改 | 编辑器界面优化、Sprite 管理 |
