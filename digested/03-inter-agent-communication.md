# 配置指南 3：Agent 间通信和协作

## 概述

本文档详细说明如何让不同的 agent 使用不同的模型，以及 agent 之间如何通信和协作完成复杂任务。

**配置文件位置**：`~/.openclaw/openclaw.json`

---

## 1. 核心概念

### 1.1 Agent 间通信机制

OpenClaw 提供两种 agent 间通信方式：

1. **子 Agent 系统（Subagent System）**
   - 主 agent 可以生成子 agent 执行特定任务
   - 支持嵌套（orchestrator 模式）
   - 自动管理生命周期

2. **Agent 间消息（Agent-to-Agent Messaging）**
   - Agent 可以直接向其他 agent 发送消息
   - 支持同步等待响应
   - 可配置访问控制

### 1.2 使用场景

- **任务分解**：主 agent 将复杂任务分配给多个子 agent
- **专业化协作**：不同专长的 agent 协同工作
- **并行处理**：同时执行多个独立任务
- **编排模式**：orchestrator agent 协调多个 worker agent

---

## 2. 为不同 Agent 配置不同模型

### 2.1 基础配置

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5"
    },
    "list": [
      {
        "id": "orchestrator",
        "name": "编排者",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": ["anthropic/claude-sonnet-4-5"]
        }
      },
      {
        "id": "coder",
        "name": "代码专家",
        "model": "anthropic/claude-sonnet-4-5"
      },
      {
        "id": "researcher",
        "name": "研究员",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": ["openai/gpt-4-turbo"]
        }
      },
      {
        "id": "writer",
        "name": "写作助手",
        "model": "openai/gpt-4-turbo"
      }
    ]
  }
}
```

### 2.2 模型选择策略

#### 策略 1：按任务复杂度

```json
{
  "agents": {
    "list": [
      {
        "id": "complex-tasks",
        "model": "anthropic/claude-opus-4-6",
        "comment": "处理复杂推理任务"
      },
      {
        "id": "simple-tasks",
        "model": "anthropic/claude-sonnet-4-5",
        "comment": "处理简单快速任务"
      }
    ]
  }
}
```

#### 策略 2：按专业领域

```json
{
  "agents": {
    "list": [
      {
        "id": "code-expert",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["code_execution", "git", "file_operations"]
      },
      {
        "id": "research-expert",
        "model": "anthropic/claude-opus-4-6",
        "skills": ["web_search", "arxiv", "wikipedia"]
      },
      {
        "id": "creative-writer",
        "model": "openai/gpt-4-turbo",
        "skills": ["writing", "translation"]
      }
    ]
  }
}
```

#### 策略 3：成本优化

```json
{
  "agents": {
    "list": [
      {
        "id": "premium",
        "model": {
          "primary": "anthropic/claude-opus-4-6",
          "fallbacks": ["anthropic/claude-sonnet-4-5"]
        },
        "comment": "高质量，高成本"
      },
      {
        "id": "standard",
        "model": "anthropic/claude-sonnet-4-5",
        "comment": "平衡质量和成本"
      },
      {
        "id": "budget",
        "model": {
          "primary": "openai/gpt-3.5-turbo",
          "fallbacks": ["ollama/llama3.1:70b"]
        },
        "comment": "低成本"
      }
    ]
  }
}
```

---

## 3. 子 Agent 系统（Subagent System）

### 3.1 基础配置

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 8,
        "maxSpawnDepth": 1,
        "maxChildrenPerAgent": 5,
        "archiveAfterMinutes": 60,
        "model": "anthropic/claude-sonnet-4-5",
        "thinking": "low",
        "runTimeoutSeconds": 300,
        "announceTimeoutMs": 30000
      }
    }
  }
}
```

**字段说明**：

| 字段 | 说明 | 默认值 | 范围 |
|------|------|--------|------|
| `maxConcurrent` | 全局最大并发子 agent 数 | `1` | 1+ |
| `maxSpawnDepth` | 最大嵌套深度 | `1` | 1-5 |
| `maxChildrenPerAgent` | 每个 agent 最多子 agent 数 | `5` | 1-20 |
| `archiveAfterMinutes` | 自动归档延迟（分钟） | `60` | 1+ |
| `model` | 子 agent 默认模型 | 继承父 agent | - |
| `thinking` | 思考级别 | `\"low\"` | `\"off\"`, `\"low\"`, `\"medium\"`, `\"high\"` |
| `runTimeoutSeconds` | 运行超时（秒，0=无限制） | `0` | 0+ |
| `announceTimeoutMs` | 通知超时（毫秒） | `30000` | 1000+ |

### 3.2 嵌套深度说明

#### Depth 1（默认）：简单任务分解

```
Main Agent (depth 0)
  └── Subagent 1 (depth 1) - 不能再生成子 agent
  └── Subagent 2 (depth 1) - 不能再生成子 agent
  └── Subagent 3 (depth 1) - 不能再生成子 agent
```

**配置**：
```json
{
  "subagents": {
    "maxSpawnDepth": 1
  }
}
```

#### Depth 2：Orchestrator 模式

```
Main Agent (depth 0)
  └── Orchestrator (depth 1) - 可以生成子 agent
      ├── Worker 1 (depth 2) - 不能再生成
      ├── Worker 2 (depth 2) - 不能再生成
      └── Worker 3 (depth 2) - 不能再生成
```

**配置**：
```json
{
  "subagents": {
    "maxSpawnDepth": 2
  }
}
```

### 3.3 使用子 Agent

#### 在对话中使用

用户可以通过 `/spawn` 命令或让 agent 自动使用 `sessions_spawn` 工具：

```
用户: 帮我分析这个项目的代码质量，并生成报告

Agent: 我会创建几个子 agent 来并行处理：
1. 代码审查 agent
2. 测试覆盖率分析 agent  
3. 文档质量检查 agent
4. 报告生成 agent

[自动调用 sessions_spawn 工具]
```

#### 工具调用示例

```json
{
  "tool": "sessions_spawn",
  "input": {
    "task": "审查 src/ 目录下的所有 Python 代码，检查代码质量问题",
    "label": "code-review",
    "model": "anthropic/claude-sonnet-4-5",
    "thinking": "medium",
    "runTimeoutSeconds": 600,
    "mode": "run",
    "cleanup": "delete"
  }
}
```

### 3.4 Per-Agent 子 Agent 配置

```json
{
  "agents": {
    "list": [
      {
        "id": "orchestrator",
        "model": "anthropic/claude-opus-4-6",
        "subagents": {
          "maxSpawnDepth": 2,
          "maxChildrenPerAgent": 10,
          "model": "anthropic/claude-sonnet-4-5",
          "allowAgents": ["worker-1", "worker-2", "worker-3"]
        }
      },
      {
        "id": "worker-1",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["code_execution"]
      },
      {
        "id": "worker-2",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["web_search"]
      },
      {
        "id": "worker-3",
        "model": "openai/gpt-4-turbo",
        "skills": ["writing"]
      }
    ]
  }
}
```

**`allowAgents` 说明**：
- 限制可以生成哪些 agent 作为子 agent
- 省略 = 可以生成任何 agent
- 空数组 = 禁止生成子 agent

---

## 4. Agent 间消息（Agent-to-Agent Messaging）

### 4.1 启用 Agent 间通信

```json
{
  "agents": {
    "defaults": {
      "tools": {
        "agentToAgent": {
          "enabled": true,
          "allow": [
            "orchestrator",
            "worker-1",
            "worker-2"
          ]
        }
      }
    }
  }
}
```

**字段说明**：

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `enabled` | 启用 agent 间消息 | `false` |
| `allow` | 允许使用此功能的 agent ID 列表 | `[]` |

**注意**：
- `allow` 是字符串数组，列出可以发送和接收 agent 间消息的 agent ID
- 如果 agent A 和 agent B 都在 `allow` 列表中，它们就可以互相通信
- 空数组 `[]` 表示禁用所有 agent 间通信

### 4.2 通信流程

```
Agent A (orchestrator)
  │
  ├─ sessions_send ──> Agent B (worker)
  │                      │
  │                      ├─ 处理任务
  │                      │
  │  <── 响应 ──────────┘
  │
  └─ 继续处理
```

### 4.3 使用 `sessions_send` 工具

```json
{
  "tool": "sessions_send",
  "input": {
    "agentId": "worker-1",
    "message": "请分析 data.csv 文件并返回统计摘要",
    "timeoutSeconds": 60
  }
}
```

**参数说明**：

| 参数 | 必需 | 说明 |
|------|------|------|
| `sessionKey` | ❌ | 目标会话键（精确指定） |
| `label` | ❌ | 目标会话标签（模糊匹配） |
| `agentId` | ❌ | Agent ID（配合 label 使用） |
| `message` | ✅ | 要发送的消息 |
| `timeoutSeconds` | ❌ | 等待响应超时（0 = 不等待） |

### 4.4 Agent 间消息的使用场景

Agent 间消息适用于以下场景：

**场景 1：任务委托**
```
Orchestrator -> Worker: "分析这个数据集并返回结果"
Worker -> Orchestrator: "分析完成，结果如下：..."
```

**场景 2：信息查询**
```
Agent A -> Agent B: "你那边的任务进度如何？"
Agent B -> Agent A: "已完成 80%，预计 5 分钟完成"
```

**场景 3：协作决策**
```
Agent A -> Agent B: "我建议使用方案 X，你觉得呢？"
Agent B -> Agent A: "方案 X 可行，但建议加上 Y 优化"
```

**注意**：
- Agent 间消息是单向的，发送后等待响应
- 如果需要多轮对话，需要在 agent 逻辑中实现
- 建议使用子 Agent 系统而不是复杂的 agent 间消息链

---

## 5. 完整协作配置示例

### 5.1 Orchestrator + Workers 模式

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5",
      "subagents": {
        "maxConcurrent": 8,
        "maxSpawnDepth": 2,
        "maxChildrenPerAgent": 5
      },
      "tools": {
        "agentToAgent": {
          "enabled": true,
          "allow": ["main", "orchestrator", "code-worker", "research-worker", "write-worker"]
        }
      }
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "name": "主 Agent",
        "model": "anthropic/claude-opus-4-6",
        "subagents": {
          "allowAgents": ["orchestrator"]
        }
      },
      {
        "id": "orchestrator",
        "name": "任务编排者",
        "model": "anthropic/claude-opus-4-6",
        "subagents": {
          "maxSpawnDepth": 2,
          "maxChildrenPerAgent": 10,
          "model": "anthropic/claude-sonnet-4-5",
          "allowAgents": ["code-worker", "research-worker", "write-worker"]
        },
        "identity": {
          "name": "编排者",
          "description": "我负责分解复杂任务并协调多个专业 agent 完成工作。"
        }
      },
      {
        "id": "code-worker",
        "name": "代码工作者",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["code_execution", "git", "file_operations"],
        "identity": {
          "name": "代码专家",
          "description": "我专注于代码编写、审查和执行。"
        }
      },
      {
        "id": "research-worker",
        "name": "研究工作者",
        "model": "anthropic/claude-opus-4-6",
        "skills": ["web_search", "web_fetch", "arxiv", "wikipedia"],
        "identity": {
          "name": "研究员",
          "description": "我专注于信息收集和研究。"
        }
      },
      {
        "id": "write-worker",
        "name": "写作工作者",
        "model": "openai/gpt-4-turbo",
        "skills": ["writing", "translation"],
        "identity": {
          "name": "写作助手",
          "description": "我专注于文档编写和内容创作。"
        }
      }
    ]
  },
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": [
        "orchestrator",
        "code-worker",
        "research-worker",
        "write-worker"
      ]
    }
  }
}
```

### 5.2 专业化团队模式

```json
{
  "agents": {
    "list": [
      {
        "id": "project-manager",
        "name": "项目经理",
        "model": "anthropic/claude-opus-4-6",
        "subagents": {
          "maxSpawnDepth": 2,
          "allowAgents": ["backend-dev", "frontend-dev", "qa-engineer", "devops"]
        }
      },
      {
        "id": "backend-dev",
        "name": "后端开发",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["code_execution", "database", "api"],
        "workspace": "/Users/ethan/projects/backend"
      },
      {
        "id": "frontend-dev",
        "name": "前端开发",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["code_execution", "web", "ui"],
        "workspace": "/Users/ethan/projects/frontend"
      },
      {
        "id": "qa-engineer",
        "name": "QA 工程师",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["testing", "code_execution"],
        "workspace": "/Users/ethan/projects/tests"
      },
      {
        "id": "devops",
        "name": "DevOps 工程师",
        "model": "anthropic/claude-sonnet-4-5",
        "skills": ["docker", "kubernetes", "ci_cd"],
        "workspace": "/Users/ethan/projects/infra"
      }
    ]
  }
}
```

---

## 6. 工作流示例

### 6.1 复杂任务分解

**用户请求**：
```
帮我创建一个完整的 Web 应用，包括：
1. 后端 API（Python FastAPI）
2. 前端界面（React）
3. 数据库设计（PostgreSQL）
4. 部署配置（Docker）
```

**Main Agent 响应**：
```
我会创建一个 orchestrator 来协调这个项目：

[调用 sessions_spawn 创建 orchestrator]
```

**Orchestrator 执行**：
```
我将任务分解为 4 个并行子任务：

1. [sessions_spawn] Backend Developer
   - 任务：设计并实现 FastAPI 后端
   - 模型：claude-sonnet-4-5
   
2. [sessions_spawn] Frontend Developer
   - 任务：创建 React 前端界面
   - 模型：claude-sonnet-4-5
   
3. [sessions_spawn] Database Designer
   - 任务：设计 PostgreSQL 数据库架构
   - 模型：claude-opus-4-6
   
4. [sessions_spawn] DevOps Engineer
   - 任务：创建 Docker 配置和部署脚本
   - 模型：claude-sonnet-4-5

等待所有子任务完成后，我会整合结果并生成最终报告。
```

### 6.2 迭代协作

**Orchestrator**：
```
[sessions_send to backend-dev]
"请实现用户认证 API"

[等待响应]

[sessions_send to frontend-dev]
"后端 API 已就绪，请实现登录界面，API 端点：/api/auth/login"

[等待响应]

[sessions_send to qa-engineer]
"请测试登录流程，前后端都已完成"
```

---

## 7. 子 Agent 管理

### 7.1 查看子 Agent

```bash
# 在对话中使用
/subagents list

# 或使用 subagents 工具
```

### 7.2 控制子 Agent

```bash
# 停止子 agent
/subagents kill <label>

# 重定向子 agent
/subagents steer <label> "新的指令"

# 查看子 agent 日志
/subagents log <label>

# 向子 agent 发送消息
/subagents send <label> "消息内容"
```

### 7.3 子 Agent 生命周期

```
创建 (sessions_spawn)
  ↓
运行中 (active)
  ↓
完成 (completed) ──> 自动归档 (60分钟后)
  ↓
归档 (archived)
```

---

## 8. 高级配置

### 8.1 沙箱模式

```json
{
  "agents": {
    "list": [
      {
        "id": "sandboxed-agent",
        "sandbox": {
          "enabled": true,
          "sessionToolsVisibility": "spawned",
          "allowNetworkAccess": false,
          "allowFileSystem": true,
          "allowedPaths": ["/tmp", "/Users/ethan/sandbox"]
        }
      }
    ]
  }
}
```

### 8.2 资源限制

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 8,
        "maxChildrenPerAgent": 5,
        "runTimeoutSeconds": 300
      },
      "maxConcurrent": 4
    }
  }
}
```

### 8.3 通知配置

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "announceTimeoutMs": 30000,
        "announceFormat": "detailed"
      }
    }
  }
}
```

---

## 9. 最佳实践

### 9.1 模型选择

- **Orchestrator**：使用高质量模型（Opus）进行任务规划
- **Workers**：使用性价比模型（Sonnet）执行具体任务
- **Research**：使用高质量模型（Opus）进行深度分析
- **Simple Tasks**：使用快速模型（Sonnet/GPT-3.5）

### 9.2 并发控制

```json
{
  "subagents": {
    "maxConcurrent": 8,        // 全局限制
    "maxChildrenPerAgent": 5   // 单个 agent 限制
  }
}
```

### 9.3 超时设置

```json
{
  "subagents": {
    "runTimeoutSeconds": 300,     // 5 分钟
    "announceTimeoutMs": 30000    // 30 秒
  }
}
```

### 9.4 错误处理

- 设置合理的 `fallbacks` 模型
- 使用 `runTimeoutSeconds` 防止卡死
- 监控 `maxConcurrent` 避免资源耗尽

---

## 10. 监控和调试

### 10.1 查看日志

```bash
# Gateway 日志
tail -f ~/.openclaw/logs/gateway.log

# 特定 agent 的会话
ls ~/.openclaw/agents/orchestrator/sessions/

# 子 agent 会话
ls ~/.openclaw/agents/orchestrator/sessions/ | grep subagent
```

### 10.2 性能监控

```bash
# 查看活跃会话
pnpm openclaw sessions list

# 查看 agent 状态
pnpm openclaw status --deep
```

### 10.3 调试配置

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "announceFormat": "detailed",
        "logLevel": "debug"
      }
    }
  }
}
```

---

## 11. 常见问题

### Q1: 子 agent 可以访问父 agent 的会话历史吗？

不能。子 agent 有独立的会话，但可以通过 `sessions_send` 与父 agent 通信。

### Q2: 如何限制某个 agent 只能生成特定类型的子 agent？

使用 `subagents.allowAgents` 配置。

### Q3: Agent 间消息是同步还是异步？

可以选择：
- `timeoutSeconds > 0`：同步等待响应
- `timeoutSeconds = 0`：异步发送，不等待

### Q4: 如何防止 agent 间通信死循环？

建议：
- 在 agent 逻辑中设计清晰的通信协议
- 使用子 Agent 系统而不是复杂的 agent 间消息链
- 在发送消息时设置合理的 `timeoutSeconds`
- 监控日志，及时发现异常通信模式

### Q5: 子 agent 的成本如何计算？

每个子 agent 独立计费，查看 `/status` 或日志中的 token 使用统计。

---

## 12. 相关配置文件

- **主配置**：`~/.openclaw/openclaw.json`
- **Agent 状态**：`~/.openclaw/agents/<agent-id>/`
- **子 Agent 会话**：`~/.openclaw/agents/<agent-id>/sessions/agent:<id>:subagent:<uuid>.jsonl`
- **日志**：`~/.openclaw/logs/gateway.log`

---

## 13. 配置检查清单

- [ ] 为不同 agent 配置了合适的模型
- [ ] 设置了 `subagents.maxSpawnDepth`（1 或 2）
- [ ] 配置了 `subagents.maxConcurrent` 和 `maxChildrenPerAgent`
- [ ] 如需 agent 间通信，启用了 `tools.agentToAgent.enabled`
- [ ] 将需要通信的 agent ID 添加到 `tools.agentToAgent.allow` 列表
- [ ] 测试了子 agent 创建和通信流程

---

## 14. 使用内置工具验证 Agent 间通信配置

配置完成后，**强烈建议使用 OpenClaw 内置诊断工具验证**：

```bash
# 全面健康检查（包含 agent 工具配置验证）
openclaw doctor

# 如果发现问题，自动修复
openclaw doctor --fix

# 查看 agent 状态和配置
openclaw status --deep

# 检查 Gateway Memory 状态（子 agent 依赖）
openclaw memory status --deep

# 查看 agent 状态目录
ls -la ~/.openclaw/agents/
```

**诊断重点**：
- `openclaw doctor` 会检查 agent 工具配置的完整性
- `openclaw memory status --deep` 验证 Gateway Memory 是否正常（子 agent 系统依赖）
- `openclaw status --deep` 显示已配置的 agent 和工具状态
- 检查日志中的子 agent 创建和通信记录：
  ```bash
  grep -iE "spawn|subagent|agent-to-agent" ~/.openclaw/logs/gateway.log | tail -n 50
  ```

**测试子 Agent 系统**：
1. 发送需要创建子 agent 的任务（如"帮我同时研究 3 个主题"）
2. 检查日志确认子 agent 创建成功
3. 查看子 agent 会话文件：
   ```bash
   ls -la ~/.openclaw/agents/main/sessions/ | grep subagent
   ```
4. 验证子 agent 完成后正确返回结果

**测试 Agent 间消息**：
1. 配置两个 agent 并启用 `agentToAgent`
2. 将两个 agent 的 ID 都添加到 `allow` 列表
3. 从一个 agent 向另一个发送消息
4. 检查日志确认消息传递和响应

详细的诊断和故障排查指南，请参阅 [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)。

---

**相关文档**：
- [01-models-and-providers.md](./01-models-and-providers.md) - 模型配置
- [02-agents-and-routing.md](./02-agents-and-routing.md) - Agent 和路由配置
- [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md) - 诊断与故障排查
