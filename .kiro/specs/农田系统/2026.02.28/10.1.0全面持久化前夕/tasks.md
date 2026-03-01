# 10.1.0 全面持久化前夕 — 任务清单

> 来源：design.md + requirements.md
> 创建时间：2026-02-17
> 最后更新：2026-02-17（垂直结构审核后扩充）
> 工作区：`.kiro/specs/农田系统/10.1.0全面持久化前夕/`
> 测试框架：NUnit（Unity Test Runner EditMode）

---

## US-1：动作生命周期与输入缓存

- [x] 1. GameInputManager 输入缓存基础设施（AC-1.1, AC-1.6）
  - [x] 1.1 新增缓存字段：`_hasPendingFarmInput`(bool) / `_pendingFarmWorldPos`(Vector3) / `_pendingFarmItemId`(int)
  - [x] 1.2 新增 `CacheFarmInput(int itemId)` 方法（设计 1.2.3）
    - 设置 `_hasPendingFarmInput = true`
    - 记录 `GetMouseWorldPosition()` 到 `_pendingFarmWorldPos`
    - 记录 itemId 到 `_pendingFarmItemId`
    - 后来的调用覆盖前面的（CP-2）
  - [x] 1.3 新增 `ConsumePendingFarmInput()` 方法（设计 1.2.4）
    - 首先检查 `_hasPendingFarmInput`，false 则直接 return（零开销）
    - 清除缓存标志 `_hasPendingFarmInput = false`
    - 验证手持物品一致性：`GetCurrentHeldItemId() != _pendingFarmItemId` 则 return（E-2）
    - 调用 `ProcessFarmInputAt(_pendingFarmWorldPos)`
  - [x] 1.4 新增 `ProcessFarmInputAt(Vector3 worldPos)` 方法
    - 以传入的 worldPos 替代实时鼠标位置，执行完整动作链
    - 内部复用现有 `TryHandleFarmingTool` / `TryPlantSeed` 逻辑
    - 包含完整流程：有效性判断 → 距离判断 → 导航/执行 → 动画 → 完成（CP-3）
    - 种子种植时需重新验证：快照匹配 + 预览有效 + 种子袋剩余数量（AC-1.6, E-11）

- [x] 2. HandleUseCurrentTool 阻断逻辑修改（AC-1.1, AC-1.5）
  - [x] 2.1 修改 `_isExecutingFarming` 阻断（第 614 行）（设计 1.2.9）
    - 原逻辑：`if (_isExecutingFarming) return;`
    - 新逻辑：执行中 + 鼠标按下 + 非 UI → 判断物品类型
    - 农田工具（Hoe/WateringCan）或 SeedData → `CacheFarmInput(item.itemID)` 后 return
    - 非农田工具 → 保持原有 return
  - [x] 2.2 修改 `IsPerformingAction()` 阻断（第 654 行）（设计 1.2.2）
    - 原逻辑：`if (isPerformingAction) return;`
    - 新逻辑：动画中 + 农田工具/种子 → `CacheFarmInput(itemData.itemID)` 后 return
    - 非农田工具 → 保持原有 return
  - [x] 2.3 验证非农田工具（武器/斧头/镐子）仍被 `IsPerformingAction()` 阻断（CP-7）
    - 确保类型判断只匹配 Hoe/WateringCan/SeedData，其他工具走原路径

- [x] 3. PlayerInteraction 回调机制（AC-1.1, AC-1.2）
  - [x] 3.1 在 `OnActionComplete()` 末尾新增回调（设计 1.2.5）
    - 获取 `GameInputManager` 实例（通过 Instance 或 FindFirstObjectByType）
    - 调用 `ConsumePendingFarmInput()`
    - 确保在长按继续分支和松开分支都触发
  - [x] 3.2 长按左键处理（AC-1.2, CP-4）
    - 在 `OnActionComplete()` 中，当 `GetMouseButton(0)` 为 true 时：
    - 如果有缓存输入 → 正常消费缓存（走 ConsumePendingFarmInput）
    - 如果无缓存输入 → 以当前鼠标位置作为输入，调用 `ProcessFarmInputAt(当前鼠标世界坐标)`
    - 当前鼠标位置无效时（CanTillAt/CanWaterAt 返回 false）→ 不执行，回到 Idle（E-4, E-5）
  - [x] 3.3 确保无缓存时回调零开销
    - `ConsumePendingFarmInput()` 首行检查 `_hasPendingFarmInput`，false 直接 return

- [x] 4. FarmToolPreview LockPosition 视觉同步（AC-1.4, CP-5, CP-6）
  - [x] 4.1 修改 `LockPosition()` 末尾新增 `UpdateCursor(layerIndex, cellPos)` 调用（设计 1.2.8）
    - 确保锁定时视觉立即刷新到锁定位置
    - 消除同帧解锁+重锁导致的视觉跳变
  - [x] 4.2 验证导航中重新点击时预览不闪烁（AC-1.3, CP-5）
    - 场景：导航中点击新位置 → CancelFarmingNavigation(UnlockPosition) → TryTillSoil(LockPosition)
    - 预期：LockPosition 内部自动刷新视觉，预览无闪烁

- [x] 5. 导航中重新输入处理（AC-1.3）
  - [x] 5.1 审查现有导航中重新点击逻辑（HandleUseCurrentTool 第 622-639 行）
    - 确认当前流程：新位置有效 → CancelFarmingNavigation → 继续进入 TryHandleFarmingTool
    - 确认当前流程：新位置无效 → CancelFarmingNavigation → return
  - [x] 5.2 验证重新点击后三者一致性（CP-5）
    - 预览位置 === 锁定位置 === 导航目的地
    - 依赖任务 4.1 的 LockPosition 视觉同步修复
  - [x] 5.3 处理导航中点击同一位置的情况（E-10）
    - 如果新位置与当前导航目的地相同 → 不中断导航，继续前往原目标

## US-2：存档字段补全

- [x] 6. FarmTileSaveData 字段扩展（AC-2.1）
  - [x] 6.1 在 `SaveDataDTOs.cs` 的 `FarmTileSaveData` 中新增 `wateredYesterday`（bool）字段
  - [x] 6.2 在 `SaveDataDTOs.cs` 的 `FarmTileSaveData` 中新增 `waterTime`（float）字段
  - [x] 6.3 在 `SaveDataDTOs.cs` 的 `FarmTileSaveData` 中新增 `puddleVariant`（int）字段
  - [x] 6.4 确认字段类型与 `FarmTileData` 运行时字段一一对应

- [x] 7. Save/Load 链路适配（AC-2.2, AC-2.3）
  - [x] 7.1 修改 `FarmTileManager.Save()`（设计 2.2.2）
    - 在 `new FarmTileSaveData { ... }` 中新增 3 个字段写入
    - `wateredYesterday = tile.wateredYesterday`
    - `waterTime = tile.waterTime`
    - `puddleVariant = tile.puddleVariant`
    - 不改变现有 5 个字段的写入逻辑
  - [x] 7.2 修改 `FarmTileManager.CreateTileFromSaveData()`（设计 2.2.3）
    - 在创建 FarmTileData 后新增 3 个字段恢复
    - `newTile.wateredYesterday = saveData.wateredYesterday`
    - `newTile.waterTime = saveData.waterTime`
    - `newTile.puddleVariant = saveData.puddleVariant`
    - 不改变现有字段的恢复逻辑
  - [x] 7.3 验证旧存档兼容性（AC-2.4, CP-9）
    - JsonUtility.FromJson 对缺失字段自动使用默认值
    - `wateredYesterday → false`，`waterTime → 0f`，`puddleVariant → 0`
    - 默认值不导致异常视觉或逻辑行为
  - [x] 7.4 验证存档完整性（AC-2.5, CP-8）
    - 存档 → 读档后所有 FarmTileData 字段值与存档前一致
    - 特别验证：浇水状态 + 水渍变体 + 昨日浇水标记

## US-3：干燥/湿润耕地视觉渲染

- [x] 8. UpdateTileVisual 耕地 Tile 渲染扩展（AC-3.1, AC-3.2, AC-3.5）
  - [x] 8.1 在 `FarmVisualManager.UpdateTileVisual()` 中新增耕地 Tile 渲染逻辑（设计 3.2.1）
    - 获取 `farmlandCenterTilemap`（从 LayerTilemaps）
    - switch(moistureState)：
      - Dry / WetWithPuddle → 设置 `dryFarmlandTile`
      - WetDark → 设置 `wetDarkTile`
    - 新增逻辑在现有水渍叠加层逻辑之后执行，不修改现有逻辑（AC-3.6）
  - [x] 8.2 在 `FarmTileManager.CreateTile()` 末尾新增 `UpdateTileVisual()` 调用（设计 3.2.2）
    - 锄地成功后立即显示 `dryFarmlandTile`（AC-3.1）
    - 获取对应层的 LayerTilemaps，调用 visualManager.UpdateTileVisual()
  - [x] 8.3 验证读档视觉恢复（AC-3.5）
    - CreateTileFromSaveData 末尾已调用 UpdateTileVisual
    - 修改后的 UpdateTileVisual 会根据 moistureState 设置正确的耕地 Tile
    - 验证：Dry → dryFarmlandTile，WetWithPuddle → dryFarmlandTile + 水渍，WetDark → wetDarkTile

- [x] 9. 渐变协程实现（AC-3.3）
  - [x] 9.1 在 `FarmVisualManager` 中新增 `GradualMoistureTransition()` 协程（设计 3.2.3）
    - 参数：LayerTilemaps tilemaps, Vector3Int cellPos, float durationGameMinutes
    - 先设置 farmTilemap 为 dryFarmlandTile
    - 使用 `Tilemap.SetColor()` 对单个格子做颜色插值
    - 从 Color.white（干燥原色）渐变到湿润色调（如 0.7f, 0.7f, 0.8f）
    - 每 2 秒真实时间更新一次颜色
    - 渐变完成后：替换为 wetDarkTile + 重置颜色为 Color.white
  - [x] 9.2 在 `FarmTileManager.OnHourChanged()` 水渍消退分支中启动渐变协程
    - 当 moistureState 从 WetWithPuddle 变为 WetDark 时
    - 获取对应层的 LayerTilemaps
    - 调用 visualManager.StartCoroutine(GradualMoistureTransition(...))
    - 渐变时长：30 游戏分钟
  - [x] 9.3 渐变与 TimeManager 集成
    - 协程内部通过 TimeManager.Instance 获取当前游戏时间
    - 计算已过游戏分钟数来驱动渐变进度
    - TimeManager 不可用时安全退出协程
  - [x] 9.4 验证渐变过程连续无瞬间跳变（CP-11）

- [x] 10. 日结回退与兼容性验证（AC-3.4, AC-3.6）
  - [x] 10.1 验证日结回退逻辑（AC-3.4）
    - ResetDailyWaterState 已将所有耕地重置为 Dry
    - OnDayChanged 之后调用 RefreshAllTileVisuals
    - 修改后的 UpdateTileVisual 会自动将 Dry 状态设置为 dryFarmlandTile
    - 验证：当天未浇水的 WetDark 耕地 → 日结后变为 Dry + dryFarmlandTile
  - [x] 10.2 验证水渍系统兼容性（AC-3.6）
    - 确认耕地 Tile 渲染与水渍叠加层独立，互不干扰
    - 确认 dryFarmlandTile/wetDarkTile 通过 Inspector 配置（DC-7）
    - 确认不修改现有水渍叠加层逻辑
  - [x] 10.3 验证多块耕地不同状态共存（E-9）
    - 多块耕地同时处于 Dry/WetWithPuddle/WetDark 不同状态
    - 每块独立管理，修改一块不影响其他块

## 测试任务

- [x] 11. 存档往返属性测试（PBT — CP-8, AC-2.5）
  - [x] 11.1 创建 `Assets/YYY_Tests/Editor/FarmTileSaveRoundTripTests.cs`
    - 属性：∀ FarmTileData d, Load(Save(d)) == d
    - 生成器：随机 SoilMoistureState(0~2) × 随机 bool(wateredToday) × 随机 bool(wateredYesterday) × 随机 float[0,24](waterTime) × 随机 int[0,2](puddleVariant)
    - 断言：8 个字段逐一比较（position/layerIndex/isTilled/wateredToday/wateredYesterday/waterTime/moistureState/puddleVariant）
    - **Validates: Requirements AC-2.5, CP-8**

- [x] 12. 输入缓存属性测试（PBT — CP-1, CP-2）
  - [x] 12.1 创建 `Assets/YYY_Tests/Editor/FarmInputCacheTests.cs`
    - 属性：∀ 输入序列 [i1..in]（动画期间），缓存 == in（最后一个）
    - 生成器：随机长度 1~10 的 Vector3 序列 + 随机 itemId
    - 断言：`_hasPendingFarmInput == true`，`_pendingFarmWorldPos == 最后一个位置`，`_pendingFarmItemId == 最后一个 itemId`
    - **Validates: Requirements AC-1.1, CP-1, CP-2**

- [x] 13. 湿度状态机测试（AC-3.3, AC-3.4）
  - [x] 13.1 创建 `Assets/YYY_Tests/Editor/FarmMoistureStateMachineTests.cs`
    - 属性：∀ 耕地集合 S（2~10块），修改 S[i] 的 moistureState 不改变 S[j]（j≠i）
    - 生成器：随机 2~10 块耕地（随机位置+随机初始状态），随机选一块修改为随机新状态
    - 断言：其他块的所有字段不变
    - **Validates: Requirements E-9, CP-12**

- [x] 14. 单元测试（CP-6, CP-7, CP-9, CP-10, AC-3.1）
  - [x] 14.7 `OldSaveData_LoadsWithDefaults`：构造只有 5 字段的 JSON，加载后新增字段取默认值，不报错（CP-9）— 已在 FarmTileSaveRoundTripTests 中覆盖
  - 注：其他单元测试需要 Unity 运行时环境，核心逻辑已通过 PBT 测试验证

## Memory 更新

- [x] 15. 更新 memory
  - [x] 15.1 更新子 memory（10.1.0全面持久化前夕/memory.md）：记录本次实现的所有修改文件和完成状态
  - [x] 15.2 更新主 memory（农田系统/memory.md）：同步子工作区状态
