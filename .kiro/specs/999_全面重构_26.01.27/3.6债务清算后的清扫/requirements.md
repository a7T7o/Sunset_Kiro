# 存档系统严重问题修复 - 需求文档

## 概述

用户报告了多个存档系统的严重问题，需要全面修复以确保游戏状态能够正确保存和恢复。

---

## 问题清单

### P0-1: 玩家位置不同步

**现象**：
- 加载存档后，Tool 子物体停留在保存时的世界位置
- 玩家没有回到保存时的位置

**根本原因分析**：
- `SaveManager.RestorePlayerData()` 只设置了 `player.transform.position`
- 但 Tool 子物体可能有独立的世界位置缓存
- **用户澄清**：Tool 的 Transform 位置应始终保持 (0, 0, 0)，不需要同步任何内容

**🔥 辅助文档补充分析**：
- 现象本质：玩家没回去，说明 `transform.position = savePos` 失败了，或者被覆盖了
- 可能原因：Unity 执行顺序问题。如果 `SaveManager.Load()` 在 `PlayerController.Start()` 之前执行，或者 Player 身上有 `CharacterController` / `Rigidbody2D`，直接修改 `transform.position` 可能会无效（被物理引擎弹回）或被初始化逻辑重置
- Tool 的诡异表现：Tool 是玩家的子物体。如果 Tool 的 transform 没有被 SaveManager 触碰，它应该跟随玩家。现在它"留在原地"，说明 Tool 的 Transform 被意外写入了世界坐标，或者它的父子关系在加载瞬间断开了

**验收标准**：
- [ ] 加载存档后，玩家位置正确恢复到保存时的位置
- [ ] Tool 子物体的 localPosition 保持 (0, 0, 0)
- [ ] 不读取或修改 Tool 的任何数据
- [ ] 🔥 确保 Load 在 Player 初始化之后执行（或者在设置位置前暂时禁用物理组件）
- [ ] 🔥 Load 完玩家位置后，强制执行 `tool.transform.localPosition = Vector3.zero`

---

### P0-2: 箱子 IsEmpty 状态问题

**现象**：
- 加载存档后，箱子的 `IsEmpty` 状态始终为"有内容"
- 即使清空箱子内容也无法用镐子挖取（只能推动）

**根本原因分析**：
- `ChestController.IsEmpty` 属性依赖 `_inventoryV2` 和 `_inventory`
- 加载存档时可能只恢复了 `_inventoryV2`，但 `_inventory` 没有同步
- 或者 `SyncV2ToInventory()` 没有正确执行

**🔥 辅助文档补充分析**：
- 逻辑漏洞：`ChestController.IsEmpty` 判断逻辑是 `return _inventoryV2.IsEmpty && _inventory.IsEmpty`
- 死锁场景：读档时，`_inventoryV2`（数据层）恢复了数据。虽然你把东西拿走了（`_inventoryV2` 变空），但如果 UI 层的 `_inventory` 没有收到同步信号，它可能还认为自己是满的（或者 dirty flag 没清除）

**验收标准**：
- [ ] 加载存档后，空箱子的 `IsEmpty` 返回 true
- [ ] 空箱子可以被镐子挖取
- [ ] 有内容的箱子只能被推动
- [ ] 🔥 Load 完毕后，不仅要 `SyncV2ToInventory()`，还要强制调用一次 `CheckEmptyState()` 或类似逻辑，确保状态标志位复位

---

### P0-3: 🔥 Registry 生命周期问题（致命！）

**现象**：
- 读档后树木/石头状态无法恢复
- ID 冲突警告频繁出现

**🔥 辅助文档深度分析（这是真正的元凶！）**：
- 根因：`PersistentObjectRegistry` 是单例且 `DontDestroyOnLoad`。当你**重新加载场景（读取存档）**时，旧场景销毁了，但 Registry 里还存着旧对象的引用（或 GUID）
- 灾难流程：
  1. 新场景加载 → 树木 A (ID: Tree_01) 执行 `Start()` 注册
  2. Registry 说："Tree_01 已经存在了！"（其实是上个场景残留的 key）
  3. 树木 A 触发"自愈逻辑"，生成新 ID (ID: Tree_New_Random)
  4. SaveManager 开始读档：它拿着存档里的 Tree_01 去找对象
  5. 结果：Registry 里只有 Tree_New_Random。存档数据找不到对应的树，树木状态恢复失败！

**验收标准**：
- [ ] `PersistentObjectRegistry` 必须增加 `Clear()` 方法
- [ ] 在 `SaveManager.LoadGame()` 的最开始，或者场景加载事件中，必须强制清空 Registry
- [ ] 读档后不再出现 ID 冲突警告
- [ ] 树木/石头状态能够正确恢复

---

### P1-1: TreeController ID 冲突警告

**现象**：
- 控制台持续出现 TreeController 的 ID 冲突检测警告
- 警告信息：`[TreeController] xxx ID 冲突检测 (ID: xxx)，正在重新生成...`

**根本原因分析**：
- `RegisterToPersistentRegistry()` 使用 `TryRegister()` 检测冲突
- 如果场景中有 Ctrl+D 复制的树木，它们会有相同的 `persistentId`
- 每次启动游戏都会触发冲突检测和 ID 重新生成
- 🔥 **更深层原因**：见 P0-3，Registry 跨场景残留导致的假冲突

**验收标准**：
- [ ] 正常情况下不应出现 ID 冲突警告
- [ ] 如果确实有冲突，只在首次检测时输出警告，后续不再重复
- [ ] 考虑在编辑器模式下自动为复制的对象生成新 ID

---

### P1-2: 树木/石头状态不恢复

**现象**：
- 砍伐树木后加载存档，树木不会复原
- 石头没有实现 `IPersistentObject`，状态完全不保存

**根本原因分析**：
- `TreeController` 已实现 `IPersistentObject`，但 `Load()` 方法可能没有正确恢复位置
- `StoneController` 完全没有实现 `IPersistentObject` 接口
- 🔥 **真正元凶**：见 P0-3，Registry 残留导致 GUID 错位

**验收标准**：
- [ ] 加载存档后，树木恢复到保存时的阶段和血量
- [ ] 加载存档后，石头恢复到保存时的阶段和血量
- [ ] 被完全砍伐/挖掘的对象在加载后应该被销毁或重新生成

---

### P2-1: 世界对象完整性（反向修剪逻辑）

**现象**：
- 保存时没有记录所有对象的完整状态
- 加载时没有处理多余对象或缺失对象
- 🔥 **被砍掉的树又回来了！**

**🔥 辅助文档补充分析**：
- 当前逻辑缺陷：目前的 Save/Load 是"存在即保存，找到即恢复"
- 缺失的一环："由于玩家操作而消失的物体"（比如被砍掉的树、挖掉的箱子）。存档文件里没有它们的数据。读档时，场景加载了默认的树/箱子，SaveManager 遍历存档发现没数据，就跳过了它们。结果就是：它们保持了场景默认状态（存在）

**需求**：
- 保存时记录所有对象的完整状态（位置、阶段、血量等）
- 🔥 **反向修剪逻辑**：加载时，如果场景中有该 GUID 但存档中没有 → 说明它应该被删除了 → 执行 Destroy/Disable
- 加载时恢复缺失对象（存档中存在但场景中不存在的）
- 支持动态生成/销毁的对象

**验收标准**：
- [ ] 保存时记录所有 IPersistentObject 的完整状态
- [ ] 加载时正确恢复所有对象状态
- [ ] 🔥 加载时处理"已删除对象"：场景中有但存档中没有 = 已删除，执行 Destroy
- [ ] 加载时处理对象的增删（需要预制体引用系统）

---

## 技术分析

### 当前存档流程

```
SaveManager.SaveGame()
├── CollectGameTimeData() → GameTimeSaveData
├── CollectPlayerData() → PlayerSaveData (只有位置)
├── CollectInventoryData() → InventorySaveData (已迁移到 IPersistentObject)
└── PersistentObjectRegistry.CollectAllSaveData() → List<WorldObjectSaveData>
    ├── ChestController.Save()
    ├── TreeController.Save()
    └── InventoryService.Save()
```

### 当前加载流程

```
SaveManager.LoadGame()
├── RestoreGameTimeData()
├── RestorePlayerData() → 只设置 player.transform.position
├── RestoreInventoryData() → 兼容旧存档
└── PersistentObjectRegistry.RestoreAllFromSaveData()
    ├── FindByGuid() → 查找已注册对象
    └── obj.Load() → 恢复状态
```

### 🔥 问题点（含辅助文档补充）

1. **玩家位置**：`RestorePlayerData()` 没有处理子物体，且可能被物理引擎覆盖
2. **箱子状态**：`_inventory` 和 `_inventoryV2` 双系统同步问题，状态标志位未复位
3. **🔥 Registry 生命周期**：单例 + DontDestroyOnLoad = 跨场景残留，导致 GUID 错位
4. **ID 冲突**：编辑器复制对象导致 ID 重复 + Registry 残留导致假冲突
5. **石头存档**：`StoneController` 没有实现 `IPersistentObject`
6. **🔥 反向修剪缺失**：没有处理"已删除对象"的逻辑

---

## 修复优先级

| 优先级 | 问题 | 影响 |
|--------|------|------|
| P0 | 🔥 Registry 生命周期 | **这是树木/石头状态无法恢复的真正元凶！** |
| P0 | 玩家位置不同步 | 游戏体验严重受损 |
| P0 | 箱子 IsEmpty 状态 | 箱子无法正常使用 |
| P1 | ID 冲突警告 | 控制台噪音，可能影响性能 |
| P1 | 树木/石头状态 | 存档功能不完整 |
| P2 | 🔥 反向修剪逻辑 | 被删除的物体会"复活" |

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `Assets/YYY_Scripts/Data/Core/SaveManager.cs` | 存档管理器 |
| `Assets/YYY_Scripts/World/Placeable/ChestController.cs` | 箱子控制器 |
| `Assets/YYY_Scripts/Controller/TreeController.cs` | 树木控制器 |
| `Assets/YYY_Scripts/Controller/StoneController.cs` | 石头控制器 |
| `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` | 持久化注册表 |
