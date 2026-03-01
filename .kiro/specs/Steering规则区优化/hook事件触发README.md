# Kiro Hook 事件触发机制 — 全面参考文档

> 创建日期：2026-02-15
> 用途：全面介绍 Kiro Hook 的所有事件类型、触发条件、机制细节、限制和开发示例

---

## 一、Hook 是什么

Hook 是 Kiro 的自动化机制，将 IDE 事件映射到智能体（Agent）动作。当指定事件发生时，自动执行预设的操作。

### 基本结构

```json
{
  "name": "Hook名称",
  "version": "1.0.0",
  "description": "描述（可选）",
  "when": {
    "type": "事件类型",
    "patterns": ["文件匹配模式"],
    "toolTypes": ["工具类型"]
  },
  "then": {
    "type": "询问智能体（askAgent）或 执行命令（runCommand）",
    "prompt": "询问智能体时的提示词",
    "command": "执行命令时的命令内容"
  }
}
```

### 两种动作类型

| 动作 | 说明 | 适用事件 |
|------|------|---------|
| 询问智能体（askAgent） | 向智能体注入一条提示词，智能体会按提示执行 | 所有事件类型 |
| 执行命令（runCommand） | 执行一条终端命令 | 消息发送时、执行完成后、工具调用前、工具调用后 |

注意：执行命令（runCommand）不能用于文件事件（文件保存/文件创建/文件删除）和用户手动触发。

---

## 二、事件类型总览

Kiro 支持 8 种事件类型，分为三大类：

| 分类 | 事件类型 | 触发时机 |
|------|---------|---------|
| 文件事件 | 文件创建（fileCreated） | 用户创建新文件时 |
| 文件事件 | 文件保存（fileEdited） | 用户保存文件时 |
| 文件事件 | 文件删除（fileDeleted） | 用户删除文件时 |
| 上下文事件 | 消息发送时（promptSubmit） | 用户发送消息给智能体时 |
| 上下文事件 | 执行完成后（agentStop） | 智能体完成一次执行后 |
| 工具事件 | 工具调用前（preToolUse） | 智能体即将调用某个工具之前 |
| 工具事件 | 工具调用后（postToolUse） | 智能体调用某个工具之后 |
| 手动事件 | 用户手动触发（userTriggered） | 用户手动点击按钮触发 |

---

## 三、各事件类型详解

### 3.1 文件创建（fileCreated）

#### 触发条件
- 用户在 IDE 中创建新文件时触发
- 智能体通过工具（如 fsWrite）创建文件时也会触发
- 必须配置 `patterns` 指定监听哪些文件

#### 配置参数
- `patterns`（必填）：文件匹配模式数组，支持通配符（glob）语法
  - `*.ts` — 匹配根目录下的 .ts 文件
  - `**/*.cs` — 递归匹配所有 .cs 文件
  - `Assets/Scripts/**/*.cs` — 匹配特定目录下的 .cs 文件

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ❌

#### 机制细节
- 每创建一个匹配文件触发一次
- 批量创建多个文件会触发多次
- 如果智能体在执行任务时创建文件，也会触发（注意循环风险）

#### 开发示例

**示例 1：新 C# 脚本自动添加文件头**
```json
{
  "name": "C#文件头自动添加",
  "version": "1.0.0",
  "description": "新建C#脚本时自动检查并添加标准文件头注释",
  "when": {
    "type": "fileCreated",
    "patterns": ["**/*.cs"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查刚创建的C#文件是否有标准文件头注释。如果没有，在文件顶部添加：// Copyright (c) 2026 YYY Project. All rights reserved. 然后添加一个空行。不做其他修改。"
  }
}
```

**示例 2：新建可编程对象（ScriptableObject）时提醒注册**
```json
{
  "name": "SO创建提醒",
  "version": "1.0.0",
  "description": "创建新的SO数据文件时提醒开发者注册到Database",
  "when": {
    "type": "fileCreated",
    "patterns": ["Assets/111_Data/**/*.asset"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检测到新建了一个数据资产文件。提醒用户：如果这是一个 ItemData 或 RecipeData，需要在 Database 中注册。只输出一句提醒，不做任何修改。"
  }
}
```

**示例 3：新建测试文件时检查命名规范**
```json
{
  "name": "测试文件命名检查",
  "version": "1.0.0",
  "when": {
    "type": "fileCreated",
    "patterns": ["**/*Tests.cs", "**/*Test.cs"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查新建的测试文件命名是否符合规范：应以 Tests.cs 结尾（不是 Test.cs）。如果不符合，建议重命名。只输出检查结果，不修改文件。"
  }
}
```

---

### 3.2 文件保存（fileEdited）

#### 触发条件
- 用户在 IDE 中保存文件时触发（Ctrl+S 或自动保存）
- 注意：界面上显示为 "File Saved"，配置中写 `fileEdited`
- 智能体通过工具修改文件不会触发此事件（只有用户手动保存才触发）
- 必须配置 `patterns`

#### 配置参数
- `patterns`（必填）：同文件创建事件

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ❌

#### 机制细节
- 每次保存触发一次，频繁保存会频繁触发
- 只监听用户的手动保存操作
- 适合做"保存后检查"类的自动化

#### 开发示例

**示例 1：保存 C# 文件后自动检查编译错误**
```json
{
  "name": "保存后编译检查",
  "version": "1.0.0",
  "description": "保存C#文件后检查是否有编译错误",
  "when": {
    "type": "fileEdited",
    "patterns": ["**/*.cs"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "用 getDiagnostics 检查刚保存的文件是否有编译错误。如果有错误，简要列出；如果没有，回复「编译正常」。不做任何代码修改。"
  }
}
```

**示例 2：保存规则文件后验证格式**
```json
{
  "name": "规则文件格式验证",
  "version": "1.0.0",
  "when": {
    "type": "fileEdited",
    "patterns": [".kiro/steering/*.md"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查刚保存的 steering 文件格式是否正确：1) 如果有 front-matter，检查 inclusion 字段是否合法（auto/manual/fileMatch）2) 检查是否有明显的格式错误。只输出检查结果。"
  }
}
```

**示例 3：保存数据脚本后检查序列化字段**
```json
{
  "name": "SO字段检查",
  "version": "1.0.0",
  "when": {
    "type": "fileEdited",
    "patterns": ["**/Data/**/*.cs"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查保存的数据脚本中是否有公共字段缺少 [SerializeField] 或 [field: SerializeField] 标记。只报告发现的问题，不自动修改。"
  }
}
```

---

### 3.3 文件删除（fileDeleted）

#### 触发条件
- 用户在 IDE 中删除文件时触发
- 通过文件管理器或命令行删除不一定触发（取决于 IDE 是否感知到）
- 必须配置 `patterns`

#### 配置参数
- `patterns`（必填）：同上

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ❌

#### 机制细节
- 文件已经被删除后才触发，此时文件内容已不可读
- 适合做"删除后清理"或"删除确认"类操作
- 注意：无法在提示词中读取已删除文件的内容

#### 开发示例

**示例 1：删除脚本后检查引用**
```json
{
  "name": "删除引用检查",
  "version": "1.0.0",
  "description": "删除C#脚本后检查是否有其他文件引用了它",
  "when": {
    "type": "fileDeleted",
    "patterns": ["**/*.cs"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "一个C#脚本被删除了。用 grepSearch 搜索项目中是否还有其他文件引用了这个被删除的类名。如果有引用，列出受影响的文件；如果没有，回复「无残留引用」。"
  }
}
```

**示例 2：删除预制体（Prefab）后提醒检查场景**
```json
{
  "name": "预制体删除提醒",
  "version": "1.0.0",
  "when": {
    "type": "fileDeleted",
    "patterns": ["**/*.prefab"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检测到一个预制体被删除。提醒用户检查场景中是否有引用该预制体的丢失引用（Missing Reference）。只输出提醒，不做修改。"
  }
}
```

**示例 3：删除数据文件后更新注册表**
```json
{
  "name": "数据文件删除清理",
  "version": "1.0.0",
  "when": {
    "type": "fileDeleted",
    "patterns": ["Assets/111_Data/**/*.asset"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "一个数据资产被删除了。提醒用户：如果这是一个已注册到 Database 的物品，需要从 Database 中移除对应引用，否则运行时会出现丢失引用（Missing Reference）。"
  }
}
```

---

### 3.4 消息发送时（promptSubmit）

#### 触发条件
- 用户在聊天框中发送消息给智能体时触发
- 每次发送消息都会触发，无例外
- 不需要 `patterns` 参数

#### 配置参数
- 无额外必填参数

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ✅

#### 机制细节
- 这是最常用的 Hook 事件之一
- 提示词会在智能体处理用户消息之前注入
- 可以用来做"前置准备"——在智能体回答之前加载必要的上下文
- 每条消息都触发，所以提示词要尽量精简，避免浪费 Token
- 执行命令（runCommand）的输出会作为上下文提供给智能体

#### 约束力与优先级
- 消息发送时的询问智能体提示词约束力属于第 4 层（较弱）
- 在上下文较长或任务复杂时，智能体可能"忘记"Hook 的指令
- 关键规则应放在 steering 规则（始终加载模式）中，而不是 Hook

#### 开发示例

**示例 1：智能规则路由（咱们项目实际在用的）**
```json
{
  "name": "智能助手",
  "version": "3",
  "description": "用户发消息时按需加载领域规则",
  "when": {
    "type": "promptSubmit"
  },
  "then": {
    "type": "askAgent",
    "prompt": "分析用户消息，如涉及以下领域且规则未加载，用 readFile 静默加载：层级→layers.md | UI→ui.md | SO→so-design.md ... 完成后立即回到用户的实际请求。"
  }
}
```

**示例 2：自动获取版本控制状态**
```json
{
  "name": "Git状态注入",
  "version": "1.0.0",
  "description": "每次对话前自动获取当前版本控制状态",
  "when": {
    "type": "promptSubmit"
  },
  "then": {
    "type": "runCommand",
    "command": "git status --short"
  }
}
```

**示例 3：项目编译状态检查**
```json
{
  "name": "编译状态前置检查",
  "version": "1.0.0",
  "when": {
    "type": "promptSubmit"
  },
  "then": {
    "type": "runCommand",
    "command": "dotnet build --no-restore --verbosity quiet 2>&1 | Select-String -Pattern 'error|warning' | Select-Object -First 5"
  }
}
```

---

### 3.5 执行完成后（agentStop）

#### 触发条件
- 智能体完成一次完整的执行后触发
- 包括：回答完用户问题、执行完任务、处理完 Hook 后
- 注意：执行完成后 Hook 本身执行完后也会再次触发（潜在循环风险）

#### 配置参数
- 无额外必填参数

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ✅

#### 机制细节
- 这是"收尾检查"的理想时机
- 循环风险：智能体停止 → Hook 执行 → 智能体再停止 → Hook 再执行...
  - Kiro 内部有防护机制，通常不会无限循环
  - 但提示词设计不当可能导致智能体做多余的事情
- 提示词必须设计得非常精简，让智能体快速判断、快速结束
- 适合做：memory 更新检查、任务完成确认、日志记录

#### 循环防护建议
- 提示词中明确列出"不需要做任何事"的条件
- 限制回复长度（如"只输出一句话"）
- 禁止在提示词中要求智能体执行新任务

#### 开发示例

**示例 1：Memory 更新检查（咱们项目实际在用的）**
```json
{
  "name": "Memory更新检查",
  "version": "3",
  "when": {
    "type": "agentStop"
  },
  "then": {
    "type": "askAgent",
    "prompt": "你是记录员。唯一职责：判断是否需要更新 memory.md。无实质性工作→回复「不需要更新」；已更新→回复「已更新」；需要更新→更新后回复「已完成 memory 更新」。禁止做其他任何事。只输出一句话。"
  }
}
```

**示例 2：执行完成后自动记录日志**
```json
{
  "name": "执行日志",
  "version": "1.0.0",
  "description": "智能体完成执行后自动记录时间戳",
  "when": {
    "type": "agentStop"
  },
  "then": {
    "type": "runCommand",
    "command": "echo 'Agent execution completed at %TIME%' >> .kiro/agent-log.txt"
  }
}
```

**示例 3：完成后生成变更摘要**
```json
{
  "name": "变更摘要",
  "version": "1.0.0",
  "when": {
    "type": "agentStop"
  },
  "then": {
    "type": "askAgent",
    "prompt": "如果这次对话中修改了代码文件，用一句话总结修改内容。如果没有修改任何文件，回复「无文件变更」。禁止输出超过两句话。"
  }
}
```

---

### 3.6 工具调用前（preToolUse）

#### 触发条件
- 智能体即将调用某个工具之前触发
- 在工具实际执行之前拦截，可以阻止工具执行
- 需要配置 `toolTypes` 指定监听哪些工具

#### 配置参数
- `toolTypes`（必填）：工具类型数组，支持内置类别和正则匹配

内置工具类别：

| 类别 | 包含的工具 | 说明 |
|------|-----------|------|
| 读取类（`read`） | readFile、readCode、readMultipleFiles、grepSearch、fileSearch、listDirectory、getDiagnostics | 文件读取相关操作 |
| 写入类（`write`） | fsWrite、fsAppend、strReplace、editCode、deleteFile、semanticRename、smartRelocate | 文件写入相关操作 |
| 终端类（`shell`） | executePwsh、controlPwshProcess | 终端命令执行 |
| 网络类（`web`） | remote_web_search、webFetch | 网络访问 |
| 规格类（`spec`） | taskStatus、updatePBTStatus、prework | 规格文档相关操作 |
| 全部（`*`） | 所有工具 | 通配符，匹配一切 |

正则匹配（用于 MCP 工具）：
- `.*sql.*` — 匹配名称中包含 "sql" 的工具
- `mcp_.*` — 匹配所有 MCP 工具
- `mcp_mcp_unity_.*` — 匹配所有 Unity MCP 工具

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ✅

#### 机制细节（重要）
- 这是唯一能"阻止"工具执行的事件
- Hook 的输出会被智能体读取：
  - 如果输出表示"拒绝/不允许"→ 智能体不会执行该工具
  - 如果输出没有拒绝 → 智能体会用相同参数重新调用该工具
- 适合做：权限控制、安全检查、操作确认
- 循环风险：工具调用前 Hook 中如果要求智能体调用工具，而该工具又触发工具调用前事件 → 无限循环
  - Kiro 有内置的循环检测，嵌套调用中会跳过 Hook
  - 但设计时仍应避免在提示词中要求调用被监听的工具

#### 开发示例

**示例 1：写操作安全确认**
```json
{
  "name": "写操作审查",
  "version": "1.0.0",
  "description": "在写入文件前检查是否符合编码规范",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["write"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查即将写入的内容是否符合项目编码规范。如果发现明显问题（如使用了弃用API、缺少必要注释），输出问题描述。如果没有问题，输出「检查通过」。"
  }
}
```

**示例 2：终端命令安全检查**
```json
{
  "name": "终端命令安全检查",
  "version": "1.0.0",
  "description": "执行终端命令前检查是否安全",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["shell"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查即将执行的终端命令是否安全。如果命令包含危险操作（如 rm -rf、格式化、删除数据库），输出「⛔ 危险操作，已阻止」。否则输出「安全」。"
  }
}
```

**示例 3：Unity MCP 工具调用日志**
```json
{
  "name": "Unity操作日志",
  "version": "1.0.0",
  "description": "记录所有 Unity MCP 工具调用",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["mcp_mcp_unity_.*"]
  },
  "then": {
    "type": "runCommand",
    "command": "echo [%DATE% %TIME%] Unity tool called >> .kiro/unity-operations.log"
  }
}
```

**示例 4：禁止删除特定目录的文件**
```json
{
  "name": "保护核心文件",
  "version": "1.0.0",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["write"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查即将执行的写操作。如果目标是删除 Assets/YYY_Scripts/Data/Core/ 下的任何文件，输出「⛔ 拒绝：核心数据文件受保护，不允许删除」。其他操作输出「允许」。"
  }
}
```

---

### 3.7 工具调用后（postToolUse）

#### 触发条件
- 智能体调用某个工具完成之后触发
- 工具已经执行完毕，结果已经返回
- 需要配置 `toolTypes`（同工具调用前事件）

#### 配置参数
- `toolTypes`（必填）：同工具调用前事件

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ✅

#### 机制细节
- 工具已经执行完，无法撤销操作
- 适合做：结果验证、日志记录、后续处理
- 与工具调用前事件的区别：调用前可以阻止，调用后只能审查和补救
- 可以访问工具的执行结果

#### 开发示例

**示例 1：写入后自动格式化检查**
```json
{
  "name": "写入后格式化",
  "version": "1.0.0",
  "description": "文件写入后自动检查格式",
  "when": {
    "type": "postToolUse",
    "toolTypes": ["write"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查刚写入的文件是否有格式问题（缩进不一致、多余空行等）。如果有，简要指出；如果没有，不输出任何内容。"
  }
}
```

**示例 2：终端命令执行后检查错误**
```json
{
  "name": "命令错误检查",
  "version": "1.0.0",
  "when": {
    "type": "postToolUse",
    "toolTypes": ["shell"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查刚执行的终端命令输出中是否有错误信息。如果有错误，简要说明错误内容和可能的原因。如果执行成功，不输出任何内容。"
  }
}
```

**示例 3：网络搜索结果质量检查**
```json
{
  "name": "搜索结果验证",
  "version": "1.0.0",
  "when": {
    "type": "postToolUse",
    "toolTypes": ["web"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查搜索结果的来源是否可靠（优先官方文档、Stack Overflow、GitHub）。如果结果来自不可靠来源，提醒需要交叉验证。简短输出即可。"
  }
}
```

---

### 3.8 用户手动触发（userTriggered）

#### 触发条件
- 用户在 Kiro 的 Hook 面板中手动点击触发按钮
- 不会自动触发，完全由用户控制
- 不需要 `patterns` 或 `toolTypes`

#### 配置参数
- 无额外必填参数

#### 可用动作
- 询问智能体（askAgent） ✅
- 执行命令（runCommand） ❌

#### 机制细节
- 最安全的事件类型——完全由用户主动触发
- 适合做：按需执行的工具、一键操作、手动检查
- 在 Kiro 的资源管理器侧边栏 "Agent Hooks" 区域可以看到所有手动触发的 Hook，每个都有一个可点击的按钮
- 不存在循环风险或意外触发的问题

#### 开发示例

**示例 1：一键生成项目状态报告**
```json
{
  "name": "项目状态报告",
  "version": "1.0.0",
  "description": "手动触发，生成当前项目的状态报告",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "生成项目状态报告：1) 用 git status 检查未提交的修改 2) 检查 .kiro/specs 下最近活跃的工作区 3) 列出最近修改的5个C#文件。以简洁的格式输出报告。"
  }
}
```

**示例 2：一键检查待办注释**
```json
{
  "name": "待办扫描",
  "version": "1.0.0",
  "description": "扫描项目中所有 TODO 和 FIXME 注释",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "用 grepSearch 搜索项目中所有 TODO 和 FIXME 注释（在 **/*.cs 文件中）。按文件分组列出，每条注释附上行号。如果超过20条，只列出最重要的20条。"
  }
}
```

**示例 3：一键验证存档系统完整性**
```json
{
  "name": "存档系统检查",
  "version": "1.0.0",
  "description": "手动检查存档系统的持久化ID是否有冲突",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "检查项目中所有持久化对象（PersistentObject）的 GUID 是否有重复。搜索所有 .prefab 文件中的 persistentId 字段，检查是否有重复值。输出检查结果。"
  }
}
```

---

## 四、机制深度解析

### 4.1 约束力层级

Hook 的提示词约束力在 Kiro 的规则体系中处于第 4 层（最弱）：

| 层级 | 机制 | 约束力 | 持久性 |
|------|------|--------|--------|
| 1（最强） | steering 规则（始终加载模式） | 每轮系统注入 | 永久 |
| 2 | steering 规则（手动加载模式） | 用户 # 引用时注入 | 当次对话 |
| 3 | 通过 readFile 加载的内容 | 留在上下文中 | 直到上下文溢出 |
| 4（最弱） | Hook 询问智能体的提示词 | 注入一次 | 容易被"淹没" |

实际影响：
- 上下文短（<30%）时，Hook 提示词约束力足够
- 上下文长（>70%）时，Hook 提示词可能被忽略
- 关键规则不要只放在 Hook 中，应同时在 steering 规则中有兜底

### 4.2 循环风险与防护

#### 已知的循环场景

| 场景 | 触发链 | 风险等级 |
|------|--------|---------|
| 执行完成后 + 询问智能体 | 智能体停止 → Hook 执行 → 智能体再停止 → Hook 再执行 | 中（有内置防护） |
| 工具调用前 + 询问智能体要求调用同类工具 | 工具调用前 → Hook 要求调用工具 → 又触发工具调用前 | 高（需手动避免） |
| 文件创建 + 询问智能体创建文件 | 创建文件 → Hook 执行 → 创建新文件 → 又触发 | 高（需手动避免） |
| 消息发送时 + 询问智能体 | 用户发消息 → Hook 执行 → 不会再触发 | 低（单次注入） |

#### Kiro 的内置防护
- 执行完成后的 Hook 执行后，不会无限触发下一轮
- 工具调用前在嵌套调用中会检测循环并跳过
- 但这些防护不是 100% 可靠，提示词设计仍需谨慎

#### 防护最佳实践
1. 执行完成后的提示词必须让智能体快速结束（一句话回复）
2. 工具调用前的提示词不要要求调用被监听类型的工具
3. 文件创建/文件保存的提示词不要要求创建/修改文件
4. 所有 Hook 提示词都应有明确的"不做任何事"的退出条件

### 4.3 性能与 Token 消耗

| 事件类型 | 触发频率 | Token 影响 |
|---------|---------|-----------|
| 消息发送时（promptSubmit） | 每条消息 | 中（提示词长度 × 消息数） |
| 执行完成后（agentStop） | 每次执行 | 中（但可能触发额外的工具调用） |
| 文件保存（fileEdited） | 每次保存 | 低-中（取决于保存频率） |
| 文件创建（fileCreated） | 偶尔 | 低 |
| 文件删除（fileDeleted） | 很少 | 低 |
| 工具调用前（preToolUse） | 每次工具调用 | 高（工具调用频繁时） |
| 工具调用后（postToolUse） | 每次工具调用 | 高（同上） |
| 用户手动触发（userTriggered） | 手动 | 无自动消耗 |

建议：
- 消息发送时和执行完成后的提示词尽量精简
- 工具调用前/后慎用 `*` 通配符，会在每次工具调用时触发
- 高频事件的 Hook 中避免要求智能体调用额外工具（会成倍增加 Token）

### 4.4 多 Hook 共存

- 同一事件类型可以有多个 Hook
- 多个 Hook 会按顺序执行（文件名排序）
- 每个 Hook 的提示词都会注入到智能体上下文
- 多个 Hook 的提示词可能互相冲突——需要注意一致性

---

## 五、事件兼容性速查表

| 事件类型 | 询问智能体 | 执行命令 | 需要文件匹配模式 | 需要工具类型 |
|---------|----------|---------|----------------|-------------|
| 文件创建（fileCreated） | ✅ | ❌ | ✅ | ❌ |
| 文件保存（fileEdited） | ✅ | ❌ | ✅ | ❌ |
| 文件删除（fileDeleted） | ✅ | ❌ | ✅ | ❌ |
| 消息发送时（promptSubmit） | ✅ | ✅ | ❌ | ❌ |
| 执行完成后（agentStop） | ✅ | ✅ | ❌ | ❌ |
| 工具调用前（preToolUse） | ✅ | ✅ | ❌ | ✅ |
| 工具调用后（postToolUse） | ✅ | ✅ | ❌ | ✅ |
| 用户手动触发（userTriggered） | ✅ | ❌ | ❌ | ❌ |

---

## 六、实战经验总结（来自咱们项目）

### 6.1 成功的 Hook 设计

**智能助手（消息发送时触发）**：
- 按需加载领域规则，节省了大量 Token
- 关键：提示词中明确了"静默加载"和"完成后回到用户请求"
- 经过多轮迭代（v1→v3）才稳定

**Memory 更新检查（执行完成后触发）**：
- 自动检查是否需要更新 memory.md
- 关键：严格限制回复为三种固定格式之一
- 禁止做任何额外操作

### 6.2 踩过的坑

1. **执行完成后 Hook 输出过多**：早期版本没有严格限制回复长度，智能体会在 Hook 触发后继续输出分析内容。解决：明确"只输出一句话"+"禁止输出分析/总结/建议"
2. **继承场景下 Hook 失控**：对话摘要（Conversation Summary）后 Hook 触发，智能体被残留上下文"拉回"继续工作。解决：在提示词中增加继承场景的前置判断
3. **Hook 提示词过长**：早期智能助手的提示词很长，每条消息都注入，Token 消耗大。解决：精简提示词，把详细规则放在 steering 文件中
4. **多 Hook 冲突**：两个 Hook 的指令互相矛盾时，智能体行为不可预测。解决：确保 Hook 职责不重叠

### 6.3 设计原则

1. **一个 Hook 只做一件事**——职责单一
2. **提示词越短越好**——长提示词约束力反而弱
3. **明确退出条件**——告诉智能体什么时候"不做任何事"
4. **关键规则不依赖 Hook**——用 steering 始终加载模式兜底
5. **先观察再优化**——不要过度设计，遇到问题再迭代

---

## 七、Hook 文件管理

### 文件位置
- 所有 Hook 文件存放在 `.kiro/hooks/` 目录
- 文件扩展名：`.kiro.hook`
- 文件名建议用短横线命名法（kebab-case）：`my-hook-name.kiro.hook`

### 启用/禁用
- JSON 中的 `"enabled": true/false` 控制
- 禁用后 Hook 不会触发，但配置保留

### 创建方式
1. 在资源管理器侧边栏的 "Agent Hooks" 区域点击 "+" 按钮
2. 使用命令面板搜索 "Open Kiro Hook UI"
3. 直接在 `.kiro/hooks/` 目录创建 JSON 文件
4. 让智能体用 createHook 工具创建

### 修改后生效
- 修改 Hook 文件后自动重新加载，不需要重启 Kiro
