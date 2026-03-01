# 3.7.5 自动化存档 - 存档修改记录

## 来源工作区
`.kiro/specs/999_全面重构_26.01.27/3.7.5自动化存档/`

## 存档相关修改摘要

### 核心贡献
- PrefabDatabase 自动扫描系统（替代旧 PrefabRegistry）
- ID 别名映射机制（处理预制体重命名）
- 渲染层级存档（sortingLayerName + sortingOrder）
- PrefabDatabaseAutoScanner 编辑器工具

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `PrefabDatabase.cs` | 新建/修改 | 自动扫描 + ID 别名映射 |
| `PrefabDatabaseAutoScanner.cs` | 新建 | 编辑器自动扫描工具 |
| `DynamicObjectFactory.cs` | 修改 | 使用 PrefabDatabase 替代 PrefabRegistry |
| `SaveDataDTOs.cs` | 修改 | 新增 sortingLayerName/sortingOrder 字段 |
| `SaveManager.cs` | 修改 | 存档/读档时处理渲染层级 |

### 关键决策
- PrefabDatabase 使用 ScriptableObject 存储，编辑器自动扫描维护
- ID 别名映射解决预制体重命名后旧存档无法加载的问题
- 渲染层级存档确保读档后物体在正确的 Sorting Layer 上

### 详细记录
完整会话记录见：`.kiro/specs/999_全面重构_26.01.27/3.7.5自动化存档/memory.md`
