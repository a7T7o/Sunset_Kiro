# 农田系统重生 - 任务列表

**版本**: 9.0.1
**创建时间**: 2026-02-05

---

## P0：修复与防御

- [x] 1. FarmTileManager 配置防御
  - [x] 1.1 在 CreateTile 开头添加 groundTilemap 强制检查
  - [x] 1.2 在 CanTillAt 开头添加 groundTilemap 强制检查
  - [x] 1.3 配置缺失时输出 Debug.LogError

- [x] 2. GameInputManager 交互反馈
  - [x] 2.1 在 TryTillSoil 中添加详细日志
  - [x] 2.2 明确输出是 CanTillAt 失败还是 CreateTile 失败

---

## P1：FarmTileManager 存档集成

- [x] 3. SaveDataDTOs 数据结构
  - [x] 3.1 新增 FarmTileListWrapper 类
  - [x] 3.2 标记 FarmTileSaveData 中的作物字段为 [Obsolete]

- [x] 4. FarmTileManager 实现 IPersistentObject
  - [x] 4.1 添加 IPersistentObject 接口实现
  - [x] 4.2 实现 Save() 方法：序列化所有耕地数据
  - [x] 4.3 实现 Load() 方法：恢复耕地数据并放置 Tile
  - [x] 4.4 添加 ClearAllTiles() 方法
  - [x] 4.5 添加 CreateTileFromSaveData() 方法

- [x] 5. 验证 FarmTileManager 存档
  - [x] 5.1 测试保存耕地数据
  - [x] 5.2 测试加载耕地数据
  - [x] 5.3 验证 Tilemap 正确显示

---

## P1：作物身份统一化

- [x] 6. SaveDataDTOs 作物数据结构
  - [x] 6.1 新增 CropSaveData 类

- [x] 7. CropController 实现 IPersistentObject
  - [x] 7.1 添加 persistentId 字段
  - [x] 7.2 添加 IPersistentObject 接口实现
  - [x] 7.3 实现 Save() 方法：序列化作物数据
  - [x] 7.4 实现 Load() 方法：恢复作物数据并注册占用

- [x] 8. 验证作物存档
  - [x] 8.1 测试保存作物数据
  - [x] 8.2 测试加载作物数据
  - [x] 8.3 验证作物 GameObject 正确显示
  - [x] 8.4 验证作物与耕地关联正确

---

## 验收测试

- [x] 9. 完整流程验收
  - [x] 9.1 锄地功能正常（Tile 显示）
  - [x] 9.2 保存游戏后耕地数据存在
  - [x] 9.3 加载游戏后耕地正确恢复
  - [x] 9.4 种植作物后保存/加载正常
  - [x] 9.5 旧存档兼容性测试
