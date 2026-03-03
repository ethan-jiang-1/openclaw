# OpenClaw 运行态解析：Agent Workspace 与上下文组装机制

本文档专注于 OpenClaw 安装后的**运行态（Runtime）**结构，重点解析 Agent 在终端用户的机器上是如何通过 `~/.openclaw` 目录感知环境、加载配置并拼装出最终的大模型 Prompt 的。

## 一、 核心运行态目录：`~/.openclaw/`

当 OpenClaw 在用户系统上安装并运行后，会在当前用户的主目录下生成一个大脑枢纽目录：`~/.openclaw/`。这里存放着 Agent 所有的运行时状态、记忆和扩展能力。

| 目录/文件 | 核心作用 |
| :--- | :--- |
| **`workspace/`** | **当前工作区**。存放指导 Agent 行为的文本配置文件（即各项 `.md` 指令）。如果是多配置运行，可能存在多个类似 `workspace-<profile>/` 的目录。 |
| **`sessions/`** | **短期记忆库**。存放所有历史对话流（JSONL 格式）。Agent 的每次会话上下文都在这里持久化。 |
| **`skills/`** | **全局技能库**。存放用户全局安装的自定义技能扩展。 |
| **`credentials/`** | **密钥库**。存放系统访问 Web Provider 等外部 API 所需的认证信息和 Token。 |

## 二、 Workspace 内部配置：定义 Agent 的"灵魂"

进入 `workspace/` 目录，你会看到一系列以大写字母命名的 Markdown 文件。这些文件并不是普通的项目说明文档，而是 **Agent 的"性格参数"与"行为守则"**。

### 2.1 核心配置文件

1. **`AGENTS.md` (系统总则)**
   - **作用**：Agent 的最高指令集。定义当前工作区的全局指导原则、代码规约、提交流程等。
   - **何时加载**：每次会话都会加载，是最基础的上下文来源。

2. **`SOUL.md` (人格基调)**
   - **作用**：定义 Agent 的沟通语气与人设（Persona）。例如规定其必须表现得像一个简练专业的 CLI 开发者，禁止说废话。
   - **影响**：直接决定 Agent 回复的语言风格和语气。

3. **`TOOLS.md` (工具箱守则)**
   - **作用**：规定 Agent 使用本地工具（如 Bash, Edit, Glob）的安全限制和最佳实践。
   - **关键内容**：例如禁止 `sed` 直接修改文件、必须使用专门的 Edit 工具等安全约束。

4. **`IDENTITY.md` (自我认知)**
   - **作用**：告诉 Agent "你是谁"，例如声明它是一个运行在特定机器上的系统级智能体。
   - **用途**：建立 Agent 对自身角色的认知基础。

5. **`USER.md` (用户画像)**
   - **作用**：提供使用者的偏好信息（如时区设置 `userTimezone`、特定使用习惯）。
   - **示例**：下午 5 点后不发打扰消息、优先使用简体中文等个性化配置。

### 2.2 特殊场景文件

6. **`MEMORY.md` (长时记忆块)**
   - **作用**：持久化的长期记忆。
   - **加载时机**：**仅在主会话（Main Session）中被加载**，子会话（如 Subagent）不会加载以节省 Token。
   - **内容示例**：用户的历史偏好、项目特有的注意事项等。

7. **`BOOTSTRAP.md` (首启引导)**
   - **作用**：用于首次唤醒时的破冰与引导。
   - **场景**：新安装或重置时，帮助 Agent 理解项目结构和快速上手。

8. **`HEARTBEAT.md` (心跳模式)**
   - **作用**：定义 Agent 在后台静默运行（心跳模式）时的巡检逻辑。
   - **场景**：如定时检查系统健康、执行周期性任务时的行为规范。

## 三、 上下文组装管线：从静态文件到运行态 Prompt

当用户发起交互时，OpenClaw 通过以下五个阶段（主要在 `src/agents/` 目录下的逻辑）将零散的配置整合为 Agent 可以"干工作"的上下文。

### 3.1 阶段一：文件加载与动态过滤

**核心文件**：`src/agents/workspace.ts`, `src/agents/bootstrap-files.ts`

系统会从 `~/.openclaw/workspace/` 读取所有的基础配置文件：

**默认加载文件列表**：
```typescript
DEFAULT_AGENTS_FILENAME = "AGENTS.md"
DEFAULT_SOUL_FILENAME = "SOUL.md"
DEFAULT_TOOLS_FILENAME = "TOOLS.md"
DEFAULT_IDENTITY_FILENAME = "IDENTITY.md"
DEFAULT_USER_FILENAME = "USER.md"
DEFAULT_HEARTBEAT_FILENAME = "HEARTBEAT.md"
DEFAULT_BOOTSTRAP_FILENAME = "BOOTSTRAP.md"
DEFAULT_MEMORY_FILENAME = "MEMORY.md"
```

**动态过滤机制**：
- **主代理 (Main Agent)**：加载完整的 Workspace 文件库，包括 `MEMORY.md`。
- **子代理 (Subagent)**：仅加载 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md` 五个核心文件，自动剥离掉 `MEMORY.md` 等繁重上下文。
- **过滤依据**：钩子 Hook 可以根据 `run.kind`（default/heartbeat/cron）或 Session 类型返回不同的文件集合。

**读取保护**：
- 通过 `openBoundaryFile()` 进行安全校验，防止路径遍历攻击。
- 文件大小限制：最大 2MB。
- 缓存机制：基于文件 inode/mtime 自动缓存，避免重复读取。

### 3.2 阶段二：钩子干预与内容注入

**核心文件**：`src/agents/bootstrap-hooks.ts`

在构建上下文之前，系统会触发内置的钩子（Hooks）。Hooks 可以动态修改即将加载的 Bootstrap 文件列表。

**关键钩子示例**：

1. **`bootstrap-extra-files`** 钩子
   - **作用**：允许通过配置 `agents.defaults.bootstrap.extraFiles` 强行注入额外的文件。
   - **配置示例**：
     ```yaml
     agents:
       defaults:
         bootstrap:
           extraFiles:
             - "packages/*/AGENTS.md"
             - "docs/reference/SPECIAL_GUIDE.md"
     ```
   - **效果**：无论 Agent 调用什么工具，这些文件都会被强行塞进上下文中。

2. **`session-memory`** 钩子
   - **作用**：在 `/new` 或 `/reset` 命令时触发。
   - **功能**：自动保存当前会话的上下文到 Memory 系统，或读取历史记忆。

3. **`boot-md`** 钩子
   - **作用**：在 Gateway 启动时运行 `BOOT.md`。
   - **场景**：网关初始化时的自动化配置或健康检查。

**Hook 触发时机**：
```typescript
triggerInternalHook("agent", "bootstrap", sessionKey, {
  workspaceDir,
  bootstrapFiles,  // 当前已加载的文件列表
  cfg,
  sessionKey,
  agentId
})
```

### 3.3 阶段三：技能嗅探与 XML 格式化

**核心文件**：`src/agents/skills.ts`, `src/agents/skills/workspace.ts`

Agent 会自动扫描它当前拥有的扩展能力。系统会按**优先级（低到高）**依次扫描：

1. 外部扩展目录（配置 `skills.load.extraDirs`）
2. 内置技能目录（`openclaw/dist/skills`）
3. 管理技能目录（`~/.openclaw/skills`）
4. 个人代理技能（`~/.agents/skills`）
5. 项目代理技能（`workspace/.agents/skills`）
6. Workspace 技能（`workspace/skills`）

**Skills 数据结构**：
每个技能目录包含一个 `SKILL.md` 文件，带 YAML 前置元数据：
```markdown
---
name: 1password
description: When to use this skill
homepage: https://...
metadata:
  openclaw:
    emoji: "🔐"
    requires: { bins: ["op"] }
    install: ["brew install 1password-cli"]
---

## Usage

Use this skill to...
```

**XML 格式化输出**：
所有符合条件的 Skills 会被转换成统一的 XML 结构：
```xml
<available_skills>
<skill name="1password" emoji="🔐">...</skill>
<skill name="github" emoji="🐙">...</skill>
</available_skills>
```

**限制与防护**：
- 最大技能数量：150 个（配置 `skills.limits.maxSkillsInPrompt`）
- 最大字符数：30K（配置 `skills.limits.maxSkillsPromptChars`）
- 可见性控制：通过 YAML frontmatter 的 `when` 条件动态控制是否加载。

### 3.4 阶段四：智能字符裁剪

**核心文件**：`src/agents/pi-embedded-helpers/bootstrap.ts`

为了防止上下文溢出模型的 Token 限制，OpenClaw 会对文件进行硬性裁剪：

**默认限制配置**：
```yaml
agents:
  defaults:
    bootstrapMaxChars: 20000        # 单文件最大字符数
    bootstrapTotalMaxChars: 150000  # 上下文总限额
```

**智能裁剪策略**：
如果文件超限，系统并非一刀切，而是采用**头尾保留策略**：
- **头部保留 70%**：通常包含核心定义、配置参数等关键信息。
- **尾部保留 20%**：通常包含示例代码、注释等参考信息。
- **中间截断 10%**：替换为 `<truncated>` 标记。

**输出格式**：
```typescript
{
  path: "AGENTS.md",
  content: "[头部 70% 内容...]\n<truncated>\n[尾部 20% 内容...]"
}
```

这种策略保证了即使在严重受限的 Token 预算下，Agent 也能看到文档最好最重要的部分。

### 3.5 阶段五：终极 System Prompt 拼装

**核心文件**：`src/agents/system-prompt.ts`（`buildAgentSystemPrompt` 函数）

最后一步，系统会将所有元素按严格顺序拼装成一个巨型 Prompt 传给大模型。

**拼装顺序（从上到下）**：

1. **Identity（身份底色）**
   ```
   You are a personal assistant running inside OpenClaw.
   ```

2. **Tooling（工具清单）**
   - 列出所有可用工具及其摘要（Bash, Edit, Glob, Grep, Read, Write, WebFetch 等）

3. **Tool Call Style（工具调用风格）**
   - 指导 Agent 如何描述工具调用（是否要 Narrative 描述）

4. **Safety（安全准则）**
   - 基于 Constitutional AI 的安全原则

5. **Skills（技能 XML）**
   - 注入阶段三生成的 `<available_skills>XML 块`

6. **Memory Recall（记忆召回）**
   - memory_search/memory_get 工具的使用指导

7. **Workspace（工作区上下文）**
   - **这是最核心的部分！**
   - 所有加载并裁剪后的 Workspace 文件（`AGENTS.md`, `SOUL.md`, `TOOLS.md` 等）都会被挂载在 `# Project Context` 标题之下。

8. **Documentation（文档链接）**
   - https://docs.openclaw.ai/ 相关链接

9. **Sandbox（沙盒信息）**
   - 如果容器化运行，注明环境差异

10. **Authorized Senders（授权发送者）**
    - 所有者的手机号码、ID 等白名单信息

11. **Current Date & Time（时间信息）**
    - 当前系统时间和时区（基于 `USER.md` 或系统默认）

12. **Messaging（消息协议）**
    - 跨会话消息传递、message 工具使用说明

13. **Voice（语音功能）**
    - TTS（文本转语音）提示

14. **Reactions（反应机制）**
    - Telegram/Signal 等平台的表情符号反应 guidance

15. **Reasoning Format（推理格式）**
    - `::<reasoning>` 和 `<final>` 标签的使用规范（如果启用）

16. **Silent Replies（静默回复）**
    - `SILENT_REPLY_TOKEN` 的使用场景

17. **Heartbeats（心跳协议）**
    - `HEARTBEAT_OK` 协议说明

18. **Runtime（运行时元数据）**
    ```
    Runtime: agent=main | host=MacBook-Pro | repo=/path/to/repo |
             os=Darwin 23.0.0 (arm64) | node=v22.0.0 |
             model=anthropic/claude-opus-4 | shell=/bin/zsh |
             channel=telegram | capabilities=inlineButtons | thinking=off
    ```

**Prompt 模式**：
系统支持三种 Prompt 模式，可根据 Session 类型动态切换：
- `"full"`：包含所有 18 个区段（默认用于主会话）
- `"minimal"`：削减部分区段（用于子会话以节省 Token）
- `"none"`：仅保留 Identity 单行（用于极简场景）

## 四、 运行时变量注入

除了 Workspace 文件和 Skills，以下**运行时动态变量**也会被注入上下文：

| 变量 | 来源 | 作用 |
| :--- | :--- | :--- |
| `repoRoot` | `findGitRoot()` 或配置 `agents.defaults.repoRoot` | 告知 Agent 当前工作目录的绝对路径 |
| `userTimezone` | 配置 `agents.defaults.userTimezone` 或系统默认 | 用于正确显示时间 |
| `timeFormat` | 配置 `agents.defaults.timeFormat`（12-hour/24-hour/auto） | 决定时间显示格式 |
| `shell` | 系统环境变量 `$SHELL` | 告知 Agent 当前使用的 Shell（如 `/bin/zsh`） |
| `model` | 当前会话使用的模型 | 用于对比默认模型和行为调整 |
| `hostname` | 系统主机名 | 用于日志和调试 |
| `os` | 系统架构（如 `Darwin arm64`） | 指导平台特定的命令选择 |

## 五、 完整示例：Context 组装全过程

让我们通过一个具体例子看 Context 如何从头到尾组装：

**用户输入**：`help me debug the auth failure`

**系统内部流程**：

```
1. [Workspace Loading]
   读取 ~/.openclaw/workspace/ 下的文件：
   ✓ AGENTS.md (18,500 字符)
   ✓ SOUL.md (2,100 字符)
   ✓ TOOLS.md (8,300 字符)
   ✓ USER.md (450 字符)
   ✗ MEMORY.md (跳过，当前是 Subagent)
   ✓ IDENTITY.md (800 字符)

2. [Hook Intervention]
   bootstrap-extra-files Hook 触发：
   + 注入 docs/REFERENCE-ARCHITECTURE.md (12,000 字符)

3. [Skills Discovery]
   扫描技能目录：
   ✗ apple-notes (缺失依赖，跳过)
   ✓ github (已配置 op CLI，加载)
   ✓ 1password (已登录，加载)
   格式化为 <available_skills>...</available_skills> (5,200 字符)

4. [Context Truncation]
   检查总大小：
   当前总字符数：~47,350 字符
   限制：150,000 字符
   结论：无需裁剪，全部保留

5. [System Prompt Assembly]
   拼装最终 Prompt（约 55,000 tokens）：
   ```
   You are a personal assistant running inside OpenClaw.

   ...

   # Project Context
   [AGENTS.md 完整内容]
   [SOUL.md 完整内容]
   [TOOLS.md 完整内容]
   [REFERENCE-ARCHITECTURE.md 完整内容]
   ...

   <available_skills>
   <skill name="github">...</skill>
   <skill name="1password">...</skill>
   </available_skills>

   ...

   Runtime: agent=subagent | host=MacBook-Pro |
            repo=/Users/user/my-project | os=Darwin arm64 |
            shell=/bin/zsh | model=anthropic/claude-opus-4
   ```

6. [Send to LLM]
   将完整 Prompt  Anthropic API

7. [Agent Responsing]
   Agent 基于上下文理解："就在 /Users/user/my-project 目录下，
   我有 GitHub 技能可用，但不能用 sed 修改文件，要用 Edit 工具"
   最终回复：给出符合调试步骤的建议
```

## 六、 总结

**OpenClaw 的 Agent 是通过以下机制实现"上下文感知"的**：

1. **Workspace 为中心**：所有指导 Agent 行为的文本配置都集中在 `~/.openclaw/workspace/` 目录下，形成静态的认知基础。

2. **动态过滤与注入**：通过 Hooks 和会话类型检测，系统智能地决定哪些配置应该被注入，哪些可以省略以节省 Token。

3. **层叠式组装**：Context 不是一次性抛给 Agent 的，而是通过 Workspace 文件 → Skills → 运行时变量 → System Prompt 这四个层级逐步构建。

4. **Token 预算意识**：系统内置了智能裁剪机制，确保即使在严重受限的情况下，Agent 也能看到最关键的信息。

通过这套机制，每一次交互，Agent 都动态地重新回答三个核心问题：
- **"我是谁？"** → 来自 `IDENTITY.md`, `SOUL.md`
- **"我在哪？"** → 来自 `repoRoot`, Runtime 变量
- **"我能干什么？"** → 来自 `TOOLS.md`, Skills 列表
- **"有什么规矩？"** → 来自 `AGENTS.md`, Safety 段落

最终，一个既懂技术规范、又懂项目背景、还知道如何安全操作的 Agent 工作态就形成了。
