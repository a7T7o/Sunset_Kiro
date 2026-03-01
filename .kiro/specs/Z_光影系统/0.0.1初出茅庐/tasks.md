# 实现计划：全局昼夜更替光影系统

## 概述

按路线 B（基础层）→ 路线 A（增强层）的顺序实现。路线 B 为核心优先级，路线 A 为后续增强。所有新脚本放在 `Assets/YYY_Scripts/Service/Rendering/` 目录下。使用 C# / Unity 6。

## 任务

- [x] 1. 创建 DayNightConfig ScriptableObject 配置资产
  - [x] 1.1 创建 `DayNightConfig.cs` ScriptableObject 类
    - 定义四季 Gradient 字段（springGradient, summerGradient, autumnGradient, winterGradient）
    - 定义天气修正色字段（rainyTint, witheringTint, weatherTintStrength）
    - 定义过渡参数字段（weatherTransitionDuration, timeJumpTransitionDuration, seasonTransitionDuration）
    - 定义路线 A 配置字段（globalLightIntensityCurve, globalLightColorGradient, nightLightActivateHour, nightLightDeactivateHour, pointLightFadeDuration）
    - 定义路线融合字段（overlayStrengthWithURP, overlayStrengthWithoutURP）
    - 实现 `GetSeasonGradient(SeasonManager.Season)` 方法
    - 添加 `[CreateAssetMenu]` 特性
    - _需求：6.1, 6.2, 6.3, 6.4, 6.5_
  - [x] 1.2 创建默认配置资产文件
    - 在 `Assets/YYY_Data/Configs/` 下创建 DayNightConfig 资产实例
    - 设置四季默认 Gradient 关键帧（参考设计文档的季节颜色预设表）
    - 设置天气修正色默认值（雨天偏灰、枯萎天偏黄）
    - 设置过渡参数默认值
    - _需求：4.1, 4.2, 4.3, 4.4, 4.5_

- [x] 2. 实现 DayNightOverlay 全屏颜色叠加组件（路线 B）
  - [x] 2.1 创建 `DayNightOverlay.cs` 组件
    - 创建或引用 SpriteRenderer，使用 Multiply 混合模式材质
    - 实现 `SetColor(Color)` 方法设置叠加颜色
    - 实现 `SetStrength(float)` 方法控制叠加强度（通过 alpha 通道）
    - 在 `LateUpdate` 中跟随摄像机位置
    - 根据摄像机正交大小动态计算 Sprite 尺寸，确保覆盖整个视口
    - 设置 Sorting Layer 为 CloudShadow，Order 为 100
    - _需求：2.1, 2.2, 2.3, 2.9, 7.1, 7.2_
  - [x] 2.2 创建 Multiply 混合模式材质
    - 创建 `DayNightMultiply.mat` 材质文件
    - 配置为 Sprites/Default shader + Multiply 混合模式
    - _需求：2.1_

- [x] 3. 实现 DayNightManager 核心协调器
  - [x] 3.1 创建 `DayNightManager.cs` 单例组件
    - 实现单例模式（参考 TimeManager/WeatherSystem 的风格）
    - 在 `Start` 中订阅 TimeManager.OnMinuteChanged、TimeManager.OnHourChanged
    - 在 `Start` 中订阅 SeasonManager.OnSeasonChanged（通过 TimeManager.OnSeasonChanged）
    - 在 `Start` 中订阅 WeatherSystem.OnWeatherChanged
    - 在 `OnDestroy` 中取消所有订阅
    - 缓存 DayNightConfig、DayNightOverlay 引用
    - _需求：1.1, 1.2, 1.3_
  - [x] 3.2 实现颜色计算核心逻辑
    - 实现 `CalculateOverlayColor(float dayProgress)` 方法
    - 根据当前季节从 DayNightConfig 获取对应 Gradient
    - 使用 `Gradient.Evaluate(dayProgress)` 计算基础色调
    - 实现 `ApplyWeatherModifier(ref Color)` 方法叠加天气修正
    - 计算最终颜色并调用 `overlay.SetColor(finalColor)`
    - _需求：2.4, 2.5, 2.6, 2.7, 2.8, 5.1, 5.2, 5.3_
  - [x] 3.3 实现平滑过渡系统
    - 实现天气变化平滑过渡（使用 `Mathf.Lerp` + 计时器，时长来自 config.weatherTransitionDuration）
    - 实现时间跳跃检测（监听 TimeManager.OnSleep 或检测 dayProgress 大幅跳变）
    - 实现时间跳跃后的平滑过渡（时长来自 config.timeJumpTransitionDuration）
    - 实现季节变化平滑过渡（时长来自 config.seasonTransitionDuration）
    - 在 `Update` 中驱动所有过渡计时器
    - _需求：3.1, 3.2, 3.3, 4.6, 5.4_
  - [x] 3.4 实现公共查询接口和调试面板
    - 实现 `GetCurrentOverlayColor()` 返回当前最终颜色
    - 实现 `GetCurrentBrightness()` 返回当前亮度（颜色灰度值）
    - 实现 `IsNightMode()` 返回是否为夜晚模式
    - 实现 `SetOverlayStrength(float)` 调整叠加强度
    - 添加 `[Header]` 和 `[SerializeField]` 暴露调试字段
    - 添加 `[ContextMenu]` 快捷调试方法（跳到特定时间预览）
    - _需求：1.4, 1.5, 7.3_

- [x] 4. 检查点 - 路线 B 基础功能验证
  - 确保所有脚本编译通过
  - 确保 DayNightManager 能正确订阅 TimeManager 事件
  - 确保颜色随时间变化平滑过渡
  - 确保天气修正正常工作
  - 确保与 CloudShadowManager 不冲突
  - 如有问题请询问用户

- [x] 5. 编写路线 B 单元测试和属性测试
  - [x] 5.1 创建 EditMode 测试文件 `DayNightSystemTests.cs`
    - 在 `Assets/Tests/EditMode/` 下创建测试文件
    - 编写特定时间点颜色验证测试（06:00, 12:00, 18:00, 22:00, 00:00）
    - 编写季节 Gradient 选择测试（四季各返回正确 Gradient）
    - 编写配置缺失回退测试
    - _需求：2.4, 2.5, 2.6, 2.7, 2.8, 4.1_
  - [x] 5.2 编写属性测试：时间→颜色映射一致性
    - **属性 1：时间→颜色映射一致性**
    - 生成 100 组随机（季节, dayProgress）组合
    - 验证计算颜色等于对应 Gradient.Evaluate 结果
    - **验证需求：1.2, 1.3, 4.2, 4.3, 4.4, 4.5**
  - [x] 5.3 编写属性测试：颜色平滑连续性
    - **属性 2：颜色平滑连续性**
    - 生成 100 组随机 dayProgress 值 t
    - 计算 t 和 t+Δ（Δ=一个时间步长）的颜色差异
    - 验证差异小于合理阈值
    - **验证需求：3.1, 3.2**
  - [x] 5.4 编写属性测试：天气修正效果
    - **属性 4：天气修正效果**
    - 生成 100 组随机（季节, dayProgress）组合
    - 比较晴天/雨天/枯萎天的颜色
    - 验证雨天亮度低于晴天，枯萎天偏黄，晴天无修正
    - **验证需求：5.1, 5.2, 5.3**
  - [x] 5.5 编写属性测试：路线开关控制行为
    - **属性 6：路线开关控制行为**
    - 随机切换路线 A 开关 100 次
    - 验证 overlay 强度和组件启用状态与开关一致
    - **验证需求：11.2, 11.3**

- [x] 6. 检查点 - 路线 B 测试通过
  - 确保所有单元测试和属性测试通过
  - 如有问题请询问用户

- [x] 7. 实现路线 A 基础设施（URP 增强层）
  - [x] 7.1 创建 `GlobalLightController.cs` 组件
    - 使用条件编译 `#if USE_URP` 隔离 URP 依赖
    - 实现 `SetLightColor(Color)` 控制 Global Light 2D 颜色
    - 实现 `SetLightIntensity(float)` 控制光照强度
    - 实现 `SetEnabled(bool)` 控制启用/禁用
    - 非 URP 环境下所有方法为空实现
    - _需求：9.1, 9.2, 9.3, 9.4_
  - [x] 7.2 创建 `PointLightManager.cs` 组件
    - 使用条件编译 `#if USE_URP` 隔离 URP 依赖
    - 实现 `ActivateNightLights(float fadeDuration)` 渐变激活夜间光源
    - 实现 `DeactivateNightLights(float fadeDuration)` 渐变关闭夜间光源
    - 实现 `RefreshLightList()` 扫描场景中带 NightLight 标签的物体
    - 使用协程实现淡入淡出动画
    - _需求：10.1, 10.2, 10.3, 10.4_
  - [x] 7.3 创建 `NightLightMarker.cs` 标记组件
    - 定义 maxIntensity、lightColor、radius 配置字段
    - 供 PointLightManager 识别和读取参数
    - _需求：10.3_

- [x] 8. 集成路线 A 到 DayNightManager
  - [x] 8.1 在 DayNightManager 中集成路线 A 控制逻辑
    - 实现 `UpdateGlobalLight()` 方法，根据 dayProgress 更新 Global Light 2D
    - 实现 `UpdatePointLights()` 方法，根据时间控制夜间光源开关
    - 实现路线 A/B 融合逻辑（路线 A 启用时自动降低路线 B 强度）
    - 实现路线 A 开关切换的平滑过渡
    - 实现 URP 未安装时的自动降级（`#if USE_URP` 检测）
    - _需求：9.5, 11.1, 11.2, 11.3, 11.4, 11.5_

- [x] 9. 检查点 - 路线 A 集成验证
  - 确保路线 A 组件在非 URP 环境下编译通过（条件编译）
  - 确保路线 A 开关切换不影响路线 B 正常工作
  - 确保路线 A/B 融合逻辑正确
  - 如有问题请询问用户

- [x] 10. 场景集成与最终接线
  - [x] 10.1 创建场景预制体和层级结构
    - 在 MANAGERS 层级下创建 DayNightManager 游戏对象
    - 创建 DayNightOverlay 子对象，配置 SpriteRenderer 和材质
    - 创建 GlobalLightController 子对象（路线 A，默认禁用）
    - 创建 PointLightManager 子对象（路线 A，默认禁用）
    - 连接所有组件引用
    - 分配 DayNightConfig 资产
    - _需求：1.1, 2.1, 2.2_
  - [x] 10.2 验证与现有系统的兼容性
    - 确认 DayNightOverlay 与 CloudShadowManager 的 Sorting Layer 不冲突
    - 确认事件订阅不影响 TimeManager、SeasonManager、WeatherSystem 的正常运行
    - 确认 Multiply 材质叠加效果符合预期
    - _需求：7.1, 7.2, 7.3_

- [x] 11. 最终检查点 - 全系统验证
  - 确保所有测试通过
  - 确保路线 B 独立运行正常
  - 确保路线 A 开关切换正常
  - 确保与所有现有系统兼容
  - 如有问题请询问用户

## 备注

- 所有任务都是必须完成的
- 路线 B（任务 1-6）是核心优先级，必须先完成
- 路线 A（任务 7-9）是增强层，在路线 B 完成后实施
- 任务 10-11 是最终集成和验证
- 每个属性测试引用设计文档中的对应属性编号
- 检查点任务确保增量验证
