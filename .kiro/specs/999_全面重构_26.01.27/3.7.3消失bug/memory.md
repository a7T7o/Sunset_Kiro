# 消失 Bug 问题审核 - 开发记忆

## 模块概述

对存档系统动态重建功能进行全面问题审核，分析树木消失的根本原因。

## 当前状态

- **完成度**: 100%
- **最后更新**: 2026-02-03
- **状态**: ✅ 自动化 GUID 管理系统实现完成 + PersistentObjectRegistry 退出报错已修复

## 会话记录

### 会话 1 - 2026-02-01

**用户需求**:
> 控制台97+34条输出...现在还是有很多问题，我需要给我一个问题审核报告

**完成任务**:
1. 获取 Unity 控制台日志（131 条）
2. 分析日志中的错误和警告
3. 读取存档文件 slot1.json
4. 读取 PrefabRegistry.cs 和 DynamicObjectFactory.cs 代码
5. 创建问题审核报告

**关键发现**:
- **问题一**：prefabId 格式错误 - 存档中存的是 `M1 (5)` 而不是 `M1`
- **问题二**：GetPrefabId() 方法解析逻辑有 bug
- **问题三**：NavGrid2D 日志刷屏（每次重建都输出两行）

---

### 会话 2 - 2026-02-01（续）

**用户需求**:
> 看锐评吧孩子，你也真是粗心

**锐评020核心指令**:
1. **核心病灶一（脏名字）**：Unity 会给重复或实例化的物体自动加后缀 `(1)`, `(Clone)`，必须用正则清洗
2. **核心病灶二（GUID 漂移）**：静态树木的 GUID 没有序列化到场景文件，每次运行都生成新 GUID
3. **防御层（工厂容错）**：如果 prefabId 查找失败，尝试清洗后再查找

**完成任务**:
1. ✅ 修复 `TreeController.GetPrefabId()` - 使用正则清洗 Unity 后缀
2. ✅ 修复 `DynamicObjectFactory.TryReconstructTree()` - 添加工厂容错

**修改文件**:
- `Assets/YYY_Scripts/Controller/TreeController.cs` - GetPrefabId() 正则清洗
- `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs` - 工厂容错

**编译状态**: ✅ 成功（只有4个无关警告）

**待用户操作**:
1. 回到 Unity 编辑器
2. 在 Hierarchy 搜索 `Tree`，全选所有树木
3. 右键 -> `Force Regenerate GUID`（如果 ContextMenu 生效）
4. **🔥 最重要：按 `Ctrl + S` 保存场景！**
5. 删除旧存档 `slot1.json`
6. 重新开始游戏测试

---

### 会话 3 - 2026-02-01（续）

**用户需求**:
> 是所有的tree还是所有的Mn树木呢，你没有说清楚
> 但是这个操作如果是一直要用手动去处理的话，那难道我游玩的过程中每次保存存档前都要先对场景所有树木都这样处理吗？

**用户质疑**：手动操作不是长远之计

**锐评021-022核心指令**（架构级解决方案）:
1. **Getter 提纯**：`PersistentId` 的 Getter 不再自动生成 ID，改为报错
2. **显式初始化**：新增 `InitializeAsNewTree()` 方法，供放置系统调用
3. **生命周期闭环**：
   - 静态树：编辑器 `OnValidate` 自动生成 GUID，保存场景后固化
   - 动态树：放置时 `InitializeAsNewTree()` 自动生成 GUID
   - 读档重建：工厂 `SetPersistentIdForLoad()` 强制使用存档中的 GUID

**完成任务**:
1. ✅ 净化 `TreeController.PersistentId` Getter - 不再自动生成 ID，改为报错
2. ✅ 新增 `TreeController.InitializeAsNewTree()` 方法
3. ✅ 修改 `PlacementManager.HandleSaplingPlacementWithSavedData()` - 调用 `InitializeAsNewTree()`
4. ✅ 编译通过（只有4个无关警告）

**修改文件**:
- `Assets/YYY_Scripts/Controller/TreeController.cs` - Getter 净化 + InitializeAsNewTree
- `Assets/YYY_Scripts/Service/Placement/PlacementManager.cs` - 调用 InitializeAsNewTree

---

### 会话 4 - 2026-02-02（上下文恢复）

**状态确认**：
- 所有代码修改已完成
- 编译通过
- 等待用户最后一次手动操作验证

**最后一次手动操作**（仅此一次，之后不再需要）:
1. 在 Unity Hierarchy 搜索 `M1` 或 `M2` 或 `M3`
2. 全选所有静态树木（M1, M1 (1), M1 (2)...）
3. 右键 → `Force Regenerate GUID`
4. **🔥 Ctrl + S 保存场景**
5. 删除旧存档 `Assets/Saves/slot1.json`
6. 重新开始游戏测试

**为什么只需要这一次**：
- 静态树木的 GUID 会被序列化到场景文件中
- 之后每次打开场景，GUID 都会从场景文件加载
- 动态放置的树木会自动调用 `InitializeAsNewTree()` 生成 GUID
- 读档重建的树木会使用存档中的 GUID

---

### 会话 5 - 2026-02-02（自动化方案设计）

**用户需求**:
> 如果我后续还需要调整我的场景，需要调整树木石头位置以及其他物品的位置甚至是新增静态树木等内容，是否我每次都要进行这样的操作？那为什么不在游戏运行的时候对所有未进行处理的内容来批量处理一次呢？

**锐评023核心指令**（自动化工作流）:
- 利用 Unity 的 `EditorSceneManager.sceneSaving` 事件
- 在按下 Ctrl+S 保存场景时，自动扫描并修复所有缺失 GUID 的对象
- 无感、零操作、安全

**设计决策**:
1. **Editor 自动化守门员**：监听场景保存事件，自动修复 GUID
2. **支持所有 IPersistentObject**：不仅是树木，还包括石头、箱子等
3. **重复检测**：检测 Ctrl+D 复制导致的 GUID 冲突
4. **构建验证**：构建前验证所有场景的 GUID 完整性
5. **手动验证工具**：提供菜单命令进行主动检查

**创建文档**:
- `0_自动化GUID管理/requirements.md` - 需求文档
- `0_自动化GUID管理/design.md` - 设计文档
- `0_自动化GUID管理/tasks.md` - 任务列表

**下一步**:
- 用户审核需求和设计
- 确认后开始实现任务

---

### 会话 6 - 2026-02-02（代码实现）

**用户需求**:
> 来吧走起

**锐评024核心指令**（战术修剪）:
1. **砍掉配置系统（Task 4）**：这是核心机制，必须 Always On，不需要配置 UI
2. **简化构建验证器（Task 3）**：降级为 Log Warning，不需要复杂的阻断和弹窗

**完成任务**:
1. ✅ 创建 `Assets/Editor/PersistentIdAutomator.cs` - 核心自动化组件
   - 监听 `EditorSceneManager.sceneSaving` 事件
   - 自动扫描并修复空 GUID
   - 检测并修复重复 GUID（Ctrl+D 复制导致）
   - 支持 `persistentId` 和 `_persistentId` 两种字段名
2. ✅ 创建 `Assets/Editor/PersistentIdValidator.cs` - 手动验证工具
   - 菜单：`工具/持久化系统/验证当前场景 GUID`
   - 菜单：`工具/持久化系统/修复当前场景 GUID`
   - 菜单：`工具/持久化系统/验证所有场景 GUID`

**编译状态**: ✅ 无语法错误

**待用户验证**:
1. 回到 Unity 编辑器，等待脚本编译
2. 打开 Primary 场景
3. 按 Ctrl+S 保存场景
4. 检查控制台是否输出 `[PersistentIdAutomator] 场景 'Primary' 已修复 X 个空 GUID`
5. 再次按 Ctrl+S，应该没有输出（幂等性）

---

## 关键决策

| 决策 | 原因 | 日期 |
|------|------|------|
| 使用正则清洗名字 | Unity 的 name 属性是"脏"的，不能直接作为 ID | 2026-02-01 |
| 添加工厂容错 | 防御性编程，哪怕存档脏了也能洗干净吃下去 | 2026-02-01 |
| Getter 提纯 | 移除副作用，将 GUID 控制权从"随机发生"收回到"显式管理" | 2026-02-01 |
| 生命周期闭环 | 静态/动态/重建三种场景各有明确的 GUID 生成时机 | 2026-02-01 |
| Editor 自动化守门员 | 利用场景保存事件自动修复 GUID，彻底解放开发者双手 | 2026-02-02 |

## 反省与教训

1. **对 Unity 命名规则的无知**：没有考虑 Unity 自动添加的 ` (N)` 后缀
2. **没有检查存档文件内容**：如果第一次测试后就检查 slot1.json，会立即发现问题
3. **测试用例不全面**：没有覆盖场景中静态物体（带 Unity 编号后缀）的情况

## 相关文件

| 文件 | 说明 |
|------|------|
| `问题审核报告.md` | 详细的问题分析报告 |
| `锐评001.md` | Code Reaper 的锐评020 |
| `锐评002.md` | Code Reaper 的锐评021（架构级方案） |
| `锐评003.md` | Code Reaper 的锐评022（最终执行核准） |
| `Assets/YYY_Scripts/Controller/TreeController.cs` | Getter 净化 + InitializeAsNewTree |
| `Assets/YYY_Scripts/Service/Placement/PlacementManager.cs` | 调用 InitializeAsNewTree |
| `Assets/YYY_Scripts/Data/Core/DynamicObjectFactory.cs` | 工厂容错 + SetPersistentIdForLoad |
| `0_自动化GUID管理/requirements.md` | 自动化 GUID 管理系统需求文档 |
| `0_自动化GUID管理/design.md` | 自动化 GUID 管理系统设计文档 |
| `0_自动化GUID管理/tasks.md` | 自动化 GUID 管理系统任务列表 |
| `0_自动化GUID管理/锐评001.md` | Code Reaper 的锐评024（战术修剪） |
| `Assets/Editor/PersistentIdAutomator.cs` | 核心自动化组件（场景保存时自动修复 GUID） |
| `Assets/Editor/PersistentIdValidator.cs` | 手动验证工具（菜单命令） |

---

### 会话 7 - 2026-02-03（PersistentObjectRegistry 退出报错修复）

**用户需求**:
> 但是当我运行再结束运行会报错："Some objects were not cleaned up when closing the scene. (Did you spawn new GameObjects from OnDestroy?)The following scene GameObjects were found:[PersistentObjectRegistry]"

**问题分析**:
- `PersistentObjectRegistry` 使用懒加载单例模式
- 当场景关闭时，如果有代码访问 `Instance` 属性，会在 `OnDestroy` 期间创建新的 GameObject
- Unity 检测到这个行为并报错

**解决方案**:
1. 添加 `_isQuitting` 静态标志
2. 在 `OnApplicationQuit()` 中设置标志为 true
3. 在 `Instance` getter 中检查标志，如果正在退出则不创建新实例

**修改文件**:
- `Assets/YYY_Scripts/Data/Core/PersistentObjectRegistry.cs` - 添加退出保护机制

**编译状态**: ✅ 成功

---

### 会话 8 - 2026-02-03（控制台日志降噪）

**用户需求**:
> 请你获取最新30控制台输出，我希望你把其中不是必要的初始化输出都在脚本中集中管理，在检查器使用一个按钮show debug info进行控制是否输出控制台内容，尽可能简化控制台信息

**问题分析**:
- 控制台输出了大量初始化日志，影响调试体验
- 需要为各个组件添加 `showDebugInfo` 开关，默认关闭

**修改的文件**:
1. `Assets/YYY_Scripts/Service/Placement/PlacementPreview.cs`
   - 将 `showDebugInfo` 默认值改为 `false`
   - 为所有 `Debug.Log` 添加 `showDebugInfo` 检查
   
2. `Assets/YYY_Scripts/Service/Placement/PlacementGridCell.cs`
   - 添加静态 `showDebugInfo` 字段
   - 移除 Awake 中的日志输出
   
3. `Assets/YYY_Scripts/Service/Inventory/InventoryBootstrap.cs`
   - 添加 `showDebugInfo` 字段（默认 false）
   - 为所有日志添加开关控制
   
4. `Assets/YYY_Scripts/Service/PersistentManagers.cs`
   - 添加 `showDebugInfo` 字段（默认 false）
   - 为初始化日志添加开关控制
   
5. `Assets/YYY_Scripts/Data/Database/ItemDatabase.cs`
   - 添加 `showDebugInfo` 字段（默认 false）
   - 为加载日志添加开关控制
   
6. `Assets/YYY_Scripts/UI/Debug/SaveLoadDebugUI.cs`
   - 添加 `showDebugInfo` 字段（默认 false）
   - 移除 UI 创建日志，为状态更新日志添加开关
   
7. `Assets/YYY_Scripts/UI/Debug/DurabilityTestUI.cs`
   - 添加 `showDebugInfo` 字段（默认 false）
   - 移除 UI 创建日志，为所有操作日志添加开关

**编译状态**: ✅ 成功

**效果**:
- 默认情况下，控制台将不再输出这些初始化日志
- 需要调试时，可以在 Inspector 中勾选对应组件的 `Show Debug Info` 开关
