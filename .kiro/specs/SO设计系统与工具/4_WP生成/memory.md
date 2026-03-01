# 4_WP生成 - 子工作区记忆

## 模块概述

WorldPrefab 生成器优化，使输出目录结构镜像 Items 文件夹结构。

## 当前状态

- **完成度**: 进行中
- **最后更新**: 2026-02-15
- **父工作区**: `.kiro/specs/SO设计系统与工具/`

## 会话记录

### 会话 1 - 2026-02-15（路径镜像优化）

**用户需求**: 优化 WP 生成器，输出路径镜像 Items 文件夹结构，支持全量重建

**完成任务**: 见父工作区 memory 会话 16

**修改文件**:
| 文件 | 操作 | 说明 |
|------|------|------|
| `WorldPrefabGeneratorTool.cs` | 修改 | 新增路径镜像逻辑 |


---

### 会话 1（续） - 2026-02-15（修复枯萎作物 WP 命名重复 ID）

**用户需求**:
> Crop_Withered_1150_大蒜 生成的 WP 变成了 WorldItem_1150_1150_大蒜，ID 重复了

**问题分析**:
`ExtractNameFromAsset()` 假设文件名格式为 `{Type}_{ID}_{Name}`（两段前缀），但枯萎作物是 `{Type}_{SubType}_{ID}_{Name}`（三段前缀 Crop_Withered_1150_大蒜），跳过前两部分后取到的是 `1150_大蒜` 而非 `大蒜`。

**完成任务**:
1. ✅ 重写 `ExtractNameFromAsset()` — 优先使用 `itemData.itemName`，回退时基于 ID 定位而非固定跳过前两段

**修改文件**:
| 文件 | 操作 | 说明 |
|------|------|------|
| `WorldPrefabGeneratorTool.cs` | 修改 | 重写 ExtractNameFromAsset，修复多前缀命名 bug |

**验证结果**（继承对话后验证）:
- ✅ Unity 编译通过，0 错误 0 警告
- ✅ getDiagnostics 无诊断问题
- ✅ 逻辑验证：优先使用 `itemData.itemName`，枯萎作物 `Crop_Withered_1150_大蒜` → `WorldItem_1150_大蒜`（正确）
- ✅ 回退逻辑：基于 `_ID_` 位置定位，兼容所有前缀格式
