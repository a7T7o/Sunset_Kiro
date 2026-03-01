# 3.7.2 存档bug大清扫 - 存档修改记录

## 来源工作区
`.kiro/specs/999_全面重构_26.01.27/3.7.2存档bug大清扫/`

## 存档相关修改摘要

### 核心贡献
- 建立了动态对象重建系统（DynamicObjectFactory）
- 创建了 PrefabRegistry（后被 PrefabDatabase 替代）
- 实现了 WorldObjectSaveData 的基础结构

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `DynamicObjectFactory.cs` | 新建/修改 | 动态对象重建（树木/石头/箱子/掉落物） |
| `PersistentObjectRegistry.cs` | 修改 | 对象注册中心基础实现 |
| `SaveDataDTOs.cs` | 修改 | WorldObjectSaveData 基础结构 |
| `SaveManager.cs` | 修改 | RestoreAllFromSaveData 流程 |

### 关键决策
- 动态对象通过 objectType 字段分发重建逻辑
- 未注册对象走 DynamicObjectFactory.TryReconstruct 路径
- 已注册对象直接调用 Load()

### 详细记录
完整会话记录见：`.kiro/specs/999_全面重构_26.01.27/3.7.2存档bug大清扫/memory.md`
