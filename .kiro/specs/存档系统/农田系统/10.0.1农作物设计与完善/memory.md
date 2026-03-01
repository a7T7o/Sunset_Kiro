# 农作物存档集成 - 存档修改记录

## 来源工作区
`.kiro/specs/农田系统/10.0.1农作物设计与完善/`

## 存档相关修改摘要

### 核心贡献
- CropSaveData 数据结构扩展
- 作物状态存档（CropState 4状态：Growing/Mature/Withered/WitheredImmature）
- 种子袋保质期系统的存档影响

### 存档数据结构
- `CropSaveData`：作物ID、生长天数、成熟后天数、当前状态、是否浇水
- 嵌套在 `WorldObjectSaveData` 中
- CropController 实现 Save/Load 接口

### 修改的存档文件
| 文件 | 操作 | 说明 |
|------|------|------|
| `SaveDataDTOs.cs` | 修改 | CropSaveData 扩展（daysSinceMature、cropState） |
| `CropController.cs` | 修改 | Save/Load 实现，CropState 存档 |

### 存档相关的设计决策
- CropController 实现 IInteractable（右键收获），交互检测用 BoxCollider2D Trigger
- 作物存档需要记录 daysSinceMature（成熟后天数），用于枯萎判断
- 种子袋保质期不影响存档结构（保质期在 InventoryItem 层面处理）

### 待完成
- [ ] 作物存档的完整集成测试验收
- [ ] 枯萎作物的存档/读档流程验证

### 详细记录
完整会话记录见：`.kiro/specs/农田系统/10.0.1农作物设计与完善/memory.md`
