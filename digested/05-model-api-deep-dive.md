# 模型 API 协议深度解析

本文档深入解释 OpenClaw 如何处理不同厂商的模型 API 协议，以及模型目录（catalog）的工作机制。

---

## 目录

1. [核心问题](#核心问题)
2. [API 协议类型](#api-协议类型)
3. [OpenClaw 如何选择 API 协议](#openclaw-如何选择-api-协议)
4. [模型目录系统](#模型目录系统)
5. [模型发现机制](#模型发现机制)
6. [配置模式：merge vs replace](#配置模式merge-vs-replace)
7. [实战示例](#实战示例)
8. [常见问题](#常见问题)

---

## 核心问题

### 问题 1：不同厂商的 API 协议不同

- **OpenAI** 使用 Chat Completions API
- **Anthropic** 使用 Messages API
- **Google Gemini** 使用 Generative AI API
- **其他厂商**可能兼容 OpenAI 或 Anthropic，也可能有自己的协议

**OpenClaw 如何知道用哪种协议？**

### 问题 2：模型目录的管理

- 厂商提供很多模型，但不一定都上架
- 用户可能想用未在目录中的模型
- 如何处理内置目录和用户自定义模型的关系？

---

## API 协议类型

OpenClaw 支持 **8 种 API 协议**（定义在 `src/config/types.models.ts`）：

| API 类型 | 说明 | 典型厂商 |
|---------|------|---------|
| `openai-completions` | OpenAI Chat Completions API | OpenAI, Moonshot, DeepSeek |
| `openai-responses` | OpenAI Responses API（带存储/压缩） | OpenAI |
| `openai-codex-responses` | Codex 专用 Responses API | OpenAI Codex |
| `anthropic-messages` | Anthropic Messages API | Anthropic, Minimax |
| `google-generative-ai` | Google Gemini API | Google Gemini |
| `github-copilot` | GitHub Copilot API | GitHub Copilot |
| `bedrock-converse-stream` | AWS Bedrock Converse API | AWS Bedrock |
| `ollama` | Ollama 原生 API（非 OpenAI 兼容） | Ollama |

### 协议差异示例

#### OpenAI Chat Completions API
```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "user", "content": "Hello"}
  ]
}
```

#### Anthropic Messages API
```json
{
  "model": "claude-opus-4",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "max_tokens": 1024
}
```

**关键差异**：
- Anthropic 必须提供 `max_tokens`
- 请求/响应格式略有不同
- 流式响应格式不同

---

## OpenClaw 如何选择 API 协议

### 核心机制：显式配置，不自动推断

OpenClaw **不会自动检测** API 协议，必须通过 `api` 字段显式指定。

### 配置层级

```
模型级别 api > Provider 级别 api > 默认 fallback
```

#### 1. Provider 级别 API（推荐）

```json
{
  "models": {
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimax.io/anthropic",
        "api": "anthropic-messages",  // ← Provider 默认 API
        "apiKey": "${MINIMAX_API_KEY}",
        "models": [
          {
            "id": "MiniMax-M2.5",
            "name": "MiniMax M2.5",
            // 继承 provider 的 api: "anthropic-messages"
            ...
          }
        ]
      }
    }
  }
}
```

#### 2. 模型级别 API（覆盖 Provider）

```json
{
  "models": {
    "providers": {
      "custom-provider": {
        "baseUrl": "https://api.example.com",
        "api": "openai-completions",  // Provider 默认
        "models": [
          {
            "id": "model-a",
            "name": "Model A",
            // 使用 provider 的 api: "openai-completions"
            ...
          },
          {
            "id": "model-b",
            "name": "Model B",
            "api": "anthropic-messages",  // ← 覆盖 provider 默认
            ...
          }
        ]
      }
    }
  }
}
```

#### 3. 默认 Fallback

如果既没有 provider 级别也没有模型级别的 `api` 字段：
- **OpenRouter**：自动使用 `openai-completions`
- **其他 provider**：默认使用 `openai-responses`

### 内置 Provider 的 API 映射

OpenClaw 内置了一些 provider 的 API 配置（`src/agents/models-config.providers.ts`）：

| Provider | API 类型 | 说明 |
|---------|---------|------|
| Minimax | `anthropic-messages` | 兼容 Anthropic API |
| Moonshot | `openai-completions` | 兼容 OpenAI API |
| DeepSeek | `openai-completions` | 兼容 OpenAI API |
| Qianfan | `openai-completions` | 百度千帆，兼容 OpenAI |
| Ollama | `ollama` | 原生 Ollama API |
| Bedrock | `bedrock-converse-stream` | AWS Bedrock 专用 |

**如果你使用内置 provider，不需要手动指定 `api` 字段。**

---

## 模型目录系统

### 三层模型来源

```
┌─────────────────────────────────────────┐
│  1. 内置目录（Built-in Catalog）          │
│     - OpenClaw 预定义的模型列表           │
│     - 包含主流厂商的常见模型              │
│     - 定义在 models-config.providers.ts  │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  2. 动态发现（Dynamic Discovery）         │
│     - Ollama: 查询本地模型列表            │
│     - Bedrock: AWS SDK 查询              │
│     - vLLM: HTTP API 查询                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  3. 用户配置（User Config）               │
│     - openclaw.json 中的 models.providers│
│     - 可以覆盖内置定义                    │
│     - 可以添加新模型                      │
└─────────────────────────────────────────┘
```

### 内置目录示例

OpenClaw 内置了这些 provider 的模型目录：

```typescript
// src/agents/models-config.providers.ts

function buildMinimaxProvider(): ProviderConfig {
  return {
    baseUrl: "https://api.minimax.io/anthropic",
    api: "anthropic-messages",
    models: [
      {
        id: "MiniMax-M2.5",
        name: "MiniMax M2.5",
        reasoning: true,
        input: ["text"],
        cost: {
          input: 0.3,
          output: 1.2,
          cacheRead: 0.03,
          cacheWrite: 0.12
        },
        contextWindow: 200000,
        maxTokens: 8192
      }
    ]
  };
}
```

**内置 provider 列表**：
- Minimax
- Moonshot
- DeepSeek
- Qianfan（百度千帆）
- Ollama
- Venice
- Huggingface
- vLLM
- Bedrock

---

## 模型发现机制

### 静态目录 vs 动态发现

#### 静态目录（大多数 provider）

- 模型列表硬编码在 OpenClaw 代码中
- 启动时直接加载
- 不需要网络请求
- 示例：Minimax, Moonshot, DeepSeek

#### 动态发现（部分 provider）

某些 provider 支持运行时查询模型列表：

##### 1. Ollama

```typescript
// 查询 http://localhost:11434/api/tags
{
  "models": [
    {"name": "llama3:latest", ...},
    {"name": "mistral:7b", ...}
  ]
}
```

OpenClaw 会：
1. 启动时查询 Ollama API
2. 为每个模型创建配置
3. 合并到模型目录

##### 2. AWS Bedrock

```typescript
// 使用 AWS SDK ListFoundationModelsCommand
const models = await bedrockClient.send(
  new ListFoundationModelsCommand({})
);
```

##### 3. vLLM / Huggingface

```typescript
// 查询 /v1/models 端点
const response = await fetch(`${baseUrl}/v1/models`);
const { data } = await response.json();
```

### 发现失败的处理

如果动态发现失败（网络错误、认证失败等）：
1. 回退到内置静态目录
2. 使用用户配置的模型
3. 记录警告日志，但不阻止启动

---

## 配置模式：merge vs replace

### `models.mode` 字段

```json
{
  "models": {
    "mode": "merge"  // 或 "replace"
  }
}
```

### Mode: `merge`（默认，推荐）

**行为**：合并内置目录和用户配置

```
内置目录 + 用户配置 = 最终模型列表
```

**合并规则**：

1. **相同模型 ID**：
   - 能力元数据（`input`, `reasoning`）从内置目录刷新
   - 用户覆盖（`cost`, `headers`, `compat`）保留
   - `contextWindow` 和 `maxTokens` 取**较大值**

2. **新模型**：
   - 内置目录中的新模型自动添加
   - 用户配置的新模型保留

3. **Provider 配置**：
   - `apiKey` 和 `baseUrl` 保留用户配置（如果非空）
   - `api` 字段从内置目录刷新

**示例**：

```json
// 内置目录
{
  "minimax": {
    "api": "anthropic-messages",
    "models": [
      {
        "id": "MiniMax-M2.5",
        "contextWindow": 200000,
        "cost": { "input": 0.3, "output": 1.2 }
      }
    ]
  }
}

// 用户配置
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "apiKey": "${MY_KEY}",
        "models": [
          {
            "id": "MiniMax-M2.5",
            "cost": { "input": 0.1, "output": 0.5 }  // 自定义价格
          }
        ]
      }
    }
  }
}

// 最终结果
{
  "minimax": {
    "api": "anthropic-messages",  // 从内置目录
    "apiKey": "${MY_KEY}",        // 从用户配置
    "models": [
      {
        "id": "MiniMax-M2.5",
        "contextWindow": 200000,  // 从内置目录
        "cost": { "input": 0.1, "output": 0.5 }  // 用户覆盖
      }
    ]
  }
}
```

### Mode: `replace`

**行为**：完全忽略内置目录，只使用用户配置

```
用户配置 = 最终模型列表
```

**适用场景**：
- 完全自定义的 provider
- 不想要任何内置模型
- 需要完全控制模型列表

**示例**：

```json
{
  "models": {
    "mode": "replace",
    "providers": {
      "my-custom-provider": {
        "baseUrl": "https://api.example.com",
        "api": "openai-completions",
        "apiKey": "${MY_KEY}",
        "models": [
          {
            "id": "my-model",
            "name": "My Custom Model",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 8192,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

**注意**：使用 `replace` 模式后，所有内置 provider（Anthropic, OpenAI 等）都不可用，除非你手动配置。

---

## 实战示例

### 示例 1：使用兼容 OpenAI 的中转服务

假设你有一个中转服务 `api.example.com`，兼容 OpenAI API，提供 Claude 模型：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "my-relay": {
        "baseUrl": "https://api.example.com/v1",
        "api": "openai-completions",  // ← 关键：指定 API 类型
        "apiKey": "${MY_RELAY_KEY}",
        "models": [
          {
            "id": "claude-opus-4",
            "name": "Claude Opus 4 (via relay)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": {
              "input": 15,
              "output": 75,
              "cacheRead": 1.5,
              "cacheWrite": 18.75
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

**为什么需要 `api: "openai-completions"`？**
- 虽然模型是 Claude，但中转服务使用 OpenAI 协议
- OpenClaw 会用 OpenAI 的请求格式调用这个服务
- 服务内部再转换成 Anthropic 格式

### 示例 2：使用兼容 Anthropic 的服务

假设你有一个服务兼容 Anthropic API：

```json
{
  "models": {
    "providers": {
      "anthropic-compatible": {
        "baseUrl": "https://api.example.com",
        "api": "anthropic-messages",  // ← 使用 Anthropic 协议
        "apiKey": "${MY_KEY}",
        "models": [
          {
            "id": "my-model",
            "name": "My Model",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 1, "output": 3, "cacheRead": 0.1, "cacheWrite": 1.25 },
            "contextWindow": 100000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

### 示例 3：同一 Provider 混合 API

假设一个 provider 同时提供兼容 OpenAI 和 Anthropic 的模型：

```json
{
  "models": {
    "providers": {
      "mixed-provider": {
        "baseUrl": "https://api.example.com",
        "api": "openai-completions",  // Provider 默认
        "apiKey": "${MY_KEY}",
        "models": [
          {
            "id": "model-openai",
            "name": "OpenAI Compatible Model",
            // 使用 provider 默认: openai-completions
            ...
          },
          {
            "id": "model-anthropic",
            "name": "Anthropic Compatible Model",
            "api": "anthropic-messages",  // ← 覆盖 provider 默认
            ...
          }
        ]
      }
    }
  }
}
```

### 示例 4：覆盖内置 Provider 的价格

假设你有 Minimax 的企业折扣价：

```json
{
  "models": {
    "mode": "merge",  // 保留内置配置，只覆盖价格
    "providers": {
      "minimax": {
        "apiKey": "${MINIMAX_API_KEY}",
        "models": [
          {
            "id": "MiniMax-M2.5",
            "cost": {
              "input": 0.15,    // 原价 0.3，折扣后
              "output": 0.6,    // 原价 1.2，折扣后
              "cacheRead": 0.015,
              "cacheWrite": 0.06
            }
          }
        ]
      }
    }
  }
}
```

**结果**：
- `api`, `reasoning`, `input`, `contextWindow` 等从内置目录继承
- `cost` 使用你的自定义价格

### 示例 5：添加未在内置目录的模型

假设 Minimax 发布了新模型 `MiniMax-M3`，但 OpenClaw 内置目录还没更新：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "apiKey": "${MINIMAX_API_KEY}",
        "models": [
          {
            "id": "MiniMax-M3",  // 新模型
            "name": "MiniMax M3",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": {
              "input": 0.5,
              "output": 2.0,
              "cacheRead": 0.05,
              "cacheWrite": 0.2
            },
            "contextWindow": 300000,
            "maxTokens": 16384
          }
        ]
      }
    }
  }
}
```

**结果**：
- 内置的 `MiniMax-M2.5` 仍然可用
- 新增的 `MiniMax-M3` 也可用
- 两个模型都使用 `api: "anthropic-messages"`（从 provider 继承）

---

## 常见问题

### Q1: 如何知道某个 provider 应该用哪种 API？

**方法 1：查看 provider 文档**
- 如果文档说"兼容 OpenAI API"，使用 `openai-completions`
- 如果文档说"兼容 Anthropic API"，使用 `anthropic-messages`

**方法 2：查看 OpenClaw 内置配置**
```bash
# 查看内置 provider 列表
grep -r "buildProvider" src/agents/models-config.providers.ts

# 查看某个 provider 的 API 类型
grep -A 5 "buildMinimaxProvider" src/agents/models-config.providers.ts
```

**方法 3：试错**
- 先试 `openai-completions`（最常见）
- 如果报错，再试 `anthropic-messages`
- 查看日志中的错误信息

### Q2: 如果不指定 `api` 字段会怎样？

- **OpenRouter**：自动使用 `openai-completions`
- **其他 provider**：默认使用 `openai-responses`
- **可能导致**：API 调用失败，因为协议不匹配

**建议**：始终显式指定 `api` 字段。

### Q3: 如何验证 API 配置是否正确？

```bash
# 1. 运行 doctor 检查配置
openclaw doctor

# 2. 查看模型列表
openclaw models list

# 3. 发送测试消息
openclaw message send --message "test" --channel test

# 4. 查看日志
tail -f ~/.openclaw/logs/gateway.log | grep -i "api\|error"
```

### Q4: `merge` 模式下，如何完全禁用某个内置 provider？

**方法 1：不配置 apiKey**
```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        // 不配置 apiKey，模型不可用
      }
    }
  }
}
```

**方法 2：使用 `replace` 模式**
```json
{
  "models": {
    "mode": "replace",
    "providers": {
      // 只列出你想要的 provider
    }
  }
}
```

### Q5: 如何添加一个完全自定义的 provider？

```json
{
  "models": {
    "mode": "merge",  // 或 "replace"
    "providers": {
      "my-custom": {
        "baseUrl": "https://api.example.com/v1",
        "api": "openai-completions",  // 必须指定
        "apiKey": "${MY_CUSTOM_KEY}",
        "models": [
          {
            "id": "custom-model-1",
            "name": "Custom Model 1",
            "reasoning": false,
            "input": ["text"],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 8192,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

### Q6: Gemini 应该用哪种 API？

**官方 Gemini API**：
```json
{
  "google-gemini": {
    "baseUrl": "https://generativelanguage.googleapis.com",
    "api": "google-generative-ai",  // ← Gemini 专用 API
    "apiKey": "${GOOGLE_API_KEY}",
    "models": [...]
  }
}
```

**通过 OpenAI 兼容服务访问 Gemini**：
```json
{
  "gemini-via-openai": {
    "baseUrl": "https://api.example.com/v1",
    "api": "openai-completions",  // ← 使用 OpenAI 协议
    "apiKey": "${MY_KEY}",
    "models": [
      {
        "id": "gemini-pro",
        ...
      }
    ]
  }
}
```

### Q7: 如何处理模型 ID 冲突？

如果两个 provider 有相同的模型 ID（如 `gpt-4`）：

```json
{
  "models": {
    "providers": {
      "openai": {
        "models": [
          { "id": "gpt-4", ... }  // openai/gpt-4
        ]
      },
      "my-relay": {
        "models": [
          { "id": "gpt-4", ... }  // my-relay/gpt-4
        ]
      }
    }
  }
}
```

**引用时使用完整路径**：
```json
{
  "agents": {
    "defaults": {
      "model": "openai/gpt-4"  // 或 "my-relay/gpt-4"
    }
  }
}
```

### Q8: 动态发现失败会影响启动吗？

**不会**。OpenClaw 的发现机制是容错的：
1. 尝试动态发现
2. 如果失败，回退到静态目录
3. 记录警告日志
4. 继续启动

**查看发现日志**：
```bash
grep -i "discovery\|catalog" ~/.openclaw/logs/gateway.log
```

---

## 最佳实践

### 1. 始终显式指定 `api` 字段

```json
// ✅ 好
{
  "my-provider": {
    "api": "openai-completions",
    ...
  }
}

// ❌ 不好（依赖默认值）
{
  "my-provider": {
    // 没有 api 字段
    ...
  }
}
```

### 2. 使用 `merge` 模式（除非有特殊需求）

```json
{
  "models": {
    "mode": "merge"  // 推荐
  }
}
```

**优点**：
- 自动获取内置模型的更新
- 只需覆盖需要修改的部分
- 减少配置维护工作

### 3. 为自定义模型提供完整信息

```json
{
  "models": [
    {
      "id": "my-model",
      "name": "My Model",
      "reasoning": false,        // 明确指定
      "input": ["text"],         // 明确指定
      "cost": { ... },           // 提供准确的价格
      "contextWindow": 8192,     // 提供准确的限制
      "maxTokens": 4096          // 提供准确的限制
    }
  ]
}
```

### 4. 使用环境变量管理 API Key

```json
{
  "providers": {
    "my-provider": {
      "apiKey": "${MY_PROVIDER_KEY}"  // ✅ 好
      // "apiKey": "sk-xxx..."        // ❌ 不好（明文）
    }
  }
}
```

### 5. 定期运行 `openclaw doctor`

```bash
# 验证配置
openclaw doctor

# 查看模型列表
openclaw models list

# 测试模型调用
openclaw message send --message "test" --channel test
```

---

## 相关文档

- [01-models-and-providers.md](./01-models-and-providers.md) - 模型配置基础
- [02-agents-and-routing.md](./02-agents-and-routing.md) - Agent 配置
- [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md) - 诊断工具

---

## 总结

1. **API 协议必须显式配置**，OpenClaw 不会自动推断
2. **8 种 API 类型**，最常用的是 `openai-completions` 和 `anthropic-messages`
3. **`merge` 模式**智能合并内置目录和用户配置
4. **动态发现**是可选的，失败不影响启动
5. **模型定义无需验证**，OpenClaw 信任你的配置

**核心原则**：显式优于隐式，配置优于猜测。
