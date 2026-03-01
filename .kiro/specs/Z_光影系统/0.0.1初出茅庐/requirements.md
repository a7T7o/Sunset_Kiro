# 需求文档：全局昼夜更替光影系统

## 简介

为 2D 像素风农场游戏设计全局昼夜更替光影系统，采用双路线架构：
- **路线 B（基础层）**：全屏颜色叠加方案，模拟星露谷物语级别的色调系统，使用 SpriteRenderer + Multiply 混合模式
- **路线 A（增强层）**：URP 2D 光照系统，作为可选的光影模组级增强

路线 B 为核心基础，必须能独立运行；路线 A 为可选增强层，通过开关控制启用。

## 术语表

- **DayNightManager**：昼夜管理器，负责协调所有光影效果的核心单例组件
- **DayNightOverlay**：全屏颜色叠加层，使用 SpriteRenderer + Multiply 混合模式实现色调变化
- **TimeManager**：已有的时间管理器单例，提供游戏时间事件和查询接口
- **SeasonManager**：已有的季节管理器单例，提供四季和植被季节信息
- **WeatherSystem**：已有的天气系统单例，提供天气类型和天气变化事件
- **CloudShadowManager**：已有的云朵阴影管理器，使用 Multiply 材质的精灵叠加方案
- **DayNightConfig**：ScriptableObject（可编程对象）配置资产，存储颜色曲线和参数
- **颜色曲线**：使用 Unity Gradient 定义的 24 小时色调变化数据
- **路线 B**：基础层，全屏颜色叠加方案，不依赖 URP
- **路线 A**：增强层，URP 2D 光照系统，依赖 URP 渲染管线
- **Global Light 2D**：URP 2D 渲染管线提供的全局光源组件
- **Point Light 2D**：URP 2D 渲染管线提供的点光源组件

## 需求

### 需求 1：昼夜管理器核心

**用户故事：** 作为玩家，我希望游戏世界有昼夜更替效果，让游戏体验更加沉浸。

#### 验收标准

1. THE DayNightManager SHALL 以单例模式运行，遵循项目现有的单例模式风格
2. WHEN TimeManager 发出 OnMinuteChanged 事件时，THE DayNightManager SHALL 根据当前游戏时间更新光影效果
3. WHEN TimeManager 发出 OnHourChanged 事件时，THE DayNightManager SHALL 触发整点光影状态检查
4. THE DayNightManager SHALL 提供公共接口查询当前光影状态（当前色调颜色、亮度级别、是否为夜晚模式）
5. THE DayNightManager SHALL 在 Inspector 中暴露调试控制面板，支持手动预览任意时间点的光影效果

### 需求 2：全屏颜色叠加（路线 B 核心）

**用户故事：** 作为玩家，我希望游戏画面随时间变化呈现不同色调，从清晨的温暖金色到深夜的蓝紫色。

#### 验收标准

1. THE DayNightOverlay SHALL 使用 SpriteRenderer 配合 Multiply 混合模式材质实现全屏颜色叠加
2. THE DayNightOverlay SHALL 放置在 CloudShadow Sorting Layer 之上，确保覆盖所有游戏元素
3. THE DayNightOverlay SHALL 跟随摄像机位置移动，始终覆盖整个可视区域
4. WHEN 游戏时间为 06:00 时，THE DayNightOverlay SHALL 显示温暖的淡金色调
5. WHEN 游戏时间为 12:00 时，THE DayNightOverlay SHALL 显示最亮的近乎无色调效果
6. WHEN 游戏时间为 18:00 时，THE DayNightOverlay SHALL 显示橙红色调
7. WHEN 游戏时间为 22:00 时，THE DayNightOverlay SHALL 显示深蓝色调
8. WHEN 游戏时间为 00:00 时，THE DayNightOverlay SHALL 显示最深的蓝紫色调
9. THE DayNightOverlay SHALL 在不依赖 URP 的情况下独立运行（仅使用内置渲染管线）

### 需求 3：颜色平滑过渡

**用户故事：** 作为玩家，我希望昼夜色调变化是平滑渐变的，不会出现突兀的颜色跳变。

#### 验收标准

1. WHEN 游戏时间在两个关键色调节点之间时，THE DayNightManager SHALL 使用插值计算平滑过渡颜色
2. THE DayNightManager SHALL 在每个 10 分钟时间步进内保持颜色连续变化
3. IF 时间发生跳跃（如睡觉、调试跳时间）时，THEN THE DayNightManager SHALL 在短时间内（0.5-1 秒）平滑过渡到目标颜色，而非瞬间跳变

### 需求 4：季节色调变体

**用户故事：** 作为玩家，我希望不同季节的光影色调有所区别，春天清新、夏天明亮、秋天温暖、冬天寒冷。

#### 验收标准

1. THE DayNightConfig SHALL 为四个季节（春、夏、秋、冬）各存储一条独立的 24 小时颜色曲线
2. WHEN 当前季节为春季时，THE DayNightManager SHALL 使用偏绿偏清新的颜色曲线
3. WHEN 当前季节为夏季时，THE DayNightManager SHALL 使用偏暖偏明亮的颜色曲线
4. WHEN 当前季节为秋季时，THE DayNightManager SHALL 使用偏橙偏暖的颜色曲线
5. WHEN 当前季节为冬季时，THE DayNightManager SHALL 使用偏蓝偏冷的颜色曲线
6. WHEN SeasonManager 发出 OnSeasonChanged 事件时，THE DayNightManager SHALL 平滑过渡到新季节的颜色曲线


### 需求 5：天气联动

**用户故事：** 作为玩家，我希望天气变化能影响光影效果，雨天更灰暗、枯萎天更干燥。

#### 验收标准

1. WHEN WeatherSystem 发出 OnWeatherChanged 事件且天气为 Rainy 时，THE DayNightManager SHALL 在当前色调基础上叠加偏灰偏暗的天气修正
2. WHEN WeatherSystem 发出 OnWeatherChanged 事件且天气为 Withering 时，THE DayNightManager SHALL 在当前色调基础上叠加偏黄偏干燥的天气修正
3. WHEN WeatherSystem 发出 OnWeatherChanged 事件且天气为 Sunny 时，THE DayNightManager SHALL 移除天气色调修正，恢复正常昼夜色调
4. WHEN 天气发生变化时，THE DayNightManager SHALL 在 1-2 秒内平滑过渡到新的天气色调

### 需求 6：ScriptableObject 配置系统

**用户故事：** 作为开发者，我希望所有光影参数都通过 ScriptableObject 配置，方便在编辑器中调整和预览。

#### 验收标准

1. THE DayNightConfig SHALL 以 ScriptableObject 形式存储所有光影配置数据
2. THE DayNightConfig SHALL 包含四季各自的 24 小时颜色 Gradient
3. THE DayNightConfig SHALL 包含天气修正颜色参数（雨天修正色、枯萎天修正色）
4. THE DayNightConfig SHALL 包含过渡时间参数（天气过渡时长、时间跳跃过渡时长）
5. THE DayNightConfig SHALL 支持在 Inspector 中实时预览颜色曲线
6. WHEN DayNightConfig 的参数在编辑器中被修改时，THE DayNightManager SHALL 在运行时实时反映变化

### 需求 7：与 CloudShadowManager 协调

**用户故事：** 作为开发者，我希望昼夜光影系统与已有的云朵阴影系统互不冲突，视觉效果互补。

#### 验收标准

1. THE DayNightOverlay SHALL 放置在 CloudShadow Sorting Layer 之上的独立 Sorting Layer 或更高 Order
2. WHILE CloudShadowManager 处于活跃状态时，THE DayNightOverlay SHALL 保持正常工作，两者视觉效果叠加
3. THE DayNightManager SHALL 提供接口调整 Overlay 强度，允许与 CloudShadowManager 的效果协调

### 需求 8：URP 迁移与配置（路线 A 前置）

**用户故事：** 作为开发者，我希望能将项目从内置渲染管线迁移到 URP，为 2D 光照系统做准备。

#### 验收标准

1. WHEN 启用路线 A 时，THE 项目 SHALL 安装并配置 URP（com.unity.render-pipelines.universal）包
2. WHEN URP 安装完成后，THE 项目 SHALL 配置 2D Renderer 作为默认渲染器
3. WHEN URP 迁移完成后，THE 项目 SHALL 将所有现有材质升级为 URP 兼容材质
4. IF URP 迁移导致现有视觉效果异常，THEN THE 开发者 SHALL 逐一修复受影响的材质和着色器

### 需求 9：URP 全局光源（路线 A 核心）

**用户故事：** 作为玩家，我希望游戏有更真实的全局光照效果，光源颜色和强度随昼夜变化。

#### 验收标准

1. WHEN 路线 A 启用时，THE DayNightManager SHALL 创建并控制一个 Global Light 2D 组件
2. WHEN 游戏时间变化时，THE Global Light 2D SHALL 根据 DayNightConfig 中的光照曲线调整颜色和强度
3. WHEN 游戏时间为白天（06:00-18:00）时，THE Global Light 2D SHALL 保持较高强度（0.8-1.0）
4. WHEN 游戏时间为夜晚（20:00-04:00）时，THE Global Light 2D SHALL 降低强度至 0.2-0.4
5. WHEN 路线 A 启用时，THE DayNightOverlay SHALL 自动降低 Multiply 叠加强度，避免与 Global Light 2D 效果过度叠加

### 需求 10：URP 局部光源（路线 A 增强）

**用户故事：** 作为玩家，我希望夜晚能看到建筑窗户透出灯光、路灯照亮道路，增加游戏氛围。

#### 验收标准

1. WHEN 路线 A 启用且游戏时间进入夜晚（18:00 之后）时，THE DayNightManager SHALL 激活场景中标记为夜间光源的 Point Light 2D 组件
2. WHEN 游戏时间进入白天（06:00 之后）时，THE DayNightManager SHALL 关闭夜间 Point Light 2D 组件
3. THE Point Light 2D 组件 SHALL 支持通过标签或组件标记进行批量管理
4. WHEN 夜间光源激活或关闭时，THE DayNightManager SHALL 使用渐变动画（淡入/淡出）而非瞬间开关

### 需求 11：路线 A/B 融合控制

**用户故事：** 作为开发者，我希望路线 A 和路线 B 能协调工作，通过开关控制路线 A 的启用状态。

#### 验收标准

1. THE DayNightManager SHALL 提供布尔开关控制路线 A（URP 增强层）的启用状态
2. WHILE 路线 A 关闭时，THE DayNightManager SHALL 仅使用路线 B（全屏颜色叠加）提供光影效果
3. WHILE 路线 A 开启时，THE DayNightManager SHALL 同时运行路线 A 和路线 B，并自动调整路线 B 的叠加强度
4. WHEN 路线 A 开关状态变化时，THE DayNightManager SHALL 平滑过渡两种模式，避免视觉跳变
5. IF 项目未安装 URP 包，THEN THE DayNightManager SHALL 自动禁用路线 A 相关功能，仅运行路线 B
