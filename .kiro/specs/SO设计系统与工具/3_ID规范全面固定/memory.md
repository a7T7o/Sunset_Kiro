# 3_ID规范全面固定 - 子工作区记忆

## 模块概述

本子工作区负责建立物品 ID 分配的唯一权威规范，解决多处文档不一致的混乱局面。

## 当前状态

- **完成度**: 100%
- **最后更新**: 2026-02-15
- **状态**: 已完成

## 权威文档位置

`ID分配规范.md` 已移至上级目录：`.kiro/specs/SO设计系统与工具/ID分配规范.md`

---

## 会话 13 - 2026-02-15（ID 分配权威规范建立）

**用户需求**:
> 1. 全面搜查所有 ID 相关文档，确保工具中的 SubTypeStartIDs 是事实标准
> 2. 解决 WitheredCropData 和 SaplingData 的 StartID 冲突（都是 1200）
> 3. 建立唯一权威 ID 规范文档
> 4. WitheredCrop 改名为 Crop_Withered（文件前缀），与 Crop 在文件夹中贴着排列
> 5. 工具先不动，只更新 ID 规范文档

**用户决策**:
- 枯萎作物和正常作物共享 11XX 段（100 个 ID 对半分）
- 1100-1149：正常作物（CropData）
- 1150-1199：枯萎作物（WitheredCropData）
- 文件前缀从 `WCrop` 改为 `Crop_Withered`
- SaplingData 独占 1200 段，冲突解除

**完成任务**:
1. ✅ 全面搜查：工具代码、所有 SO 资产、5 份文档的 ID 分配表对比
2. ✅ 创建权威 ID 规范文档（含完整 ID 总览、工具映射表、文件命名规范、变更审计日志）
3. ✅ 更新 `so-design.md` — ID 分配部分改为简要摘要 + 引用权威文档链接
4. ✅ 更新 `items.md` — 引用链改为指向权威文档，同步 ID 摘要
5. ✅ 扫描实际 SO 资产，新增「已使用 ID 登记表」

**修改文件**:
- `ID分配规范.md` - 新建（后移至上级目录）
- `so-design.md` - ID 分配改为摘要+引用
- `items.md` - 引用链指向权威文档

**关键决策**:

| 决策 | 原因 | 日期 |
|------|------|------|
| Crop 和 WitheredCrop 共享 11XX 段 | 20 种作物绰绰有余，50+50 的划分合理 | 2026-02-15 |
| 文件前缀 Crop_Withered | 文件夹排序时与 Crop_ 贴在一起 | 2026-02-15 |
| 建立权威 ID 规范文档 | 结束多处文档不一致的混乱局面 | 2026-02-15 |
| so-design.md 改为引用模式 | 避免多处维护，单一真相源 | 2026-02-15 |

---

## 会话 14 - 2026-02-15（ID 规范深度审查 + 修复执行）

**用户需求**:
> ID 规范文档不够权威，装备有子分类但规范只写了笼统的"8XXX: 装备"。要求完全读懂工具代码，彻底搞清所有 ID 细分。

**完成任务**:
1. ✅ 完整阅读 `Tool_BatchItemSOGenerator.cs` 全部代码
2. ✅ 阅读 `EquipmentData.cs`、`ItemEnums.cs`、`PlacementEnums.cs` 确认枚举定义
3. ✅ 扫描实际 SO 资产验证
4. ✅ 识别出 7 个问题

**发现的 7 个问题**:

| # | 问题 | 严重程度 |
|---|------|---------|
| 1 | 装备子分类完全遗漏（6 个子类型推荐 ID 范围） | 文档缺失 |
| 2 | 材料子分类标注为"预留"但实际已有资产 | 文档不准确 |
| 3 | 锁的 ID 细分没有记录（木锁/铁锁/特殊锁） | 文档缺失 |
| 4 | 钥匙材质体系没有记录（6 种材质+默认概率） | 文档缺失 |
| 5 | WitheredCropData StartID 仍为 1200（代码未同步） | 代码 bug |
| 6 | GetFilePrefix 中 WitheredCropData 仍为 "WCrop" | 代码 bug |
| 7 | GetFilePrefix 中缺少 LockData 的 case | 代码 bug |

**修复执行**（会话 14 续）:

用户确认分析后要求全部执行，以下修复已完成：

1. ✅ 重写 `ID分配规范.md` — 补充装备 6 个子分类、材料 4 个子分类、锁细分、钥匙材质体系
2. ✅ 修复 `Tool_BatchItemSOGenerator.cs` 3 处代码：
   - `SubTypeStartIDs` 中 WitheredCropData: 1200 → 1150
   - `GetFilePrefix` 中 WitheredCropData: "WCrop" → "Crop_Withered"
   - `GetFilePrefix` 中新增 `LockData => "Lock"`
3. ✅ 更新 `so-design.md` 引用路径（从子目录改为新路径）
4. ✅ 更新 `items.md` 引用路径
5. ✅ 创建本子工作区 memory（迁移会话 13/14 详细内容）
6. ✅ 精简主 memory（改为引用子工作区）

**修改文件**:
- `Tool_BatchItemSOGenerator.cs` - 修复 3 处代码（StartID/前缀/LockData case）
- `ID分配规范.md` - 重写，补充装备/材料/锁/钥匙子分类
- `so-design.md` - 更新引用路径
- `items.md` - 更新引用路径
- `3_ID规范全面固定/memory.md` - 新建子工作区 memory
- `memory.md` - 精简会话 13/14，改为引用

**编译状态**: ✅ Unity 编译通过，0 错误 0 警告
