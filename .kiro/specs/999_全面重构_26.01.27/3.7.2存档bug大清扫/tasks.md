# 动态对象重建系统 - 任务列表

## 任务概览

- **总任务数**: 12
- **预计工时**: 4-5 小时
- **优先级**: P0（用户直接痛点）

---

## 任务列表

### 阶段 1: 数据结构准备

- [x] 1. 扩展 TreeSaveData 数据结构
  - 在 `SaveDataDTOs.cs` 中为 `TreeSaveData` 添加字段：
    - `season` (int): 当前季节
    - `isStump` (bool): 是否为树桩
    - `stumpHealth` (int): 树桩血量
    - `hasTransitionedToNextSeason` (bool): 是否已渐变到下一季节
    - `transitionVegetationSeason` (int): 渐变时的植被季节
  - 文件: `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs`

- [x] 2. 新增 DropDataDTO 数据结构（🛡️ 封印一）
  - 在 `SaveDataDTOs.cs` 中新增 `DropDataDTO` 类
  - **必须**加上 `[Serializable]` 特性
  - 包含字段：`itemId`, `quality`, `amount`
  - 文件: `Assets/YYY_Scripts/Data/Core/SaveDataDTOs.cs`

### 阶段 2: 预制体注册机制

- [x] 3. 创建 PrefabRegistry ScriptableObject
  - 创建 `PrefabRegistry.cs`，包含：
    - `PrefabEntry` 内部类（prefabId + prefab）
    - `GetPrefab(string prefabId)` 方法
    - 字典缓存优化
  - 文件: `Assets/YYY_Scripts/Data/Core/PrefabRegistry.cs`

- [x] 4. 创建 PrefabRegistry 资产并配置
  - 在 `Assets/111_Data/Database/` 下创建 `PrefabRegistry.asset`
  - 配置树苗预制体映射（M1, M2, M3）
  - 确保所有可动态生成的预制体都已注册

### 阶段 3: 动态对象工厂

- [x] 5. 创建 DynamicObjectFactory（🛡️ 封印二）
  - 创建 `DynamicObjectFactory.cs`，包含：
    - `Initialize(PrefabRegistry)` 静态方法
    - `TryReconstruct(WorldObjectSaveData)` 静态方法
    - `TryReconstructTree()` - 树木重建（含数据验证）
    - `TryReconstructDrop()` - 掉落物重建
    - **数据有效性检查**：`health <= 0 && !isStump` → 跳过
    - **Legacy Fallback**：prefabId 为空时使用 M1
  - 文件: `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs`

### 阶段 4: TreeController 修改

- [x] 6. 修改 TreeController.Save() 方法
  - 添加 `GetPrefabId()` 私有方法
  - 在 Save() 中设置 `data.prefabId`
  - 扩展 TreeSaveData 序列化（season, isStump, stumpHealth, hasTransitionedToNextSeason, transitionVegetationSeason）
  - 文件: `Assets/YYY_Scripts/Controller/TreeController.cs`

- [x] 7. 添加 TreeController.SetPersistentIdForLoad() 方法
  - 新增公开方法，允许外部设置 persistentId
  - 仅供 DynamicObjectFactory 调用
  - 文件: `Assets/YYY_Scripts/Controller/TreeController.cs`

- [x] 8. 修改 TreeController.Load() 方法（🛡️ 封印三）
  - 恢复季节渐变状态（hasTransitionedToNextSeason, transitionVegetationSeason）
  - **UpdateVisuals() 必须是 Load() 的最后一行**
  - 文件: `Assets/YYY_Scripts/Controller/TreeController.cs`

### 阶段 5: 掉落物持久化

- [x] 9. 让 WorldItemPickup 实现 IPersistentObject
  - 实现 `Save()` 方法：返回 WorldObjectSaveData，genericData 存储 DropDataDTO
  - 实现 `Load()` 方法：从 genericData 恢复数据
  - 添加 `SetPersistentIdForLoad()` 方法
  - 文件: `Assets/YYY_Scripts/World/WorldItemPickup.cs`

### 阶段 6: 核心加载逻辑

- [x] 10. 修改 PersistentObjectRegistry.RestoreAllFromSaveData()
  - 在找不到 GUID 时调用 `DynamicObjectFactory.TryReconstruct()`
  - 重建后调用 `Load()`，然后 `SetActive(true)`（防闪烁）
  - 添加重建计数统计
  - 更新日志输出
  - 文件: `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs`

### 阶段 7: 初始化与测试

- [x] 11. 集成初始化流程
  - 在 GameManager 或 Bootstrap 中初始化 DynamicObjectFactory
  - 加载 PrefabRegistry 资产
  - 文件: 待确定（可能是 SaveManager 或 GameManager）

- [x] 12. 验证完整流程
  - 种植树苗 → 保存 → 重启 → 加载 → 树苗恢复
  - 掉落物品 → 保存 → 重启 → 加载 → 物品恢复
  - 旧存档兼容性测试

---

## 验收标准

### 功能验收

- [ ] 种植树苗 → 保存 → 重启游戏 → 加载 → 树苗出现在正确位置
- [ ] 树苗的生长阶段正确恢复
- [ ] 树苗的血量正确恢复
- [ ] 树苗的视觉外观正确恢复（无闪烁）
- [ ] 掉落物 → 保存 → 重启游戏 → 加载 → 物品出现在正确位置
- [ ] 掉落物的 itemId、quality、amount 正确恢复
- [ ] 控制台显示"重建 N 个对象"的日志

### 兼容性验收

- [ ] 旧存档（prefabId 为空）不会导致加载失败
- [ ] 旧存档使用 M1 作为默认预制体进行抢救性重建
- [ ] 静态对象（石头）的加载不受影响
- [ ] 运行中保存/加载仍然正常工作

### 性能验收

- [ ] 加载 20+ 个动态对象的时间 < 1 秒

### 🛡️ 质量封印验收

- [ ] **封印一**：DropDataDTO 有 `[Serializable]` 特性
- [ ] **封印二**：DynamicObjectFactory 在 Instantiate 前验证数据有效性
- [ ] **封印三**：TreeController.Load() 最后一行是 UpdateVisuals()

---

## 依赖关系

```
任务 1 (TreeSaveData) ─┐
任务 2 (DropDataDTO) ──┼─→ 任务 6 (TreeController.Save)
                       │
任务 3 (PrefabRegistry) ─┤
                         ├─→ 任务 5 (DynamicObjectFactory) ─→ 任务 10 (RestoreAll)
任务 4 (配置资产) ───────┘                                          │
                                                                    ▼
任务 7 (SetPersistentId) ─────────────────────────────────→ 任务 11 (集成)
任务 8 (TreeController.Load) ─────────────────────────────→     │
任务 9 (WorldItemPickup) ─────────────────────────────────→     ▼
                                                           任务 12 (验证)
```

---

## 注意事项

1. **GUID 必须复用**：重建的对象必须使用存档中的 GUID，否则下次保存会生成新 GUID
2. **预制体路径稳定性**：prefabId 使用 M1/M2/M3，不依赖文件路径
3. **向后兼容**：旧存档的 prefabId 为空时，使用 M1 作为默认预制体
4. **场景层级**：重建的对象应放置在正确的场景层级下（LAYER 1/Props 等）
5. **防闪烁**：实例化时先 SetActive(false)，Load 后再 SetActive(true)
6. **数据验证**：health <= 0 且非树桩的数据直接跳过，不生成幽灵树
