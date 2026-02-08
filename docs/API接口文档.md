# Everything Claude Code — API 接口文档

> 版本：1.0
> 日期：2026-02-07
> 项目版本：v1.4.1

---

## 1. 概述

ECC 不提供传统意义上的 REST/RPC API。本文档定义的"接口"是指各组件之间的**交互协议**和**扩展点**，供开发者创建自定义代理、命令、技能、钩子时参考。

---

## 2. 代理接口规范

### 2.1 代理定义接口

创建新代理需遵循以下接口规范：

**输入：** 用户需求文本 + 代码库上下文

**输出：** 结构化结果（由代理 body 中的指令定义）

**文件模板：**

```yaml
---
name: my-custom-agent          # 必填：kebab-case
description: 代理职责描述        # 必填：清晰简洁的描述
tools: ["Read", "Grep", "Glob"] # 必填：允许的工具列表
model: opus                     # 可选：opus（默认）或 haiku
---

## 角色

你是 [角色描述]。

## 工作流程

1. [步骤 1]
2. [步骤 2]
3. [步骤 3]

## 输出格式

[定义输出的结构和格式]

## 约束

- [约束 1]
- [约束 2]
```

### 2.2 代理委托接口

主代理通过以下方式委托给子代理：

```
触发方式：
  1. 命令模板中显式指定 → "委托给 <agent-name>"
  2. 规则建议 → rules/common/agents.md 中定义委托时机
  3. 主代理自主决策 → 根据任务复杂度判断

委托协议：
  主代理 → 创建子代理实例
         → 传入任务描述 + 相关上下文
         → 子代理独立执行（受限工具范围）
         → 子代理返回结果
         → 主代理整合结果
```

---

## 3. 命令接口规范

### 3.1 命令定义接口

**文件模板：**

```yaml
---
description: 命令功能描述    # 必填
---

## 前置条件

[执行命令前需要满足的条件]

## 执行步骤

### 步骤 1: [步骤名]
[具体指令]

### 步骤 2: [步骤名]
[具体指令，可引用代理或技能]

## 输出

[命令的预期输出格式]
```

### 3.2 命令触发接口

```
用户输入：/<command-name> [可选参数]

解析过程：
  1. Claude Code Runtime 匹配 commands/<command-name>.md
  2. 解析 frontmatter 获取元数据
  3. 加载 body 作为执行指令
  4. 注入用户参数替换占位符
  5. 按步骤执行
```

### 3.3 命令间引用接口

命令可在 body 中引用其他组件：

```markdown
## 步骤 1
使用 planner 代理分析需求。

## 步骤 2
加载 tdd-workflow 技能执行测试驱动开发。

## 步骤 3
执行 /verify 命令进行验证。
```

---

## 4. 技能接口规范

### 4.1 技能定义接口

**目录结构：**

```
skills/my-skill/
├── SKILL.md          # 必须：技能主文件
└── README.md         # 可选：详细说明
```

**SKILL.md 模板：**

```yaml
---
name: my-skill              # 必填
description: 技能描述         # 必填
version: 1.0.0              # 可选
---

## 概述

[技能的目的和使用场景]

## 核心知识

### [知识点 1]
[详细内容]

### [知识点 2]
[详细内容]

## 检查清单

- [ ] [检查项 1]
- [ ] [检查项 2]

## 代码示例

[提供具体的代码示例和最佳实践]
```

### 4.2 技能加载接口

```
加载时机：
  1. 命令执行时引用 → 命令 body 中提及技能名
  2. 代理执行时引用 → 代理 body 中提及技能名
  3. 规则中引用 → 规则文件中推荐使用的技能

加载方式：
  Claude Code Runtime 自动将 SKILL.md 内容
  注入到当前上下文中作为参考知识
```

---

## 5. 钩子接口规范

### 5.1 钩子定义接口

**hooks.json 中添加新钩子：**

```json
{
  "hooks": {
    "<EventType>": [
      {
        "matcher": "<ToolName>",
        "command": "<Shell 命令>",
        "timeout": 10000
      }
    ]
  }
}
```

### 5.2 钩子脚本 I/O 接口

#### PreToolUse 钩子

**stdin 输入（JSON）：**
```json
{
  "tool": "Bash",
  "params": {
    "command": "npm run dev"
  }
}
```

**stdout 输出协议：**
| 输出内容 | 效果 |
|---------|------|
| 空（无输出） | 允许工具执行 |
| `"BLOCK"` | 阻止工具执行 |
| 任意文本 | 附加到上下文作为提示 |

**退出码：**
| 退出码 | 含义 |
|--------|------|
| 0 | 正常完成 |
| 非 0 | 钩子执行出错（不阻止工具） |

#### PostToolUse 钩子

**stdin 输入（JSON）：**
```json
{
  "tool": "Bash",
  "params": {
    "command": "npm run dev"
  },
  "result": {
    "stdout": "...",
    "stderr": "...",
    "exitCode": 0
  }
}
```

**stdout 输出：** 附加信息，会注入到后续上下文。

#### SessionStart 钩子

**stdin 输入：** 无

**stdout 输出：** 初始化上下文信息，注入到会话开始。

#### PreCompact 钩子

**stdin 输入（JSON）：**
```json
{
  "reason": "context_limit",
  "currentTokens": 150000,
  "maxTokens": 200000
}
```

**stdout 输出：** 需要保存的状态信息。

### 5.3 自定义钩子脚本模板

```javascript
#!/usr/bin/env node

/**
 * 自定义钩子脚本模板
 *
 * 使用方式：
 * 1. 保存到 scripts/hooks/my-hook.js
 * 2. 在 hooks.json 中注册
 */

const { stdin } = process;

// 读取 stdin 输入
let input = '';
stdin.setEncoding('utf-8');
stdin.on('data', (chunk) => { input += chunk; });
stdin.on('end', () => {
  try {
    const data = input ? JSON.parse(input) : {};

    // 你的钩子逻辑
    const result = processHook(data);

    if (result.block) {
      console.log('BLOCK');
    } else if (result.message) {
      console.log(result.message);
    }

    process.exit(0);
  } catch (error) {
    console.error(`钩子执行出错: ${error.message}`);
    process.exit(1);
  }
});

function processHook(data) {
  // 实现你的逻辑
  return { block: false, message: '' };
}
```

---

## 6. 规则接口规范

### 6.1 规则定义接口

**文件模板：**

```markdown
# 规则标题

## 概述
[规则的目的和适用范围]

## 规则条目

### 规则 1: [规则名]
- **要求：** [必须/应该/建议]
- **说明：** [详细说明]
- **示例：**
  ```
  [正确示例]
  ```
- **反例：**
  ```
  [错误示例]
  ```

### 规则 2: [规则名]
...
```

### 6.2 规则加载接口

```
加载顺序：
  1. rules/common/*.md         ← 始终最先加载
  2. rules/<language>/*.md     ← 按项目语言加载
  3. 项目级 CLAUDE.md           ← 项目特定覆盖

优先级：（后加载的覆盖先加载的）
  common < language-specific < project-level
```

### 6.3 添加新语言规则

```
步骤 1: 创建语言目录
  rules/rust/

步骤 2: 创建标准文件集
  rules/rust/
  ├── coding-style.md    # 编码风格
  ├── hooks.md           # 钩子配置
  ├── patterns.md        # 设计模式
  ├── security.md        # 安全规则
  └── testing.md         # 测试规则

步骤 3: 在 rules/README.md 中注册

步骤 4: 运行验证
  node scripts/ci/validate-rules.js
```

---

## 7. 插件接口规范

### 7.1 插件发布接口

**plugin.json 规范：**

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "插件描述",
  "agents": [
    { "name": "agent-name", "path": "agents/agent-name.md" }
  ],
  "skills": [
    { "name": "skill-name", "path": "skills/skill-name" }
  ],
  "keywords": ["keyword1", "keyword2"]
}
```

**marketplace.json 规范：**

```json
{
  "name": "my-plugin",
  "displayName": "My Plugin",
  "publisher": "github-username",
  "version": "1.0.0",
  "categories": ["Productivity"],
  "installs": 0,
  "rating": 0
}
```

### 7.2 插件安装接口

```
安装命令：
  /plugin marketplace add <publisher>/<repo>
  /plugin install <plugin-name>@<repo>

安装过程：
  1. 从 marketplace 获取元数据
  2. 下载插件包
  3. 解析 plugin.json
  4. 复制 agents → ~/.claude/agents/
  5. 复制 skills → ~/.claude/skills/
  6. 注意：rules 和 hooks 不通过插件分发
```

### 7.3 插件限制说明

| 组件 | 是否通过插件分发 | 原因 |
|------|----------------|------|
| Agents | ✅ 是 | plugin.json 支持 |
| Skills | ✅ 是 | plugin.json 支持 |
| Commands | ✅ 是 | plugin.json 支持 |
| Rules | ❌ 否 | Claude Code 上游限制 |
| Hooks | ❌ 否 | hooks.json 自动加载 |
| Scripts | ❌ 否 | 需要手动安装 |

---

## 8. 脚本工具接口

### 8.1 utils.js 导出接口

```javascript
module.exports = {
  /**
   * 获取当前项目的 Git 根目录
   * @returns {string} 项目根目录绝对路径
   */
  getProjectRoot(),

  /**
   * 获取 ECC 安装根目录
   * @returns {string} ECC 根目录绝对路径
   */
  getECCRoot(),

  /**
   * 安全读取 JSON 文件
   * @param {string} filePath - 文件绝对路径
   * @returns {object|null} 解析后的对象，失败返回 null
   */
  readJsonFile(filePath),

  /**
   * 安全写入 JSON 文件
   * @param {string} filePath - 文件绝对路径
   * @param {object} data - 要写入的数据
   */
  writeJsonFile(filePath, data),

  /**
   * 确保目录存在，不存在则递归创建
   * @param {string} dirPath - 目录绝对路径
   */
  ensureDir(dirPath),

  /**
   * 标准化文件路径
   * @param {string} filePath - 原始路径
   * @returns {string} 标准化后的路径
   */
  normalizePath(filePath)
};
```

### 8.2 package-manager.js 导出接口

```javascript
module.exports = {
  /**
   * 检测当前项目使用的包管理器
   * @returns {string} 'npm' | 'pnpm' | 'yarn' | 'bun'
   */
  detectPackageManager(),

  /**
   * 获取安装依赖命令
   * @param {string} pm - 包管理器名称
   * @returns {string} 完整的安装命令
   */
  getInstallCommand(pm),

  /**
   * 获取添加包命令
   * @param {string} pm - 包管理器名称
   * @param {string} pkg - 包名
   * @param {object} options - { dev: boolean }
   * @returns {string} 完整的添加命令
   */
  getAddCommand(pm, pkg, options),

  /**
   * 获取运行脚本命令
   * @param {string} pm - 包管理器名称
   * @param {string} script - 脚本名
   * @returns {string} 完整的运行命令
   */
  getRunCommand(pm, script)
};
```

### 8.3 session-manager.js 导出接口

```javascript
module.exports = {
  /**
   * 保存当前会话状态
   * @param {object} sessionData - 会话数据对象
   * @returns {string} 保存的会话 ID
   */
  saveSession(sessionData),

  /**
   * 加载最近的会话
   * @returns {object|null} 会话数据，无会话返回 null
   */
  loadLastSession(),

  /**
   * 列出所有历史会话
   * @returns {Array<{id, alias, timestamp}>} 会话摘要列表
   */
  listSessions(),

  /**
   * 根据 ID 加载特定会话
   * @param {string} sessionId - 会话 ID
   * @returns {object|null} 会话数据
   */
  loadSession(sessionId),

  /**
   * 删除指定会话
   * @param {string} sessionId - 会话 ID
   */
  deleteSession(sessionId)
};
```

### 8.4 session-aliases.js 导出接口

```javascript
module.exports = {
  /**
   * 为会话设置别名
   * @param {string} sessionId - 会话 ID
   * @param {string} alias - 可读别名
   */
  setAlias(sessionId, alias),

  /**
   * 通过别名查找会话
   * @param {string} alias - 别名
   * @returns {string|null} 会话 ID
   */
  resolveAlias(alias),

  /**
   * 列出所有别名
   * @returns {Array<{alias, sessionId}>}
   */
  listAliases()
};
```

---

## 9. 上下文（Context）接口

### 9.1 上下文定义

**文件位置：** `contexts/<context-name>.md`

**可用上下文：**

| 名称 | 文件 | 用途 |
|------|------|------|
| dev | `contexts/dev.md` | 开发模式，允许更多操作 |
| research | `contexts/research.md` | 研究模式，侧重探索 |
| review | `contexts/review.md` | 审查模式，侧重检查 |

### 9.2 上下文激活接口

```
激活方式：
  Claude Code CLI 中通过配置指定当前上下文
  上下文内容会注入到系统提示中

上下文优先级：
  默认上下文 < 指定上下文 < 用户实时指令
```

---

## 10. CI 验证器接口

### 10.1 验证器输入输出协议

**输入：** 无参数，自动扫描对应目录

**输出：**
```
成功: "✓ N 个 <组件> 验证通过"  (exit code: 0)
失败: "✗ <文件>: <错误信息>"      (exit code: 1)
```

### 10.2 创建自定义验证器

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

const PATTERN = 'my-components/*.md';
const errors = [];

// 1. 扫描文件
const files = require('glob').sync(PATTERN);

for (const file of files) {
  const content = fs.readFileSync(file, 'utf-8');

  // 2. 执行验证逻辑
  if (!content.includes('---')) {
    errors.push(`${file}: 缺少 frontmatter`);
  }
}

// 3. 输出结果
if (errors.length > 0) {
  errors.forEach(e => console.error(`  ✗ ${e}`));
  process.exit(1);
} else {
  console.log(`✓ ${files.length} 个组件验证通过`);
}
```
