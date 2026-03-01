# 3.7.3 消失bug - 存档修改记录

## 来源工作区
`.kiro/specs/999_全面重构_26.01.27/3.7.3消失bug/`

## 存档相关修改摘要

### 核心贡献
- 修复 GUID 漂移问题（对象消失的根因）
- 创建 PersistentIdAutomator（编辑器自动修复空/重复 GUID）
- 创建 PersistentIdValidator（手动验证工具）

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `PersistentIdAutomator.cs` | 新建 | 场景保存时自动修复空/重复 GUID |
| `PersistentIdValidator.cs` | 新建 | 手动验证 GUID 完整性 |
| `SaveDataDTOs.cs` | 修改 | 数据结构调整 |
| `DynamicObjectFactory.cs` | 修改 | 重建逻辑修复 |

### 关键决策
- GUID 由 PersistentIdAutomator 在编辑器保存场景时自动分配
- 禁止在 Awake()/Start() 中生成新 GUID 覆盖已有值
- 禁止使用 GetInstanceID() 作为持久化标识

### 详细记录
完整会话记录见：`.kiro/specs/999_全面重构_26.01.27/3.7.3消失bug/memory.md`
