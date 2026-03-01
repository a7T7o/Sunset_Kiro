# ID 规范和管理器修复 - 开发记忆

## 模块概述

解决两个紧急问题：
1. 完善 SO ID 分配规范，明确钥匙/锁/树苗/建筑材料的 ID 范围
2. 修复 TimeManager、SeasonManager、WeatherSystem 的 DontDestroyOnLoad 警告（创建 PersistentManagers 统一管理）

## 当前状态

- **完成度**: 100% ✅
- **最后更新**: 2026-01-08
- **状态**: ✅ 已完成

## 包含文件

| 文件 | 说明 |
|------|------|
| `requirements.md` | 需求文档（5 个需求，15 个验收标准） |
| `design.md` | 设计文档（PersistentManagers 方案、ID 迁移方案） |
| `tasks.md` | 任务列表（7 个主任务，21 个子任务） |
| `验收指南.md` | 验收测试步骤和问题排查指南 |

## 完成内容

1. ✅ 创建 `PersistentManagers.cs` — 统一管理 DontDestroyOnLoad
2. ✅ 修改 TimeManager/SeasonManager/WeatherSystem — 移除各自的 DontDestroyOnLoad
3. ✅ 创建 `Tool_FixKeyIDs.cs` — 钥匙 ID 自动修复工具
4. ✅ 更新 SO 参数设计文档 — 添加 12XX/13XX/14XX 范围
5. ✅ 更新 `so-design.md` steering 规则
6. ✅ 执行钥匙 ID 修复（1410→1420, 1411→1421）

## 涉及的代码文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `PersistentManagers.cs` | 新增 | 统一持久化管理器 |
| `TimeManager.cs` | 修改 | 移除 DontDestroyOnLoad |
| `SeasonManager.cs` | 修改 | 移除 DontDestroyOnLoad |
| `WeatherSystem.cs` | 修改 | 移除 DontDestroyOnLoad |
| `Tool_FixKeyIDs.cs` | 新增 | 钥匙 ID 修复工具 |
| `so-design.md` | 修改 | 更新 ID 分配规范 |

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 创建 PersistentManagers 统一管理 | 集中管理持久化，避免各管理器单独调用 | 2026-01-08 |
| 钥匙 ID 范围 1420-1499 | 与 KeyLockData 中已有的验证逻辑一致 | 2026-01-08 |
