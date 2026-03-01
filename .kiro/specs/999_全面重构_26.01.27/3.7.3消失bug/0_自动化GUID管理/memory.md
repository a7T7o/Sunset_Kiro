# 自动化 GUID 管理 - 开发记忆

## 模块概述

利用 Unity 的 `EditorSceneManager.sceneSaving` 事件，在保存场景时自动扫描并修复所有缺失 GUID 的持久化对象（`IPersistentObject`），彻底解放开发者的手动操作。支持空 GUID 修复、重复 GUID 检测（Ctrl+D 复制导致）、手动验证工具。

## 当前状态

- **完成度**: 100% ✅
- **最后更新**: 2026-02-02
- **状态**: ✅ 已完成

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（自动化 GUID 生成、重复检测、构建验证） |
| `design.md` | 设计文档（Editor 自动化守门员方案） |
| `tasks.md` | 任务列表 |
| `锐评001.md` | Code Reaper 锐评024（战术修剪：砍掉配置系统、简化构建验证器） |

## 完成内容

1. ✅ 创建 `PersistentIdAutomator.cs` — 核心自动化组件（监听场景保存事件）
2. ✅ 创建 `PersistentIdValidator.cs` — 手动验证工具（菜单命令）
3. ✅ 支持 `persistentId` 和 `_persistentId` 两种字段名
4. ✅ 空 GUID 自动修复 + 重复 GUID 检测

## 涉及的代码文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `PersistentIdAutomator.cs` | 新增 | 场景保存时自动修复 GUID |
| `PersistentIdValidator.cs` | 新增 | 手动验证/修复菜单工具 |

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 监听 sceneSaving 事件 | 无感、零操作、安全 | 2026-02-02 |
| 砍掉配置系统 | 核心机制必须 Always On，不需要配置 UI | 2026-02-02 |
| 简化构建验证器 | 降级为 Log Warning，不需要复杂阻断 | 2026-02-02 |
