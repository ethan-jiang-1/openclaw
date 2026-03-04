# 配置指南 2：Agent 配置和路由规则

## 概述

本文档详细说明如何配置多个 agent（每个有独立的 workspace），定义不同的角色，以及如何通过路由规则让不同的 agent 响应不同的消息。

**配置文件位置**：`~/.openclaw/openclaw.json`

---

## 1. Agent 基础概念

### 1.1 什么是 Agent？

Agent 是 OpenClaw 中的独立 AI 实例，每个 agent：
- 有独立的**会话历史**（session history）
- 有独立的**工作目录**（workspace）
- 可以使用**不同的模型**
- 可以有**不同的技能集**（skills）
- 可以有**不同的身份和行为**（identity）

### 1.2 Agent 的用途

- **角色分离**：工作 agent vs 个人 agent
- **专业化**：代码 agent、写作 agent、数据分析 agent
- **隔离**：不同项目使用不同 workspace
- **多账号**：不同消息渠道使用不同 agent

---

## 2. Agent 配置结构

### 2.1 顶层结构

```json
{
  "agents": {
    "defaults": {
      /* 全局默认配置 */
    },
    "list": [
      /* Agent 定义数组 */
    ]
  },
  "bindings": [
    /* 路由规则数组 */
  ]
}
```

### 2.2 全局默认配置（`agents.defaults`）

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5",
      "workspace": "/Users/ethan/.openclaw/workspace",
      "compaction": {
        "mode": "safeguard"
      },
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      },
      "memorySearch": {
        "enabled": true
      },
      "identity": {
        "name": "OpenClaw",
        "description": "AI assistant"
      }
    }
  }
}
```

**字段说明**：

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `model` | 默认使用的模型 | 系统默认 |
| `workspace` | 默认工作目录 | `~/.openclaw/workspace` |
| `compaction.mode` | 上下文压缩模式 | `"safeguard"` |
| `maxConcurrent` | 最大并发请求数 | `4` |
| `subagents` | 子 agent 配置 | 见下文 |
| `memorySearch` | 记忆搜索配置 | 见下文 |
| `identity` | Agent 身份配置 | 见下文 |

### 2.3 Agent 定义（`agents.list[]`）

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "name": "主 Agent",
        "workspace": "/Users/ethan/.openclaw/workspace-main",
        "agentDir": "/Users/ethan/.openclaw/agents/main",
        "model": "anthropic/claude-opus-4-6",
        "skills": ["web_search", "code_execution"],
        "identity": {
          "name": "主助手",
          "description": "我是你的主要 AI 助手"
        }
      },
      {
        "id": "work",
        "name": "工作 Agent",
        "workspace": "/Users/ethan/.openclaw/workspace-work",
        "agentDir": "/Users/ethan/.openclaw/agents/work",
        "model": "anthropic/claude-sonnet-4-5",
        "identity": {
          "name": "工作助手",
          "description": "我专注于工作相关任务"
        }
      }
    ]
  }
}
```

**字段说明**：

| 字段 | 必需 | 说明 |
|------|------|------|
| `id` | ✅ | Agent 唯一标识符 |
| `default` | ❌ | 是否为默认 agent（只能有一个） |
| `name` | ❌ | 显示名称 |
| `workspace` | ❌ | 工作目录（覆盖全局默认） |
| `agentDir` | ❌ | Agent 状态目录（会话、记忆等） |
| `model` | ❌ | 使用的模型（覆盖全局默认） |
| `skills` | ❌ | 允许的技能列表（省略 = 全部） |
| `identity` | ❌ | Agent 身份配置 |
| `subagents` | ❌ | 子 agent 配置 |
| `tools` | ❌ | 工具配置 |
| `params` | ❌ | 流参数 |

---

## 3. Agent 的模型选择

**前置条件**：先在 `models.providers` 中定义好可用的 Provider（见 [01-models-and-providers.md](./01-models-and-providers.md)）

每个 Agent 自主决定用哪些模型，以及优先级顺序。

### 3.1 只用 1 个模型（最简单）

```json
{
  "agents": {
    "list": [
      {
        "id": "simple-agent",
        "model": "anthropic/claude-sonnet-4-5"
      }
    ]
  }
}
```

**行为**：只用这一个模型，如果 API 出错就直接返回错误。

### 3.2 用 2 个模型（主 + 备）

```json
{
  "agents": {
    "list": [
      {
        "id": "reliable-agent",
        "model": {
          "primary": "anthropic/claude-sonnet-4-5",
          "fallbacks": [
            "openrouter/anthropic/claude-sonnet-4-5"
          ]
        }
      }
    ]
  }
}
```

**行为**：
1. 先用官方 Anthropic API
2. 如果失败（限流、服务器错误等），自动切换到 OpenRouter

### 3.3 用 3 个或更多模型（多重备用）

```json
{
  "agents": {
    "list": [
      {
        "id": "bulletproof-agent",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": [
            "openrouter/anthropic/claude-opus-4-6",
            "api2d/claude-sonnet-4-5",
            "ollama/llama3.1:70b"
          ]
        }
      }
    ]
  }
}
```

**行为**（按顺序尝试）：
1. 先用 Anthropic 官方 Opus
2. 失败 → OpenRouter 的 Opus
3. 再失败 → API2D 的 Sonnet
4. 全失败 → 本地 Ollama

### 3.4 不同 Agent 用不同策略

```json
{
  "agents": {
    "list": [
      {
        "id": "premium-agent",
        "comment": "高质量，多重保障",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": [
            "openrouter/anthropic/claude-opus-4-6",
            "anthropic/claude-sonnet-4-5"
          ]
        }
      },
      {
        "id": "fast-agent",
        "comment": "快速响应，无需备用",
        "model": "anthropic/claude-sonnet-4-5"
      },
      {
        "id": "budget-agent",
        "comment": "低成本，本地优先",
        "model": {
          "primary": "ollama/llama3.1:70b",
          "fallbacks": [
            "api2d/claude-sonnet-4-5"
          ]
        }
      }
    ]
  }
}
```

### 3.5 全局默认 + Agent 覆盖

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5",
        "fallbacks": ["openrouter/anthropic/claude-sonnet-4-5"]
      }
    },
    "list": [
      {
        "id": "default-agent",
        "comment": "继承默认：sonnet + openrouter 备用"
      },
      {
        "id": "special-agent",
        "comment": "覆盖默认：opus + 3 层备用",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": [
            "openrouter/anthropic/claude-opus-4-6",
            "anthropic/claude-sonnet-4-5",
            "ollama/llama3.1:70b"
          ]
        }
      }
    ]
  }
}
```

**规则**：
- 没有指定 `model` 的 Agent → 用 `agents.defaults.model`
- 指定了 `model` 的 Agent → 完全覆盖默认

### 3.6 模型别名

为常用模型设置简短别名，方便在 `/model` 命令中切换：

```json
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-opus-4-6": { "alias": "opus" },
        "anthropic/claude-sonnet-4-5": { "alias": "sonnet" },
        "openrouter/anthropic/claude-opus-4-6": { "alias": "opus-or" },
        "ollama/llama3.1:70b": { "alias": "llama" }
      }
    }
  }
}
```

在对话中切换：`/model opus` 或 `/model llama`

---

## 4. 路由规则（Bindings）

### 3.1 路由规则结构

```json
{
  "bindings": [
    {
      "agentId": "main",
      "comment": "个人飞书账号",
      "match": {
        "channel": "feishu",
        "accountId": "default"
      }
    },
    {
      "agentId": "work",
      "comment": "工作飞书账号",
      "match": {
        "channel": "feishu",
        "accountId": "work-account"
      }
    }
  ]
}
```

### 3.2 匹配规则（`match`）

```json
{
  "match": {
    "channel": "feishu",
    "accountId": "default",
    "peer": {
      "kind": "direct",
      "id": "ou_xxx"
    },
    "guildId": "guild-id",
    "teamId": "team-id",
    "roles": ["role-id-1", "role-id-2"]
  }
}
```

**字段说明**：

| 字段 | 必需 | 说明 | 示例 |
|------|------|------|------|
| `channel` | ✅ | 消息渠道 | `"feishu"`, `"discord"`, `"slack"` |
| `accountId` | ❌ | 账号 ID（`"*"` = 任意） | `"default"`, `"work"`, `"*"` |
| `peer` | ❌ | 特定对话 | 见下文 |
| `guildId` | ❌ | Discord 服务器 ID | `"123456789"` |
| `teamId` | ❌ | Slack 团队 ID | `"T123456"` |
| `roles` | ❌ | Discord 角色 ID 列表 | `["role1", "role2"]` |

**`peer` 结构**：

```json
{
  "peer": {
    "kind": "direct",    // "direct", "group", "channel"
    "id": "ou_xxx"       // 对话 ID
  }
}
```

### 3.3 路由优先级

路由规则按以下顺序匹配（从高到低）：

1. **精确对话匹配**：`peer.kind` + `peer.id` 完全匹配
2. **父对话匹配**：线程继承（论坛主题、回复线程）
3. **服务器 + 角色匹配**：Discord `guildId` + `roles`
4. **服务器匹配**：Discord `guildId`（无角色要求）
5. **团队匹配**：Slack `teamId`
6. **账号匹配**：特定 `accountId`
7. **渠道匹配**：任意账号（`accountId: "*"`）
8. **默认 agent**：`default: true` 的 agent

---

## 4. 配置示例

### 4.1 基础多 Agent 配置

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5",
      "workspace": "/Users/ethan/.openclaw/workspace"
    },
    "list": [
      {
        "id": "personal",
        "default": true,
        "name": "个人助手",
        "workspace": "/Users/ethan/.openclaw/workspace-personal"
      },
      {
        "id": "work",
        "name": "工作助手",
        "workspace": "/Users/ethan/.openclaw/workspace-work"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "personal",
      "match": {
        "channel": "feishu",
        "accountId": "personal"
      }
    },
    {
      "agentId": "work",
      "match": {
        "channel": "feishu",
        "accountId": "work"
      }
    }
  ]
}
```

### 4.2 按渠道路由

```json
{
  "agents": {
    "list": [
      {
        "id": "feishu-agent",
        "default": true
      },
      {
        "id": "discord-agent"
      },
      {
        "id": "slack-agent"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "feishu-agent",
      "match": {
        "channel": "feishu",
        "accountId": "*"
      }
    },
    {
      "agentId": "discord-agent",
      "match": {
        "channel": "discord",
        "accountId": "*"
      }
    },
    {
      "agentId": "slack-agent",
      "match": {
        "channel": "slack",
        "accountId": "*"
      }
    }
  ]
}
```

### 4.3 按对话类型路由

```json
{
  "agents": {
    "list": [
      {
        "id": "dm-agent",
        "name": "私聊助手",
        "default": true
      },
      {
        "id": "group-agent",
        "name": "群聊助手"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "dm-agent",
      "comment": "所有私聊",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "direct",
          "id": "*"
        }
      }
    },
    {
      "agentId": "group-agent",
      "comment": "所有群聊",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "group",
          "id": "*"
        }
      }
    }
  ]
}
```

### 4.4 特定对话路由

```json
{
  "agents": {
    "list": [
      {
        "id": "project-a",
        "name": "项目 A 助手",
        "workspace": "/Users/ethan/projects/project-a"
      },
      {
        "id": "project-b",
        "name": "项目 B 助手",
        "workspace": "/Users/ethan/projects/project-b"
      },
      {
        "id": "default",
        "default": true
      }
    ]
  },
  "bindings": [
    {
      "agentId": "project-a",
      "comment": "项目 A 群组",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "group",
          "id": "oc_xxx_project_a"
        }
      }
    },
    {
      "agentId": "project-b",
      "comment": "项目 B 群组",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "group",
          "id": "oc_xxx_project_b"
        }
      }
    }
  ]
}
```

### 4.5 Discord 角色路由

```json
{
  "agents": {
    "list": [
      {
        "id": "admin-agent",
        "name": "管理员助手"
      },
      {
        "id": "member-agent",
        "name": "成员助手",
        "default": true
      }
    ]
  },
  "bindings": [
    {
      "agentId": "admin-agent",
      "comment": "管理员角色",
      "match": {
        "channel": "discord",
        "guildId": "123456789",
        "roles": ["admin-role-id", "moderator-role-id"]
      }
    },
    {
      "agentId": "member-agent",
      "comment": "普通成员",
      "match": {
        "channel": "discord",
        "guildId": "123456789"
      }
    }
  ]
}
```

---

## 5. Agent 身份配置（Identity）

### 5.1 基础身份

```json
{
  "agents": {
    "list": [
      {
        "id": "coding-assistant",
        "identity": {
          "name": "代码助手",
          "description": "我是专门帮助你编写和调试代码的 AI 助手。我擅长多种编程语言，可以帮你解决技术问题。"
        }
      }
    ]
  }
}
```

### 5.2 完整身份配置

```json
{
  "identity": {
    "name": "工作助手",
    "description": "我是你的工作 AI 助手",
    "instructions": [
      "始终保持专业和礼貌",
      "优先处理紧急任务",
      "使用简洁的语言",
      "提供可操作的建议"
    ],
    "capabilities": [
      "代码审查",
      "文档编写",
      "项目管理",
      "数据分析"
    ],
    "constraints": [
      "不处理个人事务",
      "不访问敏感数据",
      "工作时间内响应"
    ]
  }
}
```

---

## 6. 会话范围配置（Session Scope）

### 6.1 DM 范围控制

```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

**选项说明**：

| 值 | 说明 | 适用场景 |
|---|------|----------|
| `"main"` | 所有私聊共享一个会话 | 单用户使用 |
| `"per-peer"` | 每个发送者独立会话 | 多用户，不区分渠道 |
| `"per-channel-peer"` | 每个渠道+发送者独立会话 | 多渠道多用户（推荐） |
| `"per-account-channel-peer"` | 每个账号+渠道+发送者独立会话 | 多账号场景 |

### 6.2 会话键构造

**私聊**：
- `main`: `agent:main:main`
- `per-peer`: `agent:main:feishu:direct:ou_xxx`
- `per-channel-peer`: `agent:main:feishu:direct:ou_xxx`

**群聊**：
- `agent:main:feishu:group:oc_xxx`

**频道**：
- `agent:main:discord:channel:123456`

---

## 7. 技能配置（Skills）

### 7.1 全局技能控制

```json
{
  "agents": {
    "defaults": {
      "skills": ["web_search", "code_execution", "file_operations"]
    }
  }
}
```

### 7.2 Per-Agent 技能

```json
{
  "agents": {
    "list": [
      {
        "id": "research-agent",
        "skills": ["web_search", "web_fetch", "wikipedia"]
      },
      {
        "id": "coding-agent",
        "skills": ["code_execution", "file_operations", "git"]
      },
      {
        "id": "general-agent",
        "skills": null  // 允许所有技能
      }
    ]
  }
}
```

**注意**：
- 省略 `skills` 字段 = 允许所有技能
- `skills: []` = 禁用所有技能
- `skills: ["skill1", "skill2"]` = 只允许指定技能

---

## 8. 工具配置（Tools）

### 8.1 工具策略

```json
{
  "agents": {
    "list": [
      {
        "id": "restricted-agent",
        "tools": {
          "policy": "allowlist",
          "allowlist": ["web_search", "calculator"],
          "denylist": []
        }
      }
    ]
  }
}
```

### 8.2 危险工具控制

```json
{
  "agents": {
    "defaults": {
      "tools": {
        "dangerous": {
          "enabled": false,
          "requireApproval": true
        }
      }
    }
  }
}
```

---

## 9. 完整配置示例

### 9.1 多角色多渠道配置

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5",
      "workspace": "/Users/ethan/.openclaw/workspace",
      "compaction": {
        "mode": "safeguard"
      }
    },
    "list": [
      {
        "id": "personal",
        "default": true,
        "name": "个人助手",
        "workspace": "/Users/ethan/.openclaw/workspace-personal",
        "model": "anthropic/claude-opus-4-6",
        "identity": {
          "name": "小助手",
          "description": "我是你的个人 AI 助手，可以帮你处理日常事务。"
        }
      },
      {
        "id": "work",
        "name": "工作助手",
        "workspace": "/Users/ethan/.openclaw/workspace-work",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["web_search", "code_execution", "file_operations"],
        "identity": {
          "name": "工作助手",
          "description": "我专注于工作相关任务，包括代码、文档和项目管理。"
        }
      },
      {
        "id": "research",
        "name": "研究助手",
        "workspace": "/Users/ethan/.openclaw/workspace-research",
        "model": "anthropic/claude-opus-4-6",
        "skills": ["web_search", "web_fetch", "wikipedia", "arxiv"],
        "identity": {
          "name": "研究助手",
          "description": "我专门帮助你进行学术研究和信息收集。"
        }
      }
    ]
  },
  "bindings": [
    {
      "agentId": "personal",
      "comment": "个人飞书账号",
      "match": {
        "channel": "feishu",
        "accountId": "default"
      }
    },
    {
      "agentId": "work",
      "comment": "工作飞书账号",
      "match": {
        "channel": "feishu",
        "accountId": "work"
      }
    },
    {
      "agentId": "research",
      "comment": "研究群组",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "group",
          "id": "oc_research_group"
        }
      }
    },
    {
      "agentId": "work",
      "comment": "Discord 工作服务器",
      "match": {
        "channel": "discord",
        "guildId": "work-guild-id"
      }
    }
  ],
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

---

## 10. 验证和测试

### 10.1 检查配置

```bash
# 查看 agent 列表
pnpm openclaw config get agents.list

# 查看路由规则
pnpm openclaw config get bindings

# 查看会话范围
pnpm openclaw config get session.dmScope
```

### 10.2 测试路由

在不同渠道/账号发送消息，观察哪个 agent 响应：

```bash
# 查看日志
tail -f ~/.openclaw/logs/gateway.log | grep "route"
```

### 10.3 查看会话

```bash
# 列出所有会话
ls ~/.openclaw/agents/*/sessions/

# 查看特定 agent 的会话
ls ~/.openclaw/agents/work/sessions/
```

---

## 11. 常见问题

### Q1: 如何知道当前使用的是哪个 agent？

在对话中使用 `/status` 命令，或查看日志。

### Q2: 可以动态切换 agent 吗？

不能直接切换，但可以：
1. 使用不同的渠道/账号（自动路由到不同 agent）
2. 使用子 agent 系统（见下一篇文档）

### Q3: Agent 之间的会话历史是隔离的吗？

是的，每个 agent 有独立的 `agentDir`，会话历史完全隔离。

### Q4: 如何为特定用户配置专属 agent？

使用 `peer` 匹配规则，指定用户的 `id`。

---

## 12. 相关配置文件

- **主配置**：`~/.openclaw/openclaw.json`
- **Agent 状态目录**：`~/.openclaw/agents/<agent-id>/`
- **会话文件**：`~/.openclaw/agents/<agent-id>/sessions/`
- **工作目录**：配置中的 `workspace` 路径

---

## 13. 使用内置工具验证 Agent 和路由配置

配置完成后，**强烈建议使用 OpenClaw 内置诊断工具验证**：

```bash
# 全面健康检查（包含 agent 配置和路由规则验证）
openclaw doctor

# 如果发现问题，自动修复
openclaw doctor --fix

# 查看 agent 状态和路由规则
openclaw status --deep

# 检查 Gateway 是否正确加载了 agent 配置
openclaw gateway status

# 查看渠道状态（验证路由是否生效）
openclaw channels status --probe
```

**诊断重点**：
- `openclaw doctor` 会检查 agent 配置的完整性、模型引用是否合法
- `openclaw status --deep` 会显示已配置的 agent 数量和当前活跃的 agent
- `openclaw channels status --probe` 可以验证消息是否能正确路由到对应的 agent
- 如果路由规则有冲突或错误，doctor 会给出警告

**测试路由规则**：
1. 从不同渠道发送测试消息
2. 检查日志确认消息被路由到正确的 agent：
   ```bash
   tail -f ~/.openclaw/logs/gateway.log | grep -i "routing\|agent"
   ```
3. 使用 `/status` 命令在对话中查看当前 agent

详细的诊断和故障排查指南，请参阅 [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)。

---

**下一步**：阅读 [03-inter-agent-communication.md](./03-inter-agent-communication.md) 了解 agent 之间如何通信和协作。
