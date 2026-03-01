# 字段优化重构 - 开发记忆

## 模块概述

优化 ScriptableObject 系统的字段设计，解决基类 `ItemData` 中 `equipmentType` 和 `consumableType` 字段过于臃肿的问题。将这两个字段从基类移除，只在需要的子类（FoodData、PotionData）中添加，提升 Inspector 清晰度和代码可维护性。

## 当前状态

- **完成度**: 0%（仅完成文档规划，代码未执行）
- **最后更新**: 2026-01-07
- **状态**: ⏸️ 搁置（用户决定优先处理 ID 规范和管理器修复）

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（8 个需求，38 个验收标准） |
| `design.md` | 设计文档（架构对比、组件设计、正确性属性） |
| `tasks.md` | 任务列表（8 个主任务，35 个子任务） |

## 核心内容摘要

- 从 `ItemData` 基类移除 `equipmentType` 和 `consumableType`
- 在 FoodData、PotionData 中添加 `consumableType`
- 添加虚方法接口（`IsConsumable()`、`GetConsumableType()`）
- 更新批量生成/修改工具
- 创建字段清理工具
- 标注 `bagSprite` 为废弃字段

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 搁置执行，优先修复 ID 和管理器 | ID 警告和 DontDestroyOnLoad 更紧急 | 2026-01-07 |
| 采用虚方法模式 | 保持统一接口，避免直接访问字段 | 2026-01-07 |
