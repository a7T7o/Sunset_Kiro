---
inclusion: manual
priority: P1
keywords: [物品, 背包, 堆叠, ItemStack, Inventory, 品质]
lastUpdated: 2026-02-11
---

# 物品系统规范

## 物品ID分配规则

> **物品 ID 分配的权威来源是 `.kiro/specs/SO设计系统与工具/ID分配规范.md`。**
> **`so-design.md` 中有简要摘要，此处为极简版。**

```
0XXX: 工具和武器 (00XX农具, 01XX采集, 02XX武器)
1XXX: 种植与放置类 (10XX种子, 11XX农作物[含枯萎], 12XX树苗, 13XX工作台, 14XX存储/钥匙锁, 15XX交互展示, 16XX简单事件)
2XXX: 动物产品（预留）
3XXX: 矿物和材料 (30XX矿石, 31XX锭, 32XX自然, 33XX怪物)
4XXX: 消耗品 (40XX药水)
5XXX: 食品 (50XX简单, 51XX高级)
6XXX: 家具
7XXX: 特殊物品
8XXX: 装备（EquipmentData）
```

## 数据结构

### ItemData (基类)
```csharp
// 核心字段
int itemID;           // 唯一ID
string itemName;      // 物品名称
string description;   // 描述
Sprite icon;          // 图标
ItemCategory category; // 分类
int buyPrice;         // 购买价格
int sellPrice;        // 出售价格
int maxStackSize;     // 最大堆叠数
```

### ToolData (继承 ItemData)
```csharp
ToolType toolType;           // 工具类型
ToolLevel toolLevel;         // 工具等级
float energyCost;            // 精力消耗
int effectRadius;            // 作用范围
bool useQualityIdMapping;    // 动画ID映射开关
int animationDefaultId;      // 默认动画ID
```

### WeaponData (继承 ItemData)
```csharp
WeaponType weaponType;       // 武器类型
int attackPower;             // 攻击力
float attackSpeed;           // 攻击速度
float criticalChance;        // 暴击率
float criticalDamage;        // 暴击伤害
float attackRange;           // 攻击范围
float knockbackForce;        // 击退力度
float energyCostPerAttack;   // 每次攻击精力消耗
```

### SeedData (继承 ItemData)
```csharp
int growthDays;              // 生长天数
Season season;               // 适合季节
int harvestCropID;           // 收获作物ID
Vector2Int harvestAmountRange; // 收获数量范围
```

## 品质系统

> **品质定义的权威来源是 `so-design.md`，此处保持同步。**

### 品质等级（ItemQuality 枚举）

| 品质 | 值 | 价格倍率 | 星星颜色 |
|------|-----|---------|---------|
| Normal | 0 | ×1.0 | 无星星 |
| Rare | 1 | ×1.25 | 蓝色 |
| Epic | 2 | ×2.0 | 紫色 |
| Legendary | 3 | ×3.25 | 金色 |

> 价格计算后**向上取整** (`Mathf.CeilToInt`)

### 品质规则
- 品质不改变物品外观，只在 UI 显示星星
- 不同品质物品不能堆叠
- 收获时根据等级和肥料随机判定品质

## ItemDatabase 访问

### 正确做法
```csharp
// ItemDatabase 是 ScriptableObject，不是单例
// 从已有引用的 MonoBehaviour 获取
if (inventory != null)
    database = inventory.Database;
```

### 错误做法
```csharp
// ❌ 永远返回 null
database = FindFirstObjectByType<ItemDatabase>();
```

## 背包系统

### ItemStack 结构
```csharp
public struct ItemStack
{
    public int itemId;    // 物品ID
    public int quality;   // 品质等级
    public int amount;    // 数量
    public bool IsEmpty => itemId < 0 || amount <= 0;
}
```

### 容量配置
- 主背包：20 格
- 快捷栏：8 格（1-8 数字键切换）

### 堆叠逻辑
- 相同物品自动堆叠
- 达到上限后占用新格子
- 不同品质不能堆叠

## 编辑器工具

### 批量创建工具
- 菜单：`Farm → Items → 批量创建物品数据 (SO)`
- 支持多 Sprite 重排、并行 ID/名称输入
- 首 ID 自增、类型专属字段、规范命名与保存路径

### 自动化初始化
- 菜单：`Farm → Setup → 完整初始化（推荐）`
- 一键创建数据库和测试物品

### 数据库操作
- 右键 `MasterItemDatabase.asset` → 自动收集所有物品数据
- 右键 → 自动收集所有配方数据
- 右键 → 验证所有数据完整性
