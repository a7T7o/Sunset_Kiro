# 篇章三：农业系统详细策划

## 一、作物数据表（基于 SeedData 框架）

所有作物对应的 `SeedData` 必须包含以下字段（已有代码支持）：
- `season`：可种植季节（Spring/Summer/Fall/Winter/AllSeason）
- `cropPrefab`：关联的作物预制体，其中挂载 `CropController`，配置 `CropData`（成熟后收获的物品）和 `WitheredCropData`（枯萎收获物品）
- `isReHarvestable`：是否可重复收获
- `reHarvestDays`：重复收获间隔天数（若 `isReHarvestable = true`）
- `maxHarvestCount`：最大收获次数（若可重复收获）
- `seedsPerBag`：每袋种子数量
- `shelfLifeClosed`：未开封保质期（游戏天数）
- `shelfLifeOpened`：开封后保质期（游戏天数）
- `iconOpened`：开封后图标（现有素材可复用）
- `needsTrellis`：是否需要支架（仅西兰花需要）
- `plantingExp`：种植时获得的经验（采集技能）
- `harvestingExp`：收获时获得的经验（采集技能）

另外，作物成熟时掉落 `CropData`（继承自 `FoodData`），需要设计其 `energyRestore`、`healthRestore`、`sellPrice` 等。

以下设计基于现有美术资产（8种作物）并补充冬季2种（需要自制或从待整合包提取，标注“需要新制作”）。

### 1.1 春季作物（3种）

#### 大蒜 (Garlic)
| 字段 | 值 | 说明 |
|------|----|------|
| `itemID` | 1001 | 种子ID |
| `season` | Spring | 春季 |
| `cropPrefab` | Crops/Spring/Garlic | 预制体（已有） |
| `isReHarvestable` | false | 单次收获 |
| `reHarvestDays` | - | - |
| `maxHarvestCount` | - | - |
| `seedsPerBag` | 20 | 每袋20粒 |
| `shelfLifeClosed` | 365 | 一年 |
| `shelfLifeOpened` | 180 | 半年 |
| `needsTrellis` | false | - |
| `plantingExp` | 5 | - |
| `harvestingExp` | 8 | - |
| **成熟作物（CropData）** | 大蒜（物品ID 5101） |  |
| `energyRestore` | 15 | 食用恢复精力 |
| `healthRestore` | 0 | - |
| `sellPrice` | 6 | 基础售价 |

#### 花椰菜 (Cauliflower)
| 字段 | 值 |
|------|----|
| `itemID` | 1002 |
| `season` | Spring |
| `cropPrefab` | Crops/Spring/Cauliflower |
| `isReHarvestable` | false |
| `seedsPerBag` | 10 |
| `shelfLifeClosed` | 365 |
| `shelfLifeOpened` | 150 |
| `plantingExp` | 8 |
| `harvestingExp` | 12 |
| **成熟作物** | 花椰菜（ID 5102） |
| `energyRestore` | 25 |
| `sellPrice` | 15 |

#### 生菜 (Lettuce)
| 字段 | 值 |
|------|----|
| `itemID` | 1003 |
| `season` | Spring |
| `cropPrefab` | Crops/Spring/Lettuce |
| `isReHarvestable` | true |
| `reHarvestDays` | 2 |
| `maxHarvestCount` | 5 |
| `seedsPerBag` | 25 |
| `shelfLifeClosed` | 200 |
| `shelfLifeOpened` | 90 |
| `plantingExp` | 4 |
| `harvestingExp` | 6 |
| **成熟作物** | 生菜（ID 5103） |
| `energyRestore` | 10 |
| `sellPrice` | 4 |

### 1.2 夏季作物（3种）

#### 卷心菜 (Cabbage)
| 字段 | 值 |
|------|----|
| `itemID` | 1101 |
| `season` | Summer |
| `cropPrefab` | Crops/Summer/Cabbage |
| `isReHarvestable` | false |
| `seedsPerBag` | 15 |
| `shelfLifeClosed` | 365 |
| `shelfLifeOpened` | 120 |
| `plantingExp` | 6 |
| `harvestingExp` | 10 |
| **成熟作物** | 卷心菜（ID 5201） |
| `energyRestore` | 20 |
| `sellPrice` | 12 |

#### 西兰花 (Broccoli)
| 字段 | 值 |
|------|----|
| `itemID` | 1102 |
| `season` | Summer |
| `cropPrefab` | Crops/Summer/Broccoli |
| `isReHarvestable` | true |
| `reHarvestDays` | 4 |
| `maxHarvestCount` | 3 |
| `seedsPerBag` | 12 |
| `shelfLifeClosed` | 300 |
| `shelfLifeOpened` | 100 |
| `needsTrellis` | true |
| `plantingExp` | 7 |
| `harvestingExp` | 9 |
| **成熟作物** | 西兰花（ID 5202） |
| `energyRestore` | 18 |
| `sellPrice` | 10 |

#### 辣椒 (Pepper) —— 需要新制作（但可借用现有图标）
| 字段 | 值 |
|------|----|
| `itemID` | 1103 |
| `season` | Summer |
| `cropPrefab` | 需要新建（建议使用Tiny Wonder Farm的plants素材调整） |
| `isReHarvestable` | true |
| `reHarvestDays` | 3 |
| `maxHarvestCount` | 4 |
| `seedsPerBag` | 18 |
| `shelfLifeClosed` | 250 |
| `shelfLifeOpened` | 80 |
| `needsTrellis` | false |
| `plantingExp` | 5 |
| `harvestingExp` | 7 |
| **成熟作物** | 辣椒（ID 5203） |
| `energyRestore` | 12 |
| `sellPrice` | 8 |
| **备注** | 可作为烹饪香料，解锁辣味食谱 |

### 1.3 秋季作物（3种）

#### 胡萝卜 (Carrot)
| 字段 | 值 |
|------|----|
| `itemID` | 1201 |
| `season` | Fall |
| `cropPrefab` | Crops/Fall/Carrot |
| `isReHarvestable` | false |
| `seedsPerBag` | 20 |
| `shelfLifeClosed` | 365 |
| `shelfLifeOpened` | 150 |
| `plantingExp` | 5 |
| `harvestingExp` | 8 |
| **成熟作物** | 胡萝卜（ID 5301） |
| `energyRestore` | 15 |
| `sellPrice` | 7 |

#### 甜菜 (Beet)
| 字段 | 值 |
|------|----|
| `itemID` | 1202 |
| `season` | Fall |
| `cropPrefab` | Crops/Fall/Beet |
| `isReHarvestable` | false |
| `seedsPerBag` | 15 |
| `shelfLifeClosed` | 365 |
| `shelfLifeOpened` | 120 |
| `plantingExp` | 6 |
| `harvestingExp` | 10 |
| **成熟作物** | 甜菜（ID 5302） |
| `energyRestore` | 20 |
| `sellPrice` | 11 |
| **备注** | 可制糖（需要新增加工配方） |

#### 南瓜 (Pumpkin) —— 需要新制作
| 字段 | 值 |
|------|----|
| `itemID` | 1203 |
| `season` | Fall |
| `cropPrefab` | 需要新建（建议使用Tiny Wonder Farm的plants素材） |
| `isReHarvestable` | false |
| `seedsPerBag` | 8 |
| `shelfLifeClosed` | 500 |
| `shelfLifeOpened` | 200 |
| `plantingExp` | 10 |
| `harvestingExp` | 15 |
| **成熟作物** | 南瓜（ID 5303） |
| `energyRestore` | 30 |
| `sellPrice` | 25 |
| **备注** | 大型作物，收获时可能多个？可通过 `minAmount/maxAmount` 在掉落表中配置 |

### 1.4 冬季作物（2种，需要新制作）

#### 雪莲 (Snow Lotus) —— 药用
| 字段 | 值 |
|------|----|
| `itemID` | 1301 |
| `season` | Winter |
| `cropPrefab` | 需要新建（可使用001_OK中的自然素材拼凑） |
| `isReHarvestable` | false |
| `seedsPerBag` | 10 |
| `shelfLifeClosed` | 200 |
| `shelfLifeOpened` | 60 |
| `plantingExp` | 8 |
| `harvestingExp` | 12 |
| **成熟作物** | 雪莲（ID 5401） |
| `energyRestore` | 5 |
| `healthRestore` | 30 | 雪莲可制药，恢复生命 |
| `sellPrice` | 20 |

#### 冰晶莓 (Ice Berry) —— 可重复收获
| 字段 | 值 |
|------|----|
| `itemID` | 1302 |
| `season` | Winter |
| `cropPrefab` | 需要新建 |
| `isReHarvestable` | true |
| `reHarvestDays` | 5 |
| `maxHarvestCount` | 3 |
| `seedsPerBag` | 12 |
| `shelfLifeClosed` | 150 |
| `shelfLifeOpened` | 40 |
| `plantingExp` | 6 |
| `harvestingExp` | 8 |
| **成熟作物** | 冰晶莓（ID 5402） |
| `energyRestore` | 18 |
| `sellPrice` | 10 |

---

## 二、种植策略设计

### 2.1 季节搭配策略
- **春季**：以生菜作为快速循环作物，可连续收获5次，适合前期积累经验和资金；大蒜和花椰菜作为基础作物和现金作物搭配。
- **夏季**：辣椒连续收获，卷心菜单次高收益，西兰花需支架（需提前制作支架，消耗木材），可形成多层次种植。
- **秋季**：南瓜生长周期长但价值高，适合最后一批；胡萝卜和甜菜作为日常烹饪原料。
- **冬季**：作物稀少，雪莲用于制药，冰晶莓提供少量食物，主要依赖秋季储备。

**轮作收益最大化**：
- 春15日播种夏季作物（利用季节过渡最后几天？但季节变更作物会枯萎，不能跨季）。必须严格按季节种植。
- 但可通过温室（后期解锁）实现跨季种植（需要新建筑，但可用现有房屋改造）。

### 2.2 品质提升机制
利用 `ItemQuality` 系统，作物收获时根据以下因素决定品质：
- **土壤肥力**（可通过施肥提升，需要新机制）：使用肥料（如鱼、骨头制作的肥料）提升基础品质等级。
- **浇水状态**：每日保持湿润，成熟时有概率提升品质。
- **技能等级**：采集技能等级每2级，收获时品质提升概率+5%。
- **季节适宜性**：所有作物在对应季节种植，基础品质为Normal；若使用温室或反季节种植（需解锁），品质会下降一档。

品质计算公式（策划参考）：
```
最终品质 = 基础品质 + 肥力修正 + 浇水修正 + 技能修正
```
取值：0=Normal, 1=Rare, 2=Epic, 3=Legendary。通过随机阈值决定具体品质。

### 2.3 种植密度与支架系统
- 普通作物：每格1株。
- 需支架作物（西兰花）：占用1格，但必须紧邻已放置的支架（支架是一种放置物，可单独制作）。支架本身占1格，西兰花种在相邻格。支架可重复使用。

**支架配方**：木材5 → 支架1（可放置，耐久100天，冬季会损坏？需要设计）。

---

## 三、农田升级路线（与自动化三阶段对应）

农田本身不是工作台，但可以通过外围设施实现自动化。我们结合已有农田视觉系统（四层Tilemap）设计升级。

### Phase 1：手搓期（春-夏初）
- **农田类型**：基础农田（中心层使用`Farm_C0.png`干燥土壤）。
- **浇水**：手动用水壶每格浇水（`FarmTileManager.SetWatered()`），每次消耗精力。
- **收获**：手动右键点击作物（或使用镰刀范围收割，镰刀已有 `effectRadius` 字段）。
- **肥料**：无（后期可通过撒肥器实现，暂不开放）。
- **压迫感**：玩家每天必须手动浇水所有农田，时间紧迫。

### Phase 2：半自动期（夏末-秋，解锁“洒水器”）
**解锁条件**：夏季主线完成，铁匠铺重开，获得铁质工具；记忆闪回解锁“基础洒水器”图纸。
- **洒水器**（需要自制美术，或利用现有道具改色）：
  - 放置型物品，继承 `PlaceableItemData`，ID范围 1700-1709。
  - 每台洒水器覆盖3×3区域（9格）。
  - 需要每天手动向洒水器“灌水”（消耗精力，类似水壶但一次灌满9格），之后洒水器会在当天自动灌溉覆盖区域。
  - 灌水时需背包里有水（水壶蓄水机制，需要实现但可设计为：水壶右键洒水器，扣除水壶内的水）。
- **体验**：减少单格浇水时间，但仍需手动灌水，且需规划洒水器布局。

### Phase 3：全自动期（冬，解锁“自动灌溉系统”）
**解锁条件**：冬季主线推进，修复灰碑核心，记忆闪回解锁“自动灌溉控制器”。
- **自动灌溉系统**：
  - 核心建筑：水泵（需要放置在水边）+ 管道（连接水泵与洒水器）+ 控制器。
  - 水泵每早自动抽水，通过管道输送到所有连接的洒水器，洒水器自动灌溉覆盖区域。
  - 玩家只需铺设好管道网络，无需每天操作。
- **实现方式**：可利用现有 `PlacementManager` 放置管道（新美术需要自制，但可简化：用现有木材架代替），控制器作为特殊工作台，管理范围内的洒水器。
- **解放感**：农田彻底自动化，玩家只需播种和收获，精力用于地牢和社交。

### 农田扩展：开垦更多土地
- 初始：村落周围有3块小农田（共12格）。
- 随着村落复兴，可清理废墟、砍伐树木，开垦新农田。每开垦1格需要消耗锄头耐久和精力，并可能需要清理杂草/石头（用斧/镐）。
- 最大可开垦至50格（满足全自动需求）。

---

## 四、与天气系统的联动设计

### 4.1 夏季枯萎天（可逆）
- 天气：夏季第8、14、20天触发 `PlantsWitherEvent`。
- 效果：所有未浇水的作物状态变为 `WitheredImmature`（未成熟）或 `WitheredMature`（成熟）。
- 恢复：次日如果下雨（或玩家浇水），作物会从枯萎状态恢复到正常生长状态（生长阶段不变）。枯萎成熟的作物若当天未收获，次日恢复后仍可收获（但可能损失一次收获机会）。
- **设计目的**：迫使玩家在枯萎日必须浇水，否则收成受损。

### 4.2 秋季枯萎天（不可逆）
- 天气：秋季第6、14、22天触发 `PlantsWitherEvent`（且配置为不可恢复，因为 `WeatherSystem` 中秋季无 `PlantsRecoverEvent`）。
- 效果：所有作物直接死亡（变为枯萎状态，且不可恢复，收获只能得到 `WitheredCropData`，价值低且有负面效果）。
- 应对：玩家必须在枯萎日前收获所有成熟作物，未成熟的只能放弃。
- **设计目的**：制造紧张感，逼迫玩家精确规划种植时间，避免在枯萎日有大量未成熟作物。

### 4.3 冬季冰雪
- 下雪日：树苗休眠，但农田作物冬季不能种植（除非温室），所以农田无作物。
- 融化日：无直接影响。

### 4.4 作物与季节的适应性
- 如果玩家在错误季节种植（例如春季种夏季作物），作物会在季节变更当天枯萎（变为不可逆枯萎）。这是由 `CropController` 的季节检查实现的（每日检查，非适宜季节直接枯萎）。

---

## 五、作物收益分析表（每日收益）

基于前面设定的作物数据，计算每格农田每日平均收益（不考虑品质，只算基础售价）。用于数值平衡验证。

| 作物 | 生长天数 | 收获次数 | 总售价 | 每日收益 | 种子成本 | 净每日收益 | 备注 |
|------|----------|----------|--------|----------|----------|------------|------|
| 大蒜 | 4 | 1 | 6 | 1.5 | 种子价约0.3/粒（6/20=0.3） | 1.2 | - |
| 花椰菜 | 8 | 1 | 15 | 1.875 | 种子价约1.5/粒（15/10=1.5） | 0.375 | 低效但可作为进阶材料 |
| 生菜 | 3+2×4=11 | 5 | 4×5=20 | 1.82 | 种子价0.24/粒（6/25=0.24） | 1.58 | 适合前期快速回本 |
| 卷心菜 | 6 | 1 | 12 | 2.0 | 种子价0.8/粒（12/15=0.8） | 1.2 | - |
| 西兰花 | 4+4×2=12 | 3 | 10×3=30 | 2.5 | 种子价0.83/粒（10/12=0.83） | 1.67 | 需支架成本，但收益高 |
| 辣椒 | 5+3×3=14 | 4 | 8×4=32 | 2.29 | 种子价0.44/粒（8/18≈0.44） | 1.85 | 高收益 |
| 胡萝卜 | 5 | 1 | 7 | 1.4 | 种子价0.35/粒（7/20=0.35） | 1.05 | - |
| 甜菜 | 7 | 1 | 11 | 1.57 | 种子价0.73/粒（11/15≈0.73） | 0.84 | 制糖后收益更高 |
| 南瓜 | 10 | 1 | 25 | 2.5 | 种子价3.125/粒（25/8=3.125） | -0.625 | 负收益？不合理，需调整种子价或南瓜售价。建议南瓜种子价10金/袋（1.25/粒），则每日净收益(2.5-0.125)=2.375。 |
| 雪莲 | 6 | 1 | 20 | 3.33 | 种子价2/粒（20/10=2） | 1.33 | 药用价值高 |
| 冰晶莓 | 4+5×2=14 | 3 | 10×3=30 | 2.14 | 种子价0.83/粒（10/12≈0.83） | 1.31 | - |

**调整建议**：南瓜种子价应设为10金/袋（8粒），使净收益为正。辣椒种子价可能略低，但考虑到其高收益，可接受。

总体每日净收益在1-2金币/格之间，符合初期经济节奏（一天10格农田赚10-20金）。后期通过品质提升和加工（烹饪、腌制）可进一步提高收益。

---

## 六、与制作系统的联动

- **烹饪**：所有作物均可作为烹饪原料，解锁新食谱（如大蒜面包、辣椒炖肉等）。
- **腌制/发酵**（需要新工作台，如腌制桶）：可将作物转化为腌菜、果酱，延长保质期、提升售价。
- **榨油/制糖**：甜菜可制糖，用于高级料理；辣椒可制辣酱。

这些将在篇章五详细设计配方。

---

## 篇章三完成，等待继续指令

**已完成内容**：
1. 11种作物的完整 SeedData 字段表（包括需要新制作的冬季作物），覆盖ID、季节、生长参数、种子袋参数、成熟作物属性。
2. 种植策略设计（季节搭配、品质提升机制、支架系统）。
3. 农田升级路线（手动→半自动洒水器→全自动灌溉），对应自动化三阶段。
4. 天气联动（夏季枯萎可逆、秋季不可逆、冬季冰雪）。
5. 作物收益分析表，验证数值平衡，并对南瓜种子价提出调整建议。

所有设计基于现有代码框架（SeedData、CropController等），字段全部使用已有定义。对于需要新美术的作物已标注，符合规则。

请指示是否继续篇章四。