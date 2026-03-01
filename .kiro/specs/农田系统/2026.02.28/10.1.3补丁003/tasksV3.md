# 补丁003 任务清单 V3 — 预览系统重构与bug修复

> 来源：`designV3.md`（模块 J/K/L/M）
> 执行顺序：先修bug（J→K→L），再重构（M），最后验证

---

## 阶段一：bug修复

- [x] 1. 模块 J：种子预览位置修复（P4）
  - [x] 1.1 在 `FarmToolPreview.UpdateSeedPreview` 中，计算出 `cellPos` 后使用 `GetCellCenterWorld(layerIndex, cellPos)` 获取正确坐标
  - [x] 1.2 将 `seedGridRenderer.transform.position` 和 `seedPreviewRenderer.transform.position` 改为使用正确坐标（替代 `alignedPos`）
  - [x] 1.3 验证：getDiagnostics 编译通过

- [x] 2. 模块 K：耕地队列预览8+1完整化（P2）
  - [x] 2.1 新增字段 `Dictionary<Vector3Int, List<Vector3Int>> tillQueueTileGroups`
  - [x] 2.2 改造 `AddQueuePreview` Till 分支：遍历 `GetPreviewTiles` 全部 key-value，放置到 queuePreviewTilemap，记录到 tillQueueTileGroups
  - [x] 2.3 改造 `RemoveQueuePreview`：耕地类型从 tillQueueTileGroups 获取关联位置列表，逐个清除
  - [x] 2.4 改造 `ClearAllQueuePreviews`：清空 tillQueueTileGroups
  - [x] 2.5 验证：getDiagnostics 编译通过

- [x] 3. 模块 L：浇水队列预览一致性（P3）
  - [x] 3.1 `GameInputManager` 的 `FarmActionRequest` 结构体新增 `int puddleVariant` 字段（默认 -1）
  - [x] 3.2 `EnqueueAction` 中 Water 类型入队时预分配 `puddleVariant`
  - [x] 3.3 `AddQueuePreview` 新增 `puddleVariant` 参数，Water 分支使用确定性 tile
  - [x] 3.4 `GameInputManager.EnqueueAction` 调用 `AddQueuePreview` 时传递 `puddleVariant`
  - [x] 3.5 `ExecuteFarmAction` Water 分支将预分配的 variant 写入 `tileData.puddleVariant`
  - [x] 3.6 验证：getDiagnostics 编译通过

## 阶段二：预览系统视觉重构

- [x] 4. 模块 M-1：创建 FarmPreviewOverlay shader
  - [x] 4.1 在 `Assets/444_Shaders/Shader/` 下创建 `FarmPreviewOverlay.shader`
  - [x] 4.2 基于 Sprites-Default，新增 `_OverlayColor` 属性，片元着色器实现 lerp 混合
  - [x] 4.3 验证：Unity 编译 shader 无错误

- [x] 5. 模块 M-2：EnsureComponents 改造
  - [x] 5.1 加载 FarmPreviewOverlay shader，创建 Material 实例
  - [x] 5.2 赋给 ghostTilemapRenderer
  - [x] 5.3 移除 hoeOverlayRenderer / waterOverlayRenderer / overlaySprite 的创建逻辑
  - [x] 5.4 移除 CreateOverlaySprite() 和 CreateOverlayRenderer() 方法
  - [x] 5.5 验证：getDiagnostics 编译通过

- [x] 6. 模块 M-3：UpdateHoePreview 改造
  - [x] 6.1 移除 `hoeOverlayRenderer.enabled = true` 和颜色设置代码
  - [x] 6.2 在放置 1+8 tile 后，根据 isValid 设置 ghostTilemapRenderer.material 的 `_OverlayColor`
  - [x] 6.3 验证：getDiagnostics 编译通过

- [x] 7. 模块 M-4：UpdateWateringPreview 改造
  - [x] 7.1 移除 `waterOverlayRenderer.enabled = true` 和颜色设置代码
  - [x] 7.2 在放置水渍 tile 后，根据 isValid 设置 ghostTilemapRenderer.material 的 `_OverlayColor`
  - [x] 7.3 验证：getDiagnostics 编译通过

- [x] 8. 模块 M-5：Hide 方法和字段清理
  - [x] 8.1 移除 Hide 中 hoeOverlayRenderer / waterOverlayRenderer 的隐藏逻辑
  - [x] 8.2 移除 UpdateHoePreview / UpdateWateringPreview 中对其他覆盖层的隐藏代码（已不存在）
  - [x] 8.3 清理废弃字段声明（hoeOverlayRenderer、waterOverlayRenderer、overlaySprite）
  - [x] 8.4 验证：getDiagnostics 编译通过

## 阶段三：全面验证

- [x] 9. 编译验证
  - [x] 9.1 对所有修改文件执行 getDiagnostics，确认 0 错误 0 警告
  - [x] 9.2 确认无弃用 API 使用

- [x] 10. 正确性属性逐项审查
  - [x] 10.1 逐条检查 CP-J1~CP-M6 共14条正确性属性在代码中的对应实现
