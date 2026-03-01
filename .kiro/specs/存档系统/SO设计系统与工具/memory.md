# SO设计系统存档相关 - 存档修改记录

## 来源工作区
`.kiro/specs/SO设计系统与工具/`（会话 8）

## 存档相关修改摘要

### 核心贡献
- 创建 PersistentManagers 系统（DontDestroyOnLoad 统一管理）
- 修复 TimeManager/SeasonManager/WeatherSystem 的 DontDestroyOnLoad 警告

### PersistentManagers 架构

```
PersistentManagers（根物体，DontDestroyOnLoad）
  ├── TimeManager
  ├── SeasonManager
  └── WeatherSystem
```

- 单例模式，Awake 中调用 DontDestroyOnLoad
- 三个管理器从各自的 Awake 中移除 DontDestroyOnLoad 调用
- 由 PersistentManagers 统一处理持久化

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `PersistentManagers.cs` | 新建 | 统一管理持久化 |
| `TimeManager.cs` | 修改 | 移除 DontDestroyOnLoad |
| `SeasonManager.cs` | 修改 | 移除 DontDestroyOnLoad |
| `WeatherSystem.cs` | 修改 | 移除 DontDestroyOnLoad |

### 关键决策
- DontDestroyOnLoad 只能用于根物体，三个管理器作为子物体时会警告
- 集中管理比分散管理更可靠，避免重复调用和遗漏

### 详细记录
完整会话记录见：`.kiro/specs/SO设计系统与工具/memory.md`（会话 8）
