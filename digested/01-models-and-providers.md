# 配置指南 1：模型提供商定义

## 概述

本文档详细说明如何在 OpenClaw 中定义可用的模型提供商（官方和中转）。

**核心理念**：
- **Provider 层**：只负责定义"有哪些模型可用"（本文档）
- **Agent 层**：负责决定"用哪些模型，按什么顺序"（见文档 2 和 3）

**配置文件位置**：`~/.openclaw/openclaw.json`

**💡 进阶阅读**：[05-model-api-deep-dive.md](./05-model-api-deep-dive.md) - 深入理解 API 协议选择和模型目录机制

---

## 1. 配置分层理念

### 1.1 两层分离

```
┌──────────────────────────────────────────┐
│  第一层：models.providers                 │
│  ────────────────────────────────────    │
│  作用：定义"模型池"                        │
│  内容：                                   │
│    - anthropic (官方 API)                 │
│    - openrouter (中转服务)                │
│    - api2d (中转服务)                     │
│    - ollama (本地模型)                    │
│                                          │
│  这一层不关心谁用、怎么用                  │
└──────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│  第二层：agents[].model                   │
│  ────────────────────────────────────    │
│  作用：每个 Agent 自主选择                 │
│  示例：                                   │
│    Agent A: 用 1 个模型                   │
│      - anthropic/claude-opus-4-6         │
│                                          │
│    Agent B: 用 2 个模型（有备用）          │
│      - primary: anthropic/claude-sonnet  │
│      - fallback: openrouter/claude       │
│                                          │
│    Agent C: 用 3 个模型（多重备用）        │
│      - primary: anthropic/opus           │
│      - fallbacks: [openrouter/opus,      │
│                    api2d/sonnet]         │
└──────────────────────────────────────────┘
```

### 1.2 为什么这样设计？

**优点**：
1. **Provider 配置稳定**：定义一次，所有 Agent 共享
2. **Agent 灵活选择**：每个 Agent 根据需求自主决定
3. **易于维护**：修改 API key 只需改一处
4. **清晰分工**：配置职责明确

---

## 2. Provider 配置（定义模型池）

### 2.1 基础结构

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "provider-name": {
        "baseUrl": "https://api.example.com",
        "apiKey": "${ENV_VAR}",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          { /* 模型定义 */ }
        ]
      }
    }
  }
}
```

**`mode` 选项**：
- `"merge"`（推荐）：你的 provider 与内置 provider 合并
- `"replace"`：完全替换内置 provider（慎用）

### 2.2 Provider 字段说明

| 字段 | 必需 | 说明 | 示例 |
|------|------|------|------|
| `baseUrl` | ✅ | API 端点 URL | `"https://api.anthropic.com"` |
| `apiKey` | ❌ | API 密钥 | `"${ANTHROPIC_API_KEY}"` |
| `auth` | ❌ | 认证方式 | `"api-key"`, `"oauth"`, `"aws-sdk"` |
| `api` | ❌ | API 适配器 | `"anthropic-messages"`, `"openai-completions"` |
| `authHeader` | ❌ | 是否发送认证头 | `true`, `false` |
| `headers` | ❌ | 自定义 HTTP 头 | `{"X-Custom": "value"}` |
| `models` | ✅ | 模型定义数组 | 见下文 |

### 2.3 Model 字段说明

```json
{
  "models": [
    {
      "id": "claude-opus-4-6",
      "name": "Claude Opus 4",
      "reasoning": true,
      "input": ["text", "image"],
      "cost": {
        "input": 15.0,
        "output": 75.0,
        "cacheRead": 1.5,
        "cacheWrite": 18.75
      },
      "contextWindow": 200000,
      "maxTokens": 16384
    }
  ]
}
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `id` | ✅ | 模型标识符（用于引用：`provider/id`） |
| `name` | ✅ | 显示名称 |
| `reasoning` | ❌ | 是否支持推理模式 |
| `input` | ❌ | 支持的输入：`["text"]` 或 `["text", "image"]` |
| `cost` | ❌ | 每百万 token 价格（美元） |
| `contextWindow` | ❌ | 上下文窗口大小 |
| `maxTokens` | ❌ | 最大输出 token 数 |

### 2.4 模型引用格式

定义后，模型使用 `provider/model-id` 格式引用：
- `anthropic/claude-opus-4-6`
- `openrouter/anthropic/claude-sonnet-4-5`
- `ollama/llama3.1:70b`

---

## 3. 配置示例

### 3.1 官方 Provider（Anthropic）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.anthropic.com",
        "apiKey": "${ANTHROPIC_API_KEY}",
        "auth": "api-key",
        "api": "anthropic-messages",
        "authHeader": true,
        "models": [
          {
            "id": "claude-opus-4-6",
            "name": "Claude Opus 4",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": {
              "input": 15.0,
              "output": 75.0,
              "cacheRead": 1.5,
              "cacheWrite": 18.75
            },
            "contextWindow": 200000,
            "maxTokens": 16384
          },
          {
            "id": "claude-sonnet-4-5",
            "name": "Claude Sonnet 4.5",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {
              "input": 3.0,
              "output": 15.0,
              "cacheRead": 0.3,
              "cacheWrite": 3.75
            },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

**说明**：
- 定义了 Anthropic 官方 API
- 提供了 2 个模型：Opus 和 Sonnet
- Agent 可以选择用其中任何一个，或两个都用（作为主+备）

### 3.2 中转 Provider（OpenRouter）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "openrouter": {
        "baseUrl": "https://openrouter.ai/api/v1",
        "apiKey": "${OPENROUTER_API_KEY}",
        "auth": "api-key",
        "api": "openai-completions",
        "authHeader": true,
        "headers": {
          "HTTP-Referer": "https://your-site.com",
          "X-Title": "OpenClaw"
        },
        "models": [
          {
            "id": "anthropic/claude-opus-4-6",
            "name": "Claude Opus (via OpenRouter)",
            "reasoning": true,
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 16384
          },
          {
            "id": "anthropic/claude-sonnet-4-5",
            "name": "Claude Sonnet (via OpenRouter)",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "openai/gpt-4-turbo",
            "name": "GPT-4 Turbo (via OpenRouter)",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

**说明**：
- OpenRouter 作为中转服务
- 提供了 3 个模型
- Agent 可以选择用 OpenRouter 作为备用

### 3.3 本地 Provider（Ollama）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "api": "openai-completions",
        "auth": "api-key",
        "apiKey": "ollama",
        "injectNumCtxForOpenAICompat": true,
        "models": [
          {
            "id": "llama3.1:70b",
            "name": "Llama 3.1 70B",
            "reasoning": false,
            "input": ["text"],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 128000,
            "maxTokens": 4096
          },
          {
            "id": "qwen2.5:32b",
            "name": "Qwen 2.5 32B",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 32768,
            "maxTokens": 2048
          }
        ]
      }
    }
  }
}
```

**说明**：
- 本地运行的 Ollama
- 免费，但需要本地资源
- Agent 可以选择用作低成本备用

### 3.4 完整示例：多个 Provider

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.anthropic.com",
        "apiKey": "${ANTHROPIC_API_KEY}",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-6",
            "name": "Claude Opus 4",
            "contextWindow": 200000,
            "maxTokens": 16384
          },
          {
            "id": "claude-sonnet-4-5",
            "name": "Claude Sonnet 4.5",
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      },
      "openrouter": {
        "baseUrl": "https://openrouter.ai/api/v1",
        "apiKey": "${OPENROUTER_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "anthropic/claude-opus-4-6",
            "name": "Claude Opus (OpenRouter)",
            "contextWindow": 200000,
            "maxTokens": 16384
          },
          {
            "id": "anthropic/claude-sonnet-4-5",
            "name": "Claude Sonnet (OpenRouter)",
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      },
      "api2d": {
        "baseUrl": "https://api.api2d.com/v1",
        "apiKey": "${API2D_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "claude-sonnet-4-5",
            "name": "Claude Sonnet (API2D)",
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      },
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "api": "openai-completions",
        "apiKey": "ollama",
        "models": [
          {
            "id": "llama3.1:70b",
            "name": "Llama 3.1 70B",
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

**现在 Agent 可以自由选择**：

```json
{
  "agents": {
    "list": [
      {
        "id": "agent-a",
        "model": "anthropic/claude-opus-4-6"
      },
      {
        "id": "agent-b",
        "model": {
          "primary": "anthropic/claude-sonnet-4-5",
          "fallbacks": ["openrouter/anthropic/claude-sonnet-4-5"]
        }
      },
      {
        "id": "agent-c",
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

---

## 4. 环境变量管理

### 4.1 设置环境变量

在 `~/.profile` 或 `~/.zshrc` 中：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export OPENROUTER_API_KEY="sk-or-..."
export API2D_API_KEY="..."
```

### 4.2 在配置中引用

```json
{
  "apiKey": "${ANTHROPIC_API_KEY}"
}
```

### 4.3 直接写入（不推荐）

```json
{
  "apiKey": "sk-ant-actual-key-here"
}
```

**安全提示**：
- 优先使用环境变量
- 不要将 API key 提交到 git
- 定期轮换密钥

---

## 5. 验证配置

### 5.1 检查 Provider 配置

```bash
# 查看所有 provider
pnpm openclaw config get models.providers

# 查看特定 provider
pnpm openclaw config get models.providers.anthropic

# 列出所有可用模型
pnpm openclaw models list
```

### 5.2 测试 Provider 连接

```bash
# 使用特定模型发送测试消息
pnpm openclaw message send \
  --message "Hello" \
  --model "anthropic/claude-sonnet-4-5"
```

### 5.3 查看日志

```bash
# 查看 API 调用日志
tail -f ~/.openclaw/logs/gateway.log | grep "model"
```

---

## 6. 常见 Provider 配置

### 6.1 Anthropic（官方）

```json
{
  "anthropic": {
    "baseUrl": "https://api.anthropic.com",
    "apiKey": "${ANTHROPIC_API_KEY}",
    "api": "anthropic-messages"
  }
}
```

### 6.2 OpenAI（官方）

```json
{
  "openai": {
    "baseUrl": "https://api.openai.com/v1",
    "apiKey": "${OPENAI_API_KEY}",
    "api": "openai-completions"
  }
}
```

### 6.3 OpenRouter（中转）

```json
{
  "openrouter": {
    "baseUrl": "https://openrouter.ai/api/v1",
    "apiKey": "${OPENROUTER_API_KEY}",
    "api": "openai-completions",
    "headers": {
      "HTTP-Referer": "https://your-site.com"
    }
  }
}
```

### 6.4 Cloudflare AI Gateway（代理）

```json
{
  "cloudflare": {
    "baseUrl": "https://gateway.ai.cloudflare.com/v1/YOUR_ACCOUNT/YOUR_GATEWAY/openai",
    "apiKey": "${OPENAI_API_KEY}",
    "api": "openai-completions"
  }
}
```

### 6.5 Ollama（本地）

```json
{
  "ollama": {
    "baseUrl": "http://localhost:11434/v1",
    "api": "openai-completions",
    "apiKey": "ollama",
    "injectNumCtxForOpenAICompat": true
  }
}
```

---

## 7. 高级配置

### 7.1 自定义 HTTP 头

```json
{
  "providers": {
    "custom": {
      "baseUrl": "https://api.example.com",
      "apiKey": "${API_KEY}",
      "headers": {
        "X-Custom-Header": "value",
        "X-Organization": "my-org"
      }
    }
  }
}
```

### 7.2 AWS Bedrock

```json
{
  "providers": {
    "bedrock": {
      "auth": "aws-sdk",
      "api": "bedrock",
      "models": [
        {
          "id": "anthropic.claude-3-opus-20240229-v1:0",
          "name": "Claude 3 Opus (Bedrock)"
        }
      ]
    }
  }
}
```

### 7.3 Azure OpenAI

```json
{
  "providers": {
    "azure": {
      "baseUrl": "https://YOUR_RESOURCE.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT",
      "apiKey": "${AZURE_OPENAI_API_KEY}",
      "api": "openai-completions",
      "headers": {
        "api-version": "2024-02-15-preview"
      }
    }
  }
}
```

---

## 8. 常见问题

### Q1: Provider 配置修改后需要重启吗？

A: 是的，需要重启 Gateway：
```bash
pnpm openclaw gateway restart
```

### Q2: 如何知道有哪些内置 Provider？

A: 查看配置：
```bash
pnpm openclaw models list
```

### Q3: `mode: "merge"` 和 `mode: "replace"` 的区别？

A:
- `"merge"`：你的 provider 与内置 provider 共存
- `"replace"`：只使用你定义的 provider，忽略内置

### Q4: 可以定义同名 Provider 覆盖内置的吗？

A: 可以，你的配置会覆盖同名的内置 provider。

### Q5: 如何测试 Provider 是否配置正确？

A: 使用该 provider 的模型发送测试消息：
```bash
pnpm openclaw message send \
  --message "test" \
  --model "your-provider/model-id"
```

---

## 9. 下一步

Provider 配置完成后，下一步是配置 Agent 如何选择和使用这些模型：

**阅读**：[02-agents-and-routing.md](./02-agents-and-routing.md)

在那里你会学到：
- 如何让每个 Agent 选择 1 个、2 个或多个模型
- 如何设置模型优先级（primary + fallbacks）
- 如何为不同场景配置不同的模型策略

---

## 10. 相关配置文件

- **主配置**：`~/.openclaw/openclaw.json`
- **环境变量**：`~/.profile` 或 `~/.zshrc`
- **日志**：`~/.openclaw/logs/gateway.log`

---

**配置检查清单**：
- [ ] 定义了所有需要的 provider
- [ ] 为每个 provider 配置了正确的 baseUrl
- [ ] API key 使用环境变量（安全）
- [ ] 为每个 provider 定义了可用的 models
- [ ] 测试了 provider 连接
- [ ] 查看了日志确认无错误

---

## 11. 使用内置工具验证 Provider 配置

配置完成后，**强烈建议使用 OpenClaw 内置诊断工具验证**，而不是手动检查：

```bash
# 全面健康检查（包含模型配置验证）
openclaw doctor

# 如果发现问题，自动修复
openclaw doctor --fix

# 检查 Gateway 是否正确加载了模型配置
openclaw gateway status

# 查看状态概览（包含 provider 数量和模型数量）
openclaw status --deep
```

**诊断重点**：
- `openclaw doctor` 会检查模型目录、hooks 中的模型引用是否合法
- `openclaw status --deep` 会显示已加载的 provider 数和模型数
- 如果 API key 缺失或格式错误，doctor 和日志都会给出提示

详细的诊断和故障排查指南，请参阅 [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)。
