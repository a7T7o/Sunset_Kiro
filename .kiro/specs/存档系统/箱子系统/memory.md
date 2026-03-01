# 箱子系统存档集成 - 存档修改记录

## 来源工作区
`.kiro/specs/箱子系统/`

## 存档相关修改摘要

### 核心贡献
- 箱子存档数据结构（ChestSaveData）
- 箱子层级结构的存档处理（父物体 PersistentObject + 子物体 ChestController）
- Initialize/Load 顺序问题的发现和修复

### 箱子存档架构

```
箱子父物体（PlaceableObject + PersistentObject）
  └── Box 子物体（ChestController + SpriteRenderer + Collider）
```

- PersistentObject 在父物体上，负责 GUID 和注册/注销
- ChestController 在子物体上，负责箱子逻辑和存档数据
- 获取 ChestController 必须用 GetComponentInChildren，不能在父物体上 GetComponent

### 存档数据结构
- `ChestSaveData`：箱子类型、容量、物品列表
- 嵌套在 `WorldObjectSaveData` 中，通过 `objectType = "Chest"` 标识
- DynamicObjectFactory 中 Chest 类型走专门的重建路径

### 关键踩坑记录
1. **Initialize/Load 顺序**（3.7.6 修复）：
   - 读档时 Load() 先于 Start() 执行
   - Initialize() 中 `_inventory` 的 null 检查防止覆盖 Load() 数据
2. **箱子 IsEmpty 判断**（3.7.4 修复）：
   - 需要检查所有槽位而非只检查 Count

### 涉及的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `ChestController.cs` | 修改 | Save/Load 实现、Initialize null 检查 |
| `DynamicObjectFactory.cs` | 修改 | Chest 类型重建路径 |
| `SaveDataDTOs.cs` | 修改 | ChestSaveData 结构 |

### 详细记录
完整会话记录见：`.kiro/specs/箱子系统/memory.md`
