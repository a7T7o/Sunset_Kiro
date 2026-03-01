# 自动化 GUID 管理系统 - 设计文档

## 架构概述

本系统采用 **Editor 自动化守门员** 模式，利用 Unity 的编辑器回调机制，在场景保存时自动检测和修复 GUID 问题。

```
┌─────────────────────────────────────────────────────────────┐
│                    Unity Editor                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │  Scene Saving   │───▶│  PersistentIdAutomator          │ │
│  │  (Ctrl+S)       │    │  - OnSceneSaving()              │ │
│  └─────────────────┘    │  - ScanAndFixGuids()            │ │
│                         │  - DetectDuplicates()           │ │
│  ┌─────────────────┐    └─────────────────────────────────┘ │
│  │  Build Process  │───▶│  PersistentIdBuildValidator     │ │
│  │  (Build)        │    │  - OnPreprocessBuild()          │ │
│  └─────────────────┘    │  - ValidateAllScenes()          │ │
│                         └─────────────────────────────────┘ │
│                                                              │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │  Menu Command   │───▶│  PersistentIdValidator          │ │
│  │  (手动验证)     │    │  - ValidateCurrentScene()       │ │
│  └─────────────────┘    │  - ShowReport()                 │ │
│                         └─────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 组件设计

### 1. PersistentIdAutomator（核心自动化组件）

**职责**：监听场景保存事件，自动修复 GUID 问题

**位置**：`Assets/Editor/PersistentIdAutomator.cs`

**关键实现**：

```csharp
[InitializeOnLoad]
public static class PersistentIdAutomator
{
    // 配置
    private static bool _enabled = true;
    private static LogLevel _logLevel = LogLevel.Normal;
    
    static PersistentIdAutomator()
    {
        EditorSceneManager.sceneSaving += OnSceneSaving;
    }
    
    private static void OnSceneSaving(Scene scene, string path)
    {
        if (!_enabled || Application.isPlaying) return;
        
        var result = ScanAndFixGuids(scene);
        LogResult(result);
    }
    
    private static ScanResult ScanAndFixGuids(Scene scene)
    {
        // 1. 收集所有 IPersistentObject
        // 2. 检测空 GUID 并修复
        // 3. 检测重复 GUID 并修复
        // 4. 返回统计结果
    }
}
```

**访问私有字段的策略**：

由于 `persistentId` 是私有字段，需要通过 `SerializedObject` 访问：

```csharp
private static bool TryFixGuid(MonoBehaviour obj, out string fieldName)
{
    // 尝试常见的字段名
    string[] possibleNames = { "persistentId", "_persistentId" };
    
    SerializedObject so = new SerializedObject(obj);
    foreach (var name in possibleNames)
    {
        SerializedProperty prop = so.FindProperty(name);
        if (prop != null && prop.propertyType == SerializedPropertyType.String)
        {
            if (string.IsNullOrEmpty(prop.stringValue))
            {
                prop.stringValue = System.Guid.NewGuid().ToString();
                so.ApplyModifiedPropertiesWithoutUndo();
                fieldName = name;
                return true;
            }
            fieldName = name;
            return false; // 已有 GUID，无需修复
        }
    }
    fieldName = null;
    return false;
}
```

### 2. PersistentIdValidator（手动验证工具）

**职责**：提供手动验证和修复功能

**位置**：`Assets/Editor/PersistentIdValidator.cs`

**菜单项**：
- `工具/持久化系统/验证当前场景 GUID`
- `工具/持久化系统/验证所有场景 GUID`
- `工具/持久化系统/修复当前场景 GUID`

### 3. PersistentIdBuildValidator（构建验证）

**职责**：在构建前验证所有场景的 GUID 完整性

**位置**：`Assets/Editor/PersistentIdBuildValidator.cs`

**实现**：使用 `IPreprocessBuildWithReport` 接口

```csharp
public class PersistentIdBuildValidator : IPreprocessBuildWithReport
{
    public int callbackOrder => 0;
    
    public void OnPreprocessBuild(BuildReport report)
    {
        var issues = ValidateAllScenes();
        if (issues.Count > 0)
        {
            // 显示对话框，让用户选择修复或取消构建
        }
    }
}
```

## 数据结构

### ScanResult（扫描结果）

```csharp
public class ScanResult
{
    public int TotalScanned;      // 扫描的对象总数
    public int EmptyFixed;        // 修复的空 GUID 数量
    public int DuplicatesFixed;   // 修复的重复 GUID 数量
    public List<string> Warnings; // 警告信息
    public List<string> Errors;   // 错误信息
}
```

### ValidationReport（验证报告）

```csharp
public class ValidationReport
{
    public string SceneName;
    public int TotalObjects;
    public int EmptyGuids;
    public int DuplicateGuids;
    public List<ObjectInfo> Issues;
}

public class ObjectInfo
{
    public string ObjectPath;     // 层级路径
    public string ComponentType;  // 组件类型
    public string CurrentGuid;    // 当前 GUID
    public IssueType Issue;       // 问题类型
}
```

## 配置系统

### EditorPrefs 配置项

| 键 | 类型 | 默认值 | 说明 |
|---|------|--------|------|
| `PersistentId.AutoFix.Enabled` | bool | true | 是否启用自动修复 |
| `PersistentId.AutoFix.LogLevel` | int | 1 | 日志级别 (0=静默, 1=简洁, 2=详细) |
| `PersistentId.Build.FailOnIssue` | bool | false | 构建时发现问题是否中断 |

### 配置窗口

提供 `Edit/Preferences/持久化系统` 配置面板：

```csharp
[SettingsProvider]
public static SettingsProvider CreatePersistentIdSettingsProvider()
{
    return new SettingsProvider("Preferences/持久化系统", SettingsScope.User)
    {
        guiHandler = (searchContext) =>
        {
            // 绘制配置 UI
        }
    };
}
```

## 正确性属性

### P1: GUID 唯一性
**属性**：同一场景内，所有 `IPersistentObject` 的 `PersistentId` 必须唯一
**验证**：保存场景后，扫描所有对象，检查是否有重复 GUID

### P2: GUID 非空性
**属性**：所有 `IPersistentObject` 的 `PersistentId` 不能为空
**验证**：保存场景后，扫描所有对象，检查是否有空 GUID

### P3: GUID 稳定性
**属性**：已有 GUID 的对象，其 GUID 不应被自动修改
**验证**：记录修复前后的 GUID，确保只有空 GUID 被修改

### P4: 幂等性
**属性**：多次保存场景，结果应该一致（不会产生新的修复）
**验证**：连续保存两次，第二次应该报告 0 个修复

## 错误处理

### 场景保存时

1. **对象被销毁**：跳过，不报错
2. **SerializedObject 访问失败**：记录警告，继续处理其他对象
3. **字段名不匹配**：记录警告，建议检查组件实现

### 构建验证时

1. **场景无法加载**：记录错误，继续验证其他场景
2. **发现问题**：显示对话框，提供"修复并继续"/"取消构建"选项

## 日志规范

### 简洁模式（默认）

```
[PersistentIdAutomator] 场景 'Primary' 已修复 3 个空 GUID
```

### 详细模式

```
[PersistentIdAutomator] 正在扫描场景 'Primary'...
[PersistentIdAutomator] 发现 45 个 IPersistentObject
[PersistentIdAutomator] 修复空 GUID: Tree_M1_00/Tree (TreeController)
[PersistentIdAutomator] 修复空 GUID: Tree_M1_01/Tree (TreeController)
[PersistentIdAutomator] 修复空 GUID: Stone_C1_M1/Stone (StoneController)
[PersistentIdAutomator] 完成：修复 3 个空 GUID，0 个重复 GUID
```

## 兼容性考虑

### 现有代码兼容

1. **不修改运行时代码**：所有逻辑都在 Editor 脚本中
2. **保留手动功能**：现有的 ContextMenu "重新生成持久化 ID" 保留
3. **不改变接口**：`IPersistentObject` 接口不变

### 字段名兼容

不同组件可能使用不同的字段名：
- `TreeController`: `persistentId`
- `StoneController`: `_persistentId`

自动化系统需要支持这两种命名。

## 测试策略

### 单元测试

1. **GUID 生成测试**：验证生成的 GUID 格式正确
2. **重复检测测试**：验证能正确检测重复 GUID
3. **字段访问测试**：验证能正确访问不同命名的字段

### 集成测试

1. **场景保存测试**：创建测试场景，保存后验证 GUID 完整性
2. **构建验证测试**：模拟构建流程，验证验证器正常工作

### 手动测试

1. 新建场景，放置树木和石头
2. 保存场景，检查控制台输出
3. 复制物体（Ctrl+D），再次保存，验证重复检测
4. 使用菜单命令验证场景
