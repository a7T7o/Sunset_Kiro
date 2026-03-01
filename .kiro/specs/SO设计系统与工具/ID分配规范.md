# 物品 ID 分配权威规范

> **本文档是项目中物品 ID 分配的唯一权威来源。**
> 所有工具、steering 规则、设计文档中的 ID 信息均以本文档为准。
> 
> 事实标准来源：`Tool_BatchItemSOGenerator.cs` 的 `SubTypeStartIDs` 字典

---

## ID 总览

```
0XXX: 工具和武器
    00XX: 农具（ToolData）          起始 0
    01XX: 采集工具（ToolData）
    02XX: 武器（WeaponData）        起始 200

1XXX: 种植与放置类
    10XX: 种子（SeedData）          起始 1000
    11XX: 农作物（共享段）
        1100-1149: 正常作物（CropData）      起始 1100
        1150-1199: 枯萎作物（WitheredCropData） 起始 1150
    12XX: 树苗（SaplingData）       起始 1200
    13XX: 工作台（WorkstationData） 起始 1300
    14XX: 存储/钥匙锁
        1400-1409: 存储（StorageData）       起始 1400
        1410-1419: 锁（LockData）            起始 1410
        1420-1499: 钥匙（KeyData）           起始 1420
    15XX: 交互展示（InteractiveDisplayData） 起始 1500
    16XX: 简单事件（SimpleEventData）        起始 1600

2XXX: 动物产品（预留，尚未设计）

3XXX: 矿物和材料（MaterialData）    起始 3200
    30XX: 矿石（Ore）              已有 3000-3002
    31XX: 锭（Ingot）              已有 3100-3102
    32XX: 自然材料（Natural）       已有 3200-3201
    33XX: 怪物掉落（Monster）       预留

4XXX: 消耗品
    40XX: 药水（PotionData）        起始 4000

5XXX: 食品
    50XX: 简单食物（FoodData）      起始 5000
    51XX: 高级食物

6XXX: 家具（FurnitureData）         起始 6000

7XXX: 特殊物品（SpecialData）       起始 7000

8XXX: 装备（EquipmentData）         起始 8000
    8000-8099: 头盔（Helmet）       槽位 0
    8100-8199: 盔甲（Armor）        槽位 2
    8200-8299: 裤子（Pants）        槽位 1
    8300-8399: 鞋子（Shoes）        槽位 3
    8400-8499: 戒指（Ring）         槽位 4/5
    8500-8599: 饰品（Accessory）    预留
```

---

## 工具映射表（SubTypeStartIDs 事实标准）

| ItemSOType | StartID | 容量 | 说明 |
|---|---|---|---|
| ToolData | 0 | 00XX-01XX | 农具+采集工具 |
| WeaponData | 200 | 02XX | 武器 |
| SeedData | 1000 | 10XX | 种子 |
| CropData | 1100 | 1100-1149 | 正常作物 |
| WitheredCropData | 1150 | 1150-1199 | 枯萎作物 |
| SaplingData | 1200 | 12XX | 树苗 |
| WorkstationData | 1300 | 13XX | 工作台 |
| StorageData | 1400 | 1400-1409 | 存储容器 |
| LockData | 1410 | 1410-1419 | 锁 |
| KeyData | 1420 | 1420-1499 | 钥匙 |
| InteractiveDisplayData | 1500 | 15XX | 交互展示（告示牌等） |
| SimpleEventData | 1600 | 16XX | 简单事件（传送点等） |
| MaterialData | 3200 | 30XX-33XX | 材料（含矿石/锭/自然/怪物） |
| PotionData | 4000 | 40XX | 药水 |
| FoodData | 5000 | 50XX-51XX | 食物 |
| FurnitureData | 6000 | 6XXX | 家具 |
| SpecialData | 7000 | 7XXX | 特殊物品 |
| EquipmentData | 8000 | 8XXX | 装备 |
| ItemData | 0 | - | 基础物品（通用） |

---

## 装备子分类 ID 范围

> 来源：`DrawEquipmentSettings()` 中的推荐 ID 范围和槽位映射

| 子类型 | EquipmentType 枚举值 | ID 范围 | 装备槽位 | 状态 |
|---|---|---|---|---|
| 头盔（Helmet） | 1 | 8000-8099 | 槽位 0 | 预留 |
| 盔甲（Armor） | 2 | 8100-8199 | 槽位 2 | 预留 |
| 裤子（Pants） | 3 | 8200-8299 | 槽位 1 | 预留 |
| 鞋子（Shoes） | 4 | 8300-8399 | 槽位 3 | 已有 8300 |
| 戒指（Ring） | 5 | 8400-8499 | 槽位 4/5 | 已有 8400 |
| 饰品（Accessory） | 6 | 8500-8599 | 预留 | 预留 |

装备不可堆叠，`maxStackSize` 固定为 1。

---

## 材料子分类 ID 范围

> 来源：`DrawMaterialSettings()` 中的 MaterialSubType 枚举和推荐 ID 范围

| 子类型 | MaterialSubType 枚举 | ID 范围 | 状态 |
|---|---|---|---|
| 矿石（Ore） | Ore | 30XX | 已有 3000-3002（黄铜矿/生铁矿/黄金矿） |
| 锭（Ingot） | Ingot | 31XX | 已有 3100-3102（黄铜锭/生铁锭/黄金锭） |
| 自然材料（Natural） | Natural | 32XX | 已有 3200-3201（木材/石料） |
| 怪物掉落（Monster） | Monster | 33XX | 预留 |

矿石可设置熔炼属性（`canBeSmelt` + `smeltResultID`），产物指向对应锭的 ID。

---

## 锁的 ID 细分

> 来源：`DrawLockSettings()` 中的说明

| ID | 锁类型 | ChestMaterial |
|---|---|---|
| 1410 | 木锁 | Wood |
| 1411 | 铁锁 | Iron |
| 1412+ | 特殊锁 | Special |

锁的使用规则：
- 必须与箱子材质匹配才能上锁
- 使用后箱子变为上锁状态，锁不可取下
- 所有上过锁的箱子不能再次上锁
- `unlockChance` 固定为 0（锁本身不参与开锁概率计算）

---

## 钥匙材质体系

> 来源：`DrawKeySettings()` 中的 MaterialTier 枚举和默认开锁概率

| MaterialTier | 默认开锁概率 | 对应箱子材质 |
|---|---|---|
| Wood（木） | 10% | Wood |
| Stone（石） | 15% | Wood |
| Iron（铁） | 20% | Iron |
| Brass（铜） | 25% | Iron |
| Steel（钢） | 30% | Iron |
| Gold（金） | 40% | Special |

开锁规则：
- 最终概率 = 钥匙概率 + 箱子概率
- 成功：保留钥匙
- 失败：消耗钥匙
- 概率来自 `KeyLockData.GetDefaultUnlockChanceByTier()`

---

## 文件命名规范

| SO 类型 | 文件前缀 | 示例 |
|---|---|---|
| ToolData | `Tool` | `Tool_0_Axe_0.asset` |
| WeaponData | `Weapon` | `Weapon_200_Sword_0.asset` |
| SeedData | `Seed` | `Seed_1000_大蒜种子.asset` |
| CropData | `Crop` | `Crop_1100_大蒜.asset` |
| WitheredCropData | `Crop_Withered` | `Crop_Withered_1150_大蒜.asset` |
| SaplingData | `Sapling` | `Sapling_1200_橡树苗.asset` |
| WorkstationData | `Workstation` | `Workstation_1300_铁砧.asset` |
| StorageData | `Storage` | `Storage_1400_木箱.asset` |
| LockData | `Lock` | `Lock_1410_木锁.asset` |
| KeyData | `Key` | `Key_1420_木钥匙.asset` |
| InteractiveDisplayData | `Display` | `Display_1500_告示牌.asset` |
| SimpleEventData | `Event` | `Event_1600_传送点.asset` |
| MaterialData | `Material` | `Material_3200_Wood.asset` |
| PotionData | `Potion` | `Potion_4000_HP药水.asset` |
| FoodData | `Food` | `Food_5000_烤鱼.asset` |
| FurnitureData | `Furniture` | `Furniture_6000_木桌.asset` |
| SpecialData | `Special` | `Special_7000_传送石.asset` |
| EquipmentData | `Equipment` | `Equipment_8000_铁头盔.asset` |
| ItemData | `Item` | `Item_0_通用物品.asset` |

---

## 已使用 ID 登记表

> 从 `Assets/111_Data/Items/` 实际 SO 资产扫描，最后更新：2026-02-15
> 每次批量生成 SO 后应同步更新此表。

### 工具（ToolData）— 0-18 已用，下一个可用：19

| ID | 文件名 | 说明 |
|---|---|---|
| 0 | Tool_0_Axe_0 | 斧头 Lv0 |
| 1 | Tool_1_Axe_1 | 斧头 Lv1 |
| 2 | Tool_2_Axe_2 | 斧头 Lv2 |
| 3 | Tool_3_Axe_3 | 斧头 Lv3 |
| 4 | Tool_4_Axe_4 | 斧头 Lv4 |
| 5 | Tool_5_Axe_5 | 斧头 Lv5 |
| 6 | Tool_6_Pickaxe_0 | 镐子 Lv0 |
| 7 | Tool_7_Pickaxe_1 | 镐子 Lv1 |
| 8 | Tool_8_Pickaxe_2 | 镐子 Lv2 |
| 9 | Tool_9_Pickaxe_3 | 镐子 Lv3 |
| 10 | Tool_10_Pickaxe_4 | 镐子 Lv4 |
| 11 | Tool_11_Pickaxe_5 | 镐子 Lv5 |
| 12 | Tool_12_Hoe_0 | 锄头 Lv0 |
| 13 | Tool_13_Hoe_1 | 锄头 Lv1 |
| 14 | Tool_14_Hoe_2 | 锄头 Lv2 |
| 15 | Tool_15_Hoe_3 | 锄头 Lv3 |
| 16 | Tool_16_Hoe_4 | 锄头 Lv4 |
| 17 | Tool_17_Hoe_5 | 锄头 Lv5 |
| 18 | Tool_18_WateringCan_0 | 水壶 Lv0 |

### 武器（WeaponData）— 200-205 已用，下一个可用：206

| ID | 文件名 | 说明 |
|---|---|---|
| 200 | Weapon_200_Sword_0 | 剑 Lv0 |
| 201 | Weapon_201_Sword_1 | 剑 Lv1 |
| 202 | Weapon_202_Sword_2 | 剑 Lv2 |
| 203 | Weapon_203_Sword_3 | 剑 Lv3 |
| 204 | Weapon_204_Sword_4 | 剑 Lv4 |
| 205 | Weapon_205_Sword_5 | 剑 Lv5 |

### 种子（SeedData）— 1000-1006 已用，下一个可用：1007

| ID | 文件名 | 说明 |
|---|---|---|
| 1000 | Seed_1000_大蒜 | 大蒜种子 |
| 1001 | Seed_1001_生菜 | 生菜种子 |
| 1002 | Seed_1002_花椰菜 | 花椰菜种子 |
| 1003 | Seed_1003_卷心菜 | 卷心菜种子 |
| 1004 | Seed_1004_西兰花 | 西兰花种子 |
| 1005 | Seed_1005_甜菜 | 甜菜种子 |
| 1006 | Seed_1006_胡萝卜 | 胡萝卜种子 |

### 作物（CropData）— 无资产

> 尚未创建任何 CropData SO。下一个可用：1100

### 枯萎作物（WitheredCropData）— 无资产

> 尚未创建任何 WitheredCropData SO。下一个可用：1150

### 树苗（SaplingData）— 1200-1202 已用，下一个可用：1203

| ID | 文件名 | 说明 |
|---|---|---|
| 1200 | Sapling_1200_M1树苗 | M1 树苗 |
| 1201 | Sapling_1201_M2树苗 | M2 树苗 |
| 1202 | Sapling_1202_M3树苗 | M3 树苗 |

### 存储（StorageData）— 1400-1403 已用，下一个可用：1404

| ID | 文件名 | 说明 |
|---|---|---|
| 1400 | Storage_1400_小木箱子_0 | 小木箱子 |
| 1401 | Storage_1401_大木箱子_0 | 大木箱子 |
| 1402 | Storage_1402_小铁箱子_0 | 小铁箱子 |
| 1403 | Storage_1403_大铁箱子_0 | 大铁箱子 |

### 锁（LockData）— 1410-1411 已用，下一个可用：1412

| ID | 文件名 | 说明 |
|---|---|---|
| 1410 | Item_1410_Lock_1_0 | 木锁 |
| 1411 | Item_1411_Lock_5_0 | 铁锁 |

> 注意：锁的文件名前缀仍为旧格式 `Item_`，后续批量生成将使用 `Lock_` 前缀

### 钥匙（KeyData）— 1420-1425 已用，下一个可用：1426

| ID | 文件名 | 说明 |
|---|---|---|
| 1420 | Key_1420_Key_0 | 钥匙 0 |
| 1421 | Key_1421_Key_1 | 钥匙 1 |
| 1422 | Key_1422_Key_2 | 钥匙 2 |
| 1423 | Key_1423_Key_3 | 钥匙 3 |
| 1424 | Key_1424_Key_4 | 钥匙 4 |
| 1425 | Key_1425_Key_5 | 钥匙 5 |

### 材料（MaterialData）— 3000-3002, 3100-3102, 3200-3201 已用

| ID | 文件名 | 子类型 | 说明 |
|---|---|---|---|
| 3000 | Material_3000_黄铜矿 | Ore | 黄铜矿 |
| 3001 | Material_3001_生铁矿 | Ore | 生铁矿 |
| 3002 | Material_3002_黄金矿 | Ore | 黄金矿 |
| 3100 | Material_3100_黄铜锭 | Ingot | 黄铜锭 |
| 3101 | Material_3101_生铁锭 | Ingot | 生铁锭 |
| 3102 | Material_3102_黄金锭 | Ingot | 黄金锭 |
| 3200 | Material_3200_Wood | Natural | 木材 |
| 3201 | Material_3201_石料 | Natural | 石料 |

> 下一个可用：矿石 3003，锭 3103，自然材料 3202

### 装备（EquipmentData）— 8300, 8400 已用

| ID | 文件名 | 子类型 | 槽位 | 说明 |
|---|---|---|---|---|
| 8300 | Equipment_8300_足力健 | Shoes | 3 | 鞋子 |
| 8400 | Equipment_8400_恶魔戒指 | Ring | 4/5 | 戒指 |

> 下一个可用：头盔 8000，盔甲 8100，裤子 8200，鞋子 8301，戒指 8401

### 空目录（尚无资产）

| 类型 | 目录 | 状态 |
|---|---|---|
| CropData | Crops/ | 空 |
| WitheredCropData | Crops_Withered/ | 空 |
| FoodData | Foods/ | 空 |
| PotionData | Potions/ | 空 |
| WorkstationData | Placeable/Workstations/ | 目录不存在 |
| InteractiveDisplayData | Placeable/Displays/ | 目录不存在 |
| SimpleEventData | Placeable/Events/ | 目录不存在 |
| FurnitureData | Furniture/ | 目录不存在 |
| SpecialData | Special/ | 目录不存在 |

---

## 预留段（尚未设计）

| 段 | 用途 | 状态 |
|---|---|---|
| 2XXX | 动物产品 | 预留 |
| 33XX | 怪物掉落 | 预留（MaterialData 子分类） |
| 51XX | 高级食物 | 预留（FoodData 子分类） |
| 8000-8099 | 头盔 | 预留（EquipmentData 子分类） |
| 8100-8199 | 盔甲 | 预留（EquipmentData 子分类） |
| 8200-8299 | 裤子 | 预留（EquipmentData 子分类） |
| 8500-8599 | 饰品 | 预留（EquipmentData 子分类） |

---

## 变更审计日志

| 日期 | 变更内容 | 原因 |
|---|---|---|
| 2026-02-15 | 文档创建，从工具 SubTypeStartIDs 提取完整映射 | 建立唯一权威源 |
| 2026-02-15 | WitheredCropData 从 1200 改为 1150 | 与 SaplingData(1200) 冲突，用户决定 Crop 和 WitheredCrop 共享 11XX 段 |
| 2026-02-15 | 文件前缀 WCrop 改为 Crop_Withered | 文件夹排序时与 Crop 贴在一起 |
| 2026-02-15 | 新增「已使用 ID 登记表」区域 | 记录所有已创建 SO 的 ID 占用情况，方便后续分配 |
| 2026-02-15 | 补充装备 6 个子分类 ID 范围和槽位映射 | 深度审查发现工具中有完整定义但规范文档遗漏 |
| 2026-02-15 | 材料子分类矿石/锭从"预留"改为"已有" | 实际已有 3000-3002 矿石和 3100-3102 锭资产 |
| 2026-02-15 | 新增锁的 ID 细分（木锁/铁锁/特殊锁） | 工具 DrawLockSettings 中有明确定义 |
| 2026-02-15 | 新增钥匙材质体系（6 种材质+默认开锁概率） | 工具 DrawKeySettings 中有完整体系 |
| 2026-02-15 | 工具代码同步：WitheredCropData StartID 1200→1150 | 规范与代码统一 |
| 2026-02-15 | 工具代码同步：GetFilePrefix WCrop→Crop_Withered | 规范与代码统一 |
| 2026-02-15 | 工具代码同步：GetFilePrefix 新增 LockData→"Lock" | 之前走默认"Item"，现在有独立前缀 |
| 2026-02-15 | 文档路径从子目录迁移到 `SO设计系统与工具/ID分配规范.md` | 用户手动移动文件 |
