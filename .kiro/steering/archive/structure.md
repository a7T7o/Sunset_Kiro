---
inclusion: manual
priority: P3
keywords: [项目结构, 文件夹, 命名规范, 场景结构]
lastUpdated: 2026-02-11
archivedFrom: .kiro/steering/structure.md
archivedDate: 2026-01-09
---

## 中文概述（面向开发者本人）

- **最后更新**：2026-02-11
- **根目录结构**：
  - `Assets/` - Unity 资源（脚本、精灵、预制体、场景）
  - `Docx/` - 设计文档和技术方案
  - `Packages/` - 包清单
  - `ProjectSettings/` - Unity 项目设置
- **脚本组织**（`Assets/YYY_Scripts/`）：
  - `Anim/` - 动画控制（Player、NPC）
  - `Combat/` - 战斗系统（命中检测、资源节点注册）
  - `Controller/` - 控制器（Player、NPC、Input、TreeController、StoneController）
  - `Core/` - 核心框架（Events、Pool、Services）
  - `Data/` - 数据结构（Items、Database、Enums、Recipes、Core）
  - `Effects/` - 特效（落叶等）
  - `Events/` - 事件定义（PlacementEvents、ToolEvents）
  - `Farm/` - 农田系统（FarmTileManager、CropController 等）
  - `Interfaces/` - 接口定义（IInteractable、IResourceNode）
  - `Service/` - 核心服务（Camera、Crafting、Equipment、Inventory、Navigation、Placement、Player、Rendering）
  - `UI/` - UI 组件（Box、Crafting、Debug、Inventory、Tabs、Toolbar、Utility）
  - `Utils/` - 工具类和扩展
  - `World/` - 世界物体（Placeable、拾取物、生成器）
- **其他资源目录**：
  - `Assets/000_Scenes/` - 场景文件
  - `Assets/100_Anim/` + `Assets/100_Animations/` - 动画资源
  - `Assets/111_Data/` - SO 数据资源（Database、Items、Recipes、Drop Table）
  - `Assets/222_Prefabs/` - 预制体（Box、Farm、Tree、Rock、UI 等）
  - `Assets/444_Shaders/` - 着色器和材质
  - `Assets/Editor/` - 编辑器工具脚本
  - `Assets/Sprites/` - 精灵图资源
  - `Assets/Scripts/Utils/` - 旧工具类（部分保留）
- **命名规范**：
  - 类名：PascalCase（如 `TimeManager`）
  - 私有字段：_camelCase（如 `_currentTime`）
  - 常量：UPPER_CASE（如 `MAX_HEALTH`）
  - GameObject：PascalCase + 下划线（如 `Tree_M1_00`）

> **注意**：此为目录结构概述，详细规范请参考对应的 steering 规则（如 `layers.md`、`coding-standards.md`）。

---

# Project Structure

## Root Folders

```
Sunset/
├── Assets/              # Unity assets (scripts, sprites, prefabs, scenes)
├── Docx/               # Design documents and technical solutions
├── Plan/               # Planning documents
├── Library/            # Unity cache (ignored)
├── Logs/               # Unity logs (ignored)
├── Packages/           # Package manifest
├── ProjectSettings/    # Unity project settings
├── Temp/               # Temporary files (ignored)
└── UserSettings/       # User-specific settings (ignored)
```

## Assets Organization

### Scripts (`Assets/YYY_Scripts/`)

**Core Framework**:
- `Core/Events/` - Event system (EventBus)
- `Core/Pool/` - Object pooling
- `Core/Services/` - Service locator

**Services**:
- `Service/` - Core managers and services
  - `TimeManager.cs`, `SeasonManager.cs`, `WeatherSystem.cs`
  - `Camera/` - Camera system
  - `Crafting/` - Crafting system
  - `Equipment/` - Equipment system
  - `Inventory/` - Inventory service and bootstrap
  - `Navigation/` - NavGrid2D pathfinding
  - `NPC/` - NPC services
  - `Placement/` - Placement system (Manager, Preview, Validator, GridCalculator, Navigator, LayerDetector)
  - `Player/` - Player services (AutoNavigator)
  - `Rendering/` - Rendering utilities
  - `PersistentManagers.cs` - Manager persistence

**Controllers**:
- `Controller/Player/` - Player movement and input
- `Controller/Input/` - GameInputManager
- `Controller/NPC/` - NPC controllers
- `Controller/TreeController.cs`, `StoneController.cs`, `RockController.cs`

**Animation**:
- `Anim/Player/` - Player animation controllers
- `Anim/NPC/` - NPC animation controllers

**Game Systems**:
- `Data/` - Data structures
  - `Items/` - ItemData, ToolData, WeaponData, SeedData, CropData
  - `Database/` - Item database
  - `Enums/` - Enum definitions
  - `Recipes/` - Crafting recipes
  - `Core/` - SaveDataDTOs, DynamicObjectFactory, PersistentObjectRegistry, InventoryItem
- `Farm/` - Farming system (FarmTileManager, CropController, CropManager, FarmlandBorderManager, FarmToolPreview, SeedBagHelper)
- `Combat/` - Combat system (PlayerToolHitEmitter, ResourceNodeRegistry)
- `Effects/` - Visual effects (LeafFallEffect, LeafSpawner)
- `Events/` - Event definitions (PlacementEvents, ToolEvents)
- `UI/` - UI components (Box, Crafting, Debug, Inventory, Tabs, Toolbar, Utility)
- `World/` - World objects (Placeable/ChestController, WorldItemPickup, WorldItemDrop, WorldItemPool, WorldSpawnService)
- `Interfaces/` - IInteractable, IResourceNode

**Utilities**:
- `Utils/` - Helper classes, Attributes, Editor tools

### Sprites (`Assets/Sprites/`)

```
Sprites/
├── 0_Tool & Weapon 'Anim/  # Tool/weapon animation sprites
├── Equipments/             # Equipment sprites
├── Farm/                   # Farming sprites
├── Generated/              # Auto-generated sprites
├── House/                  # Building sprites
├── Props/                  # World prop sprites
├── Shadow_WorldPrefab/     # Shadow sprites for world prefabs
└── UI/                     # UI icons and elements
```

### Prefabs (`Assets/222_Prefabs/`)

```
222_Prefabs/
├── 000_Temp/          # Temporary prefabs
├── Box/               # Chest/box prefabs
├── Dungeon props/     # Dungeon/combat props
├── Farm/              # Farming-related prefabs
├── House/             # Building prefabs
├── Pan/               # Pan prefabs
├── Rock/              # Rock and mineral prefabs
├── Tree/              # Tree prefabs (with growth stages)
├── UI/                # UI panel prefabs
└── WorldItems/        # World item prefabs
```

### Data (`Assets/111_Data/`)

```
111_Data/
├── Database/       # ScriptableObject databases
├── Drop Table/     # Drop table data
├── Items/          # Item data assets
└── Recipes/        # Recipe data assets
```

### Editor Tools (`Assets/Editor/`)

Custom editor scripts for workflow automation:
- Batch operations (prefabs, sprites, animations)
- Animation setup tools
- Tilemap utilities
- Debug visualizers

## File Structure

```
Assets/Scripts/Utils/       # Legacy utils (partially retained)
Assets/YYY_Scripts/         # Main scripts directory
```

## Naming Conventions

### Scripts
- **PascalCase** for classes: `TimeManager`, `PlayerController`
- **camelCase** for private fields: `currentTime`, `isMoving`
- **PascalCase** for public properties: `CurrentSeason`, `IsActive`
- **UPPER_CASE** for constants: `MAX_HEALTH`, `DEFAULT_SPEED`

### GameObjects
- **PascalCase** with underscores: `Tree_M1_00`, `Player_Character`
- Prefixes for organization: `UI_`, `NPC_`, `Enemy_`

### Assets
- **Descriptive names**: `Player_Idle_Down_0`, `Tree_Spring_Large`
- **Numbered sequences**: `_0`, `_1`, `_2` for animation frames
- **Quality suffix**: `_0_High`, `_0_Low` for quality variants

## Scene Structure

> **详细的场景层级结构请参考 `layers.md`，此处仅为概述。**

### Sorting Layers (Bottom to Top)
1. Background
2. Ground
3. Objects
4. Characters
5. Effects
6. UI

## Key Files

- `Assets/InputSystem_Actions.inputactions` - Input system configuration
- `Packages/manifest.json` - Package dependencies
- `ProjectSettings/ProjectVersion.txt` - Unity version
- `ProjectSettings/TagManager.asset` - Tags and layers
