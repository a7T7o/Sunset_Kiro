# 10.0.2.1 纠正 - 开发记忆

## 模块概述
废弃 CropManager，全面对齐树木系统的 Prefab 驱动模式。包括播种链路重写、收获改掉落模式、SeedData 字段精简。

## 当前状态
- **完成度**: 100%
- **最后更新**: 2026-02-14
- **状态**: 已完成

## 会话记录

### 会话 3（续） - 2026-02-14（续接自会话 3，10.X 纠正一条龙执行）

**用户需求**:
> 10.0.2.X 和 10.0.3 一起并发工作，先全面废弃 CropManager，生成"10.X纠正设计"和"10.X纠正任务"文档，然后一条龙完成所有任务。

**完成任务**:
1. 读取所有关键代码文件（继承后重新读取）：CropController、CropManager、GameInputManager、SeedData、SeedBagHelper、DynamicObjectFactory、TreeController、FarmTileData、FarmToolPreview
2. 读取全面调查报告、999迭代需求001、10.0.3 requirements、farming.md、communication.md、workspace-memory.md、coding-standards.md
3. 创建 `10.X纠正设计.md` — 13 章完整设计文档
4. 创建 `10.X纠正任务.md` — 16 个任务
5. 执行全部 16 个任务（一条龙模式）：
   - 任务1: FarmTileData 新增 cropController 运行时引用
   - 任务2: CropController 新增 5 个掉落配置字段
   - 任务3: 重写 HarvestMature/HarvestWitheredMature 为掉落模式
   - 任务4: 扩展 DestroyCrop 清除 FarmTileData 引用 + 取消注册 PersistentObjectRegistry
   - 任务5: Initialize(带位置) 设置 FarmTileData.cropController
   - 任务6: Load() 恢复 FarmTileData.cropController
   - 任务7: ExecutePlantSeed 重写（绕过 CropManager，直接 Instantiate seedData.cropPrefab + 种子袋自动初始化）
   - 任务8: ExecuteTillSoil 替换 CropManager 引用为 tileData.cropController
   - 任务9: FarmToolPreview 替换 CropManager 引用
   - 任务10: TryHarvestCropAtMouse 标记 Obsolete
   - 任务11: SeedData 三个字段标记 Obsolete + OnValidate/GetTooltipText 适配
   - 任务12: CropManager 整个类标记 Obsolete
   - 任务13: Unity 编译验证 — 零错误，2 个 Obsolete 警告（Tool_BatchItemSOGenerator）
   - 任务14: CropSystemTests — 21/21 全部通过
   - 任务15: farming.md 规范更新
   - 任务16: memory.md 更新

**修改文件**:
- `FarmTileData.cs` - 新增 cropController 运行时引用
- `CropController.cs` - 新增掉落字段、重写收获逻辑、扩展 DestroyCrop、Initialize/Load 设置引用
- `GameInputManager.cs` - 播种链路重写（绕过 CropManager）、ExecuteTillSoil 替换引用、TryHarvestCropAtMouse 废弃
- `FarmToolPreview.cs` - 替换 CropManager.GetCrop 为 tileData.cropController
- `SeedData.cs` - growthDays/harvestCropID/harvestAmountRange 标记 Obsolete
- `CropManager.cs` - 整个类标记 Obsolete
- `farming.md` - 更新播种链路、收获机制、FarmTileData、SeedData、已知问题
- `10.X纠正设计.md` - 新建，13 章完整设计文档
- `10.X纠正任务.md` - 新建，16 个任务
- `memory.md` - 新建，本工作区记忆

**关键设计决策**:
- CropManager 整个类标记 Obsolete 而非删除文件，保证编译通过
- SeedData 旧字段标记 Obsolete 而非删除，保证存档兼容
- FarmTileData 新增 NonSerialized 的 cropController 引用替代 CropManager.GetCrop
- 收获从"直接加背包"改为"掉落到地面"（学习 TreeController.SpawnDrops）
- 播种时自动初始化未初始化的种子袋（修复 InventoryBootstrap 注入种子的 bug）
- DetermineHarvestQuality 随机品质废弃，改由 dropQuality 字段控制

**遗留问题**:
- [ ] Collect 动画集成（收获时播放弯腰拔起动画）— 预留后续迭代
- [ ] Tool_BatchItemSOGenerator 中 2 个 Obsolete 警告需要后续清理
- [ ] CropController Prefab 上需要配置 dropItemData/witheredDropItemData（Inspector 操作）

---

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 废弃 CropManager 而非删除 | 保证编译通过，渐进迁移 | 2026-02-14 |
| 收获改掉落模式 | 与树木系统统一，学习 TreeController.SpawnDrops | 2026-02-14 |
| FarmTileData.cropController 替代 activeCrops 字典 | 更高效，与数据模型一致 | 2026-02-14 |
| 种子袋自动初始化 | 修复 InventoryBootstrap 注入种子时未初始化的 bug | 2026-02-14 |

## 涉及的代码文件

### 核心文件（修改/新增）
| 文件 | 操作 | 说明 |
|------|------|------|
| `CropController.cs` | 修改 | 掉落字段、收获重写、DestroyCrop 扩展、Initialize/Load 设置引用 |
| `GameInputManager.cs` | 修改 | 播种链路重写、ExecuteTillSoil 替换、TryHarvestCropAtMouse 废弃 |
| `FarmTileData.cs` | 修改 | 新增 cropController 运行时引用 |
| `FarmToolPreview.cs` | 修改 | 替换 CropManager 引用 |
| `SeedData.cs` | 修改 | 3 个字段标记 Obsolete |
| `CropManager.cs` | 修改 | 整个类标记 Obsolete |

### 相关文件（引用/依赖）
| 文件 | 关系 |
|------|------|
| `TreeController.cs` | SpawnDrops 参考实现 |
| `DynamicObjectFactory.cs` | 存档重建（已使用 seedData.cropPrefab，不受影响） |
| `WorldSpawnService.cs` | SpawnMultiple 掉落服务 |
| `SeedBagHelper.cs` | 种子袋辅助（未修改） |
| `CropSystemTests.cs` | 测试（21/21 通过，未修改） |
| `farming.md` | 规范文档（已更新） |


### 会话 4 - 2026-02-15（needsWatering 迁移至 Controller）

**用户需求**:
> CropController 的"生长规则"区域新增 `needsWatering` 参数，从 SeedData 删除（标记废弃）该字段，统一由 Controller 管控

**完成任务**:
1. CropController.cs — 生长规则区域新增 `needsWatering` 字段（默认 true），位于 `daysUntilWithered` 之前
2. CropController.cs — `HandleGrowingDay()` 逻辑适配：`needsWatering=false` 时每天自动生长，不累计缺水天数
3. SeedData.cs — `needsWatering` 字段标记 `[System.Obsolete]`，保留字段兼容存档

**修改文件**:
- `CropController.cs` — 新增 needsWatering 字段 + HandleGrowingDay 逻辑适配
- `SeedData.cs` — needsWatering 标记 [System.Obsolete]

**编译状态**: ✅ 成功（0 错误 0 警告）

**关联代办**: TD_000_农田系统.md #5（Controller 与 SO 数据流完善）— needsWatering 已完成迁移
