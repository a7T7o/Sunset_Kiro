# 批量修改 SO 参数 - 需求文档

## 背景

现有 `Tool_BatchItemSOModifier.cs` 只覆盖了少量硬编码属性（通用6个 + 工具6个 + 武器5个），无法满足"显示所有属性交集并批量修改"的需求。需要全面重写，实现真正的全属性覆盖。

## 核心需求

### 1. 自动获取选中的 SO 资产
- 自动跟随 Project 窗口的多选操作
- 只识别 `ItemData` 及其所有子类
- 显示选中数量、类型分布

### 2. 属性交集计算与显示
- 分析所有选中 SO 的实际类型
- 计算类型交集：所有选中项共有的属性才显示
  - 例：全选 SeedData → 显示 ItemData + SeedData 全部字段
  - 例：混选 SeedData + ToolData → 只显示 ItemData 字段
  - 例：混选 CropData + WitheredCropData → 显示 ItemData + FoodData 字段（共同基类）
- 按 Header 分组显示，保持和 Inspector 一致的视觉结构
- 不需要获取实际数据值，只提供空白编辑框

### 3. 勾选控制机制
- 每个属性左侧有勾选框
- 勾选 = 该属性将被修改为新值
- 不勾选 = 该属性保持原值不变
- 提供"全选/全不选"快捷操作

### 4. 批量应用
- 底部"应用"按钮，一键将勾选的属性值写入所有选中 SO
- 应用前弹出确认对话框（显示修改数量和影响范围）
- 修改后自动 `SetDirty` + `SaveAssets`
- 自动同步 ItemDatabase（如果存在）

### 5. 排除字段
- `itemID` 和 `itemName` 不允许批量修改（这两个是唯一标识）
- `icon` 不批量修改（每个物品图标不同）
- 带 `[System.Obsolete]` 标记的字段不显示

## 技术方案

采用 Unity `SerializedObject` / `SerializedProperty` 反射方式：
- 自动发现所有序列化字段
- 自动适配新增的 SO 子类和字段
- 利用 `EditorGUI.PropertyField` 渲染，显示效果与 Inspector 一致

## 继承关系参考

```
ItemData（基类）
├── ToolData
├── WeaponData
├── SeedData
├── FoodData
│   ├── CropData
│   └── WitheredCropData
├── MaterialData
├── PotionData
├── KeyLockData
├── EquipmentData
├── PlaceableItemData（抽象）
│   ├── SaplingData
│   ├── WorkstationData
│   ├── StorageData
│   ├── InteractiveDisplayData
│   └── SimpleEventData
```

## 附加功能（基础功能完成后）

- 支持按 Header 分组折叠/展开
- 显示每个属性当前在选中 SO 中的值分布（如"3个=true, 2个=false"）
- 搜索/过滤属性名
- 预设保存/加载（常用修改组合）
