# 农作物控制器设计与SO迭代 - 设计文档

**创建日期**: 2026-02-11
**工作区**: `.kiro/specs/农田系统/10.0.1农作物设计与完善/`
**依据**: requirements.md + 全面分析文档V2 + 锐评002审视报告

---

## 一、SO 继承体系

### 目标继承结构

```
ItemData
├── SeedData              (种子/种子袋，不可食用)
├── FoodData              (食物基类)
│   ├── CropData          (正常作物，可食用)
│   └── WitheredCropData  (枯萎作物，可食用，负面buff)
├── ToolData              (工具)
└── ...
```

### 1.1 SeedData 新增字段

```csharp
[Header("=== 种子袋配置 ===")]
[Tooltip("每袋种子数量")]
[Range(1, 20)]
public int seedsPerBag = 5;

[Tooltip("未打开保质期（天）")]
[Range(1, 28)]
public int shelfLifeClosed = 7;

[Tooltip("打开后保质期（天）")]
[Range(1, 14)]
public int shelfLifeOpened = 2;

[Tooltip("已打开状态的图标")]
public Sprite iconOpened;

[Header("=== 枯萎样式 ===")]
[Tooltip("枯萎阶段Sprite（至少1个，可扩展为多个）")]
public Sprite[] witheredStageSprites;
```

### 1.2 CropData 重构

```csharp
// 改为继承 FoodData
[CreateAssetMenu(fileName = "Crop_New", menuName = "Farm/Items/Crop", order = 2)]
public class CropData : FoodData
{
    [Header("=== 作物专属属性 ===")]
    [Tooltip("对应的种子ID")]
    public int seedID;

    [Tooltip("收获经验值")]
    public int harvestExp = 10;

    [Tooltip("对应的枯萎作物ID（WitheredCropData）")]
    public int witheredCropID;

    // 移除 canBeCrafted、usedInRecipes、qualityInfo（继承自 FoodData 的字段已覆盖）
}
```

### 1.3 WitheredCropData 新建

```csharp
[CreateAssetMenu(fileName = "WCrop_New", menuName = "Farm/Items/Withered Crop", order = 4)]
public class WitheredCropData : FoodData
{
    [Header("=== 枯萎作物属性 ===")]
    [Tooltip("对应的正常作物ID")]
    public int normalCropID;

    [Tooltip("对应的种子ID")]
    public int seedID;
}
```

### 1.4 ID 范围

| SO 类型 | ID 范围 | 示例 |
|---------|---------|------|
| SeedData | 10XX | 1001 番茄种子 |
| CropData | 11XX | 1101 番茄 |
| WitheredCropData | 12XX | 1201 枯萎番茄 |
| 腐烂的食物 | 5999 | 5999 腐烂的食物（FoodData，全局唯一） |

---

## 二、CropController 重构

### 2.1 状态枚举

```csharp
public enum CropState
{
    Growing,           // 生长中（Stage 0 ~ N-2）
    Mature,            // 成熟（最后阶段，可收获）
    WitheredImmature,  // 未成熟枯萎（缺水/过季，锄头清除，无产出）
    WitheredMature     // 成熟枯萎（过熟/过季，可收获枯萎作物）
}
```

### 2.2 接口实现

CropController 实现 `IInteractable` 接口（不实现 IResourceNode）：

```csharp
public class CropController : MonoBehaviour, IPersistentObject, IInteractable
{
    // IInteractable
    public int InteractionPriority => 40;
    public float InteractionDistance => 1.2f;

    public bool CanInteract(InteractionContext context)
    {
        return state == CropState.Mature || state == CropState.WitheredMature;
    }

    public void OnInteract(InteractionContext context)
    {
        Harvest(context);
    }

    public string GetInteractionHint(InteractionContext context)
    {
        if (state == CropState.Mature) return "收获";
        if (state == CropState.WitheredMature) return "收获（枯萎）";
        return "";
    }
}
```

### 2.3 交互检测方式

使用 BoxCollider2D (IsTrigger = true)，不注册到 ResourceNodeRegistry。

理由（锐评002确认）：
- ResourceNodeRegistry 是线性遍历 O(N)，不适合大量作物
- BoxCollider2D 利用 Unity 物理引擎的 QuadTree，查询效率 O(logN)
- HandleRightClickAutoNav 已有 Physics2D 检测 IInteractable 的链路

### 2.4 OnDayChanged 逻辑

```
OnDayChanged(year, day, totalDays):
    if state == Growing:
        检查浇水 → 生长或停滞
        检查缺水天数 → 可能变为 WitheredImmature
        检查是否达到成熟 → 转为 Mature
    elif state == Mature:
        if (!seedData.isReHarvestable):
            daysSinceMature++
            if daysSinceMature >= 2:
                state = WitheredMature
                UpdateVisuals()
```

### 2.5 过季枯萎

```
OnSeasonChanged(newSeason):
    if seedData.season != Season.AllSeason && seedData.season != newSeason:
        if state == Mature:
            state = WitheredMature
        elif state == Growing:
            state = WitheredImmature
        UpdateVisuals()
```

### 2.6 收获逻辑

```
Harvest(context):
    if state == Mature:
        产出 CropData 物品
        数量 = Random(harvestAmountRange)
        品质 = 随机判定
        if isReHarvestable:
            重置到指定阶段，state = Growing
        else:
            销毁 GameObject
    elif state == WitheredMature:
        产出 WitheredCropData 物品
        数量 = Random(harvestAmountRange)
        品质 = Normal（固定）
        销毁 GameObject（枯萎不可重复收获）
```

### 2.7 视觉更新

```
UpdateVisuals():
    if state == WitheredImmature || state == WitheredMature:
        // 使用枯萎 Sprite
        int idx = Clamp(currentStage, 0, witheredStageSprites.Length - 1)
        sprite = seedData.witheredStageSprites[idx]
        color = WitheredColor
    else:
        // 使用正常 Sprite
        sprite = seedData.growthStageSprites[currentStage]
        color = White
        if state == Mature: 启用成熟闪烁
```

---

## 三、种子袋保质期系统

### 3.1 动态属性 Key 定义

| Key | 类型 | 说明 |
|-----|------|------|
| `bag_opened` | bool | 是否已打开 |
| `bag_remaining` | int | 袋内剩余种子数 |
| `shelf_expire_day` | int | 过期的游戏总天数 |

### 3.2 种子袋初始化

购买/获得种子袋时：
```
item.SetProperty("bag_opened", false)
item.SetProperty("bag_remaining", seedData.seedsPerBag)
item.SetProperty("shelf_expire_day", currentTotalDays + seedData.shelfLifeClosed)
```

### 3.3 打开种子袋

触发条件：背包右键 或 种植时自动打开

```
OpenSeedBag(item):
    if item.GetPropertyBool("bag_opened"): return  // 已打开
    
    item.SetProperty("bag_opened", true)
    
    // 保质期重算：Min(当前剩余, 打开后最大)
    int currentExpire = item.GetPropertyInt("shelf_expire_day")
    int remainingDays = currentExpire - currentTotalDays
    int newExpire = currentTotalDays + Min(remainingDays, seedData.shelfLifeOpened)
    item.SetProperty("shelf_expire_day", newExpire)
```

### 3.4 种植消耗

```
PlantSeed(item):
    if !item.GetPropertyBool("bag_opened"):
        OpenSeedBag(item)  // 自动打开
    
    int remaining = item.GetPropertyInt("bag_remaining")
    remaining -= 1
    
    if remaining <= 0:
        从背包移除该物品
    else:
        item.SetProperty("bag_remaining", remaining)
```

### 3.5 日结保质期检查

InventoryService 订阅 OnDayChanged：

```
CheckShelfLife(totalDays):
    遍历所有背包槽位:
        if item.HasProperty("shelf_expire_day"):
            if totalDays >= item.GetPropertyInt("shelf_expire_day"):
                // 过期 → 替换为腐烂食物
                ReplaceWithRottenFood(slotIndex)
```

### 3.6 腐烂食物生成（关键）

```
ReplaceWithRottenFood(slotIndex):
    // 创建腐烂食物 InventoryItem
    var rottenItem = new InventoryItem(5999, 0, 1)
    // ★ 关键：不设置任何动态属性，确保 properties.Count == 0
    // 这样 CanStackWith() 才能返回 true，腐烂食物才能堆叠
    
    替换槽位物品为 rottenItem
    尝试与已有腐烂食物堆叠
```

---

## 四、CropInstanceData 扩展

### 新增字段

```csharp
/// <summary>
/// 成熟后经过的天数（用于过熟枯萎判断）
/// </summary>
public int daysSinceMature;
```

### CropSaveData 新增字段

```csharp
public int daysSinceMature;
```

---

## 五、收获交互流程

### 右键点击成熟作物

```
玩家右键点击
  ↓
HandleRightClickAutoNav() 检测 Physics2D
  ↓
找到 CropController (IInteractable, BoxCollider2D Trigger)
  ↓
CanInteract() == true（Mature 或 WitheredMature）
  ↓
计算距离
  ├─ 近距离 → 直接 OnInteract() → collect 动画 → Harvest()
  └─ 远距离 → FollowTarget() 导航 → 到达后 OnInteract()
```

### 右键点击未成熟作物

```
玩家右键点击
  ↓
HandleRightClickAutoNav() 检测 Physics2D
  ↓
找到 CropController (IInteractable)
  ↓
CanInteract() == false（Growing 或 WitheredImmature）
  ↓
不作为 IInteractable 处理 → 走普通导航逻辑
```

### 枯萎未成熟作物清除

```
玩家左键使用锄头
  ↓
检测到 WitheredImmature 作物
  ↓
播放锄头动画 → 清除作物 → 耕地恢复为空
```

---

## 六、与现有系统的兼容性

### 存档兼容

- CropInstanceData 新增字段默认值为 0，旧存档加载不受影响
- CropSaveData 新增字段默认值为 0
- 种子袋动态属性在旧存档中不存在，加载时默认为"未打开"

### CropData 继承重构风险

- CropData 从 ItemData 改为 FoodData，需要检查所有引用 CropData 的代码
- FoodData 继承自 ItemData，所以 CropData 仍然是 ItemData 的子类
- 现有的 `CropData cropData = database.GetItem(id) as CropData` 仍然有效
- 需要重新配置现有 CropData SO 的 FoodData 字段（energyRestore 等）

---

## 七、正确性属性

### P1：种子袋保质期单调递减

对于任意种子袋物品，其 `shelf_expire_day` 只能在打开时重算（可能减少），不能增加。

### P2：腐烂食物无动态属性

所有腐烂食物的 InventoryItem 实例，`properties.Count == 0`，确保可堆叠。

### P3：作物状态转换合法性

CropState 只能按以下路径转换：
- Growing → Mature（达到最后阶段）
- Growing → WitheredImmature（缺水/过季）
- Mature → WitheredMature（过熟/过季）
- Mature → Growing（可重复收获作物收割后重置）

不允许：WitheredImmature → Growing、WitheredMature → Mature

### P4：收获产出一致性

- Mature 状态收获 → 产出 CropData 物品
- WitheredMature 状态收获 → 产出 WitheredCropData 物品
- Growing/WitheredImmature 状态 → 不可收获

### P5：过季枯萎全覆盖

季节切换时，所有非 AllSeason 且不匹配新季节的作物，无论当前状态，都必须转为枯萎状态。

---

## 八、测试框架

使用 Unity Test Framework (NUnit) 进行 EditMode 测试。

测试重点：
1. CropState 状态转换合法性
2. 种子袋保质期计算
3. 腐烂食物堆叠性
4. 收获产出正确性
5. 过季枯萎覆盖率
