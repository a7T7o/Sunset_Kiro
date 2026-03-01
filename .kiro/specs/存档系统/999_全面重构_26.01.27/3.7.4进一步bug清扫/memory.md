# 3.7.4 进一步bug清扫 - 存档修改记录

## 来源工作区
`.kiro/specs/999_全面重构_26.01.27/3.7.4进一步bug清扫/`

## 存档相关修改摘要

### 核心贡献
- 实现反向修剪机制（PruneStaleRecords）
- 石头假死机制（SetActive(false) 而非 Destroy()）
- 箱子 IsEmpty 判断修复
- 掉落物与容器的关联存档

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `PersistentObjectRegistry.cs` | 修改 | 反向修剪（PruneStaleRecords 替代 Clear） |
| `StoneController.cs` | 修改 | 假死机制，SetActive(false) 存档 |
| `SaveManager.cs` | 修改 | 加载流程优化 |
| `SaveDataDTOs.cs` | 修改 | 数据结构扩展 |

### 关键决策
- 反向修剪：只移除"存档中没有但注册表中有"的对象，保留两者都有的
- 石头不销毁只隐藏：避免重建开销，读档时恢复 SetActive(true)
- 箱子 IsEmpty 需要检查所有槽位而非只检查 Count

### 详细记录
完整会话记录见：`.kiro/specs/999_全面重构_26.01.27/3.7.4进一步bug清扫/memory.md`
