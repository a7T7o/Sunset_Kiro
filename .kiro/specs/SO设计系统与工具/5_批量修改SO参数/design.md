# 批量修改 SO 参数 - 设计文档

## 整体架构

重写 `Tool_BatchItemSOModifier.cs`，从硬编码方式改为基于 `SerializedProperty` 的反射方式。

## 核心数据结构

### PropertyEntry（属性条目）

每个可修改的属性对应一个条目：

```csharp
class PropertyEntry
{
    string propertyPath;      // SerializedProperty 路径（如 "buyPrice"）
    string displayName;       // 显示名称（如 "购买价格"）
    string headerGroup;       // 所属 Header 分组（如 "=== 经济属性 ==="）
    bool isEnabled;           // 是否勾选修改
    SerializedProperty templateProperty;  // 模板属性（用于渲染编辑器和存储新值）
}
```

### HeaderGroup（分组）

```csharp
class HeaderGroup
{
    string name;              // 分组名称
    bool isFolded;            // 是否折叠
    List<PropertyEntry> properties;  // 该组下的属性列表
}
```

## 核心流程

### 1. 选择变更时 → 重建属性列表

```
OnSelectionChanged()
  → 收集所有选中的 ItemData
  → 计算共同基类（最近公共祖先类型）
  → 用第一个 SO 创建 SerializedObject 作为模板
  → 遍历模板的所有 SerializedProperty
  → 过滤排除字段（itemID, itemName, icon, Obsolete 等）
  → 只保留共同基类及其父类中定义的字段
  → 按 Header 分组组织
```

### 2. 共同基类计算算法

```
输入：所有选中 SO 的 System.Type 集合
输出：最近公共祖先类型

算法：
1. 对每个类型，构建从自身到 ItemData 的继承链
2. 从最深层开始比较所有继承链
3. 找到所有链共有的最深类型 = 共同基类
```

示例：
- {SeedData, SeedData} → SeedData
- {SeedData, ToolData} → ItemData
- {CropData, WitheredCropData} → FoodData
- {SaplingData, StorageData} → PlaceableItemData

### 3. 字段过滤规则

保留条件（全部满足）：
- 是序列化字段（`SerializedProperty` 可见）
- 声明在共同基类或其父类中（不是子类独有的）
- 不在排除列表中
- 不带 `[System.Obsolete]` 特性
- 不是 `m_Script` 等 Unity 内部字段

排除列表：
- `m_Script`、`m_Name`（Unity 内部）
- `itemID`、`itemName`、`icon`（唯一标识，不批量改）

### 4. 属性渲染

使用 `EditorGUI.PropertyField` 渲染每个属性：
- 左侧 Toggle 控制是否启用
- 禁用状态下灰显
- 自动支持所有字段类型（int、float、bool、enum、string、Object 引用、Vector2、Vector3 等）

### 5. 应用修改

```
对每个选中的 SO：
  创建 SerializedObject
  对每个 isEnabled=true 的 PropertyEntry：
    找到对应的 SerializedProperty
    从模板复制值到目标
  ApplyModifiedProperties()
```

值复制使用 `SerializedProperty` 的类型分支：
- int → intValue
- float → floatValue
- bool → boolValue
- string → stringValue
- enum → enumValueIndex
- Object → objectReferenceValue
- Vector2 → vector2Value
- Vector3 → vector3Value
- 等等

## UI 布局

```
┌─────────────────────────────────────────┐
│        📝 批量修改物品 SO（增强版）         │
├─────────────────────────────────────────┤
│ 🖼️ 选中的 SO（自动跟随 Project 选择）     │
│ ✓ 已选择 5 个 SO（共同类型: SeedData）    │
│ ┌─────────────────────────────────┐     │
│ │ [1000] 大蒜 (SeedData)          │     │
│ │ [1001] 土豆 (SeedData)          │     │
│ │ ...                             │     │
│ └─────────────────────────────────┘     │
├─────────────────────────────────────────┤
│ [全选] [全不选]                          │
├─────────────────────────────────────────┤
│ ▼ === 基础识别信息 ===                    │
│   □ 描述          [____________]         │
│   □ 分类          [Plant      ▼]        │
├─────────────────────────────────────────┤
│ ▼ === 视觉资源 ===                       │
│   □ Bag Sprite    [None (Sprite) ]      │
│   □ 背包图标旋转   [✓]                   │
│   □ World Prefab  [None (GameObject)]   │
├─────────────────────────────────────────┤
│ ▼ === 经济属性 ===                       │
│   □ 购买价格      [----160----]          │
│   □ 出售价格      [----156----]          │
├─────────────────────────────────────────┤
│ ▼ === 堆叠属性 ===                       │
│   □ 最大堆叠数    [----32-----]          │
│ ...（更多分组）                           │
├─────────────────────────────────────────┤
│ ▼ === 种植专属属性 ===（SeedData 独有）    │
│   □ 适合季节      [Spring    ▼]         │
│   □ 可重复收获    [□]                    │
│   □ 重复收获天数  [----2------]          │
│ ...                                     │
├─────────────────────────────────────────┤
│                                         │
│  [🚀 应用修改到 5 个 SO（3 个字段）]      │
│                                         │
└─────────────────────────────────────────┘
```

## 菜单入口

保持原有菜单路径：`Tools/📝 批量修改物品 SO`

## 文件

只修改一个文件：`Assets/Editor/Tool_BatchItemSOModifier.cs`（完全重写）
