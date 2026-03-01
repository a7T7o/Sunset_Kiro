# 自动化 GUID 管理系统 - 任务列表

## 任务概览（锐评024修剪后）

| 任务 | 优先级 | 预估时间 | 状态 |
|------|--------|----------|------|
| 1. 核心自动化组件 | P0 | 30min | ✅ 完成 |
| 2. 手动验证工具 | P1 | 20min | ✅ 完成 |
| ~~3. 构建验证器~~ | ~~P2~~ | ~~15min~~ | ❌ 已砍（锐评024） |
| ~~4. 配置系统~~ | ~~P2~~ | ~~15min~~ | ❌ 已砍（锐评024） |
| 5. 测试验证 | P0 | 15min | ⏳ 待执行 |

---

## 任务详情

### 1. 核心自动化组件 ✅

- [x] 1.1 创建 `Assets/Editor/PersistentIdAutomator.cs`
  - 实现 `[InitializeOnLoad]` 静态类
  - 订阅 `EditorSceneManager.sceneSaving` 事件
  - 实现 `OnSceneSaving` 回调

- [x] 1.2 实现 GUID 扫描和修复逻辑
  - 遍历场景根对象
  - 递归查找所有 `IPersistentObject` 组件
  - 使用 `SerializedObject` 访问私有字段
  - 支持 `persistentId` 和 `_persistentId` 两种字段名

- [x] 1.3 实现重复 GUID 检测
  - 使用 Dictionary 记录已见 GUID
  - 对重复的 GUID 重新生成（保留第一个）
  - 输出警告日志

- [x] 1.4 实现日志输出
  - 简洁模式：只输出修复数量
  - 无修复时不输出日志

---

### 2. 手动验证工具 ✅

- [x] 2.1 创建 `Assets/Editor/PersistentIdValidator.cs`
  - 实现静态工具类

- [x] 2.2 实现菜单命令
  - `工具/持久化系统/验证当前场景 GUID`
  - `工具/持久化系统/修复当前场景 GUID`
  - `工具/持久化系统/验证所有场景 GUID`

- [x] 2.3 实现验证报告
  - 统计总对象数、空 GUID 数、重复 GUID 数
  - 列出问题对象的层级路径
  - 使用 EditorUtility.DisplayDialog 显示结果

---

### ~~3. 构建验证器~~ ❌ 已砍

> **锐评024裁决**：降级为 Log Warning，但由于 SceneGuidAutomator 已在保存时工作，构建验证器不再必要。

---

### ~~4. 配置系统~~ ❌ 已砍

> **锐评024裁决**：这是修复严重 Bug 的核心机制，必须 Always On。不需要配置 UI。

---

### 5. 测试验证 ⏳

- [ ] 5.1 功能测试
  - 新建测试场景
  - 放置树木和石头（不手动生成 GUID）
  - 保存场景，验证自动修复
  - 检查控制台输出

- [ ] 5.2 重复检测测试
  - 复制已有 GUID 的对象（Ctrl+D）
  - 保存场景，验证重复检测和修复
  - 检查警告日志

- [ ] 5.3 幂等性测试
  - 连续保存两次
  - 第二次应报告 0 个修复

- [ ] 5.4 手动验证测试
  - 使用菜单命令验证场景
  - 检查报告内容

---

## 完成标准

1. ✅ 保存场景时自动修复空 GUID
2. ✅ 保存场景时检测并修复重复 GUID
3. ✅ 提供手动验证菜单命令
4. ❌ ~~提供配置面板控制行为~~（已砍）
5. ⏳ 所有测试通过
6. ⏳ 更新 memory.md 记录

---

## 已创建文件

| 文件 | 说明 |
|------|------|
| `Assets/Editor/PersistentIdAutomator.cs` | 核心自动化组件 |
| `Assets/Editor/PersistentIdValidator.cs` | 手动验证工具 |
