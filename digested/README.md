# OpenClaw 配置指南索引

本目录包含 OpenClaw 的详细配置指南，帮助你充分利用多模型、多 Agent 和 Agent 协作功能。

---

## 📚 文档列表

### [01-models-and-providers.md](./01-models-and-providers.md)
**模型和提供商配置**

学习如何配置：
- ✅ 多个模型提供商（官方 + 中转）
- ✅ 主要模型和备用模型（fallback）
- ✅ 不同场景的模型选择策略
- ✅ 环境变量管理
- ✅ 模型目录和别名

**适合场景**：
- 需要配置多个 API 提供商
- 需要设置备用模型防止服务中断
- 需要优化成本和性能

---

### [02-agents-and-routing.md](./02-agents-and-routing.md)
**Agent 配置和路由规则**

学习如何配置：
- ✅ 多个独立 Agent（不同 workspace）
- ✅ Agent 角色和身份定义
- ✅ 路由规则（按渠道、账号、对话类型）
- ✅ 会话范围控制
- ✅ 技能和工具配置

**适合场景**：
- 需要为不同用途创建专门的 Agent
- 需要在不同渠道使用不同 Agent
- 需要隔离工作和个人 Agent

---

### [03-inter-agent-communication.md](./03-inter-agent-communication.md)
**Agent 间通信和协作**

学习如何配置：
- ✅ 为不同 Agent 配置不同模型
- ✅ 子 Agent 系统（任务分解）
- ✅ Agent 间消息传递
- ✅ Orchestrator + Workers 模式
- ✅ 并发和资源控制

**适合场景**：
- 需要将复杂任务分解给多个 Agent
- 需要不同专业 Agent 协同工作
- 需要实现编排模式（orchestrator pattern）

---

### [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)
**配置诊断与故障排查**

学习如何使用：
- ✅ `openclaw doctor` - 全面健康检查
- ✅ Gateway 诊断工具
- ✅ 渠道状态检查
- ✅ 安全审计
- ✅ Memory 系统诊断
- ✅ 日志分析技巧

**适合场景**：
- 配置后验证是否正确
- 遇到问题需要排查
- 定期健康检查
- 安全配置审计

---

### [05-model-api-deep-dive.md](./05-model-api-deep-dive.md)
**模型 API 协议深度解析**（⭐ 进阶阅读）

深入理解：
- ✅ OpenAI vs Anthropic vs Gemini API 协议差异
- ✅ OpenClaw 如何选择 API 协议（显式配置，不自动推断）
- ✅ 模型目录系统（内置 + 动态发现 + 用户配置）
- ✅ `merge` vs `replace` 模式的工作机制
- ✅ 如何配置兼容 OpenAI/Anthropic 的中转服务

**适合场景**：
- 不确定某个 provider 应该用哪种 API
- 想理解模型目录的合并机制
- 需要配置自定义 provider
- 想添加未在内置目录的模型

---

## 🚀 快速开始

### 0. 配置前的准备：运行诊断

**在开始配置之前，强烈建议先运行诊断工具了解当前状态**：

```bash
# 全面健康检查
openclaw doctor

# 查看当前状态
openclaw status --deep

# 检查 Gateway 是否运行
openclaw gateway status

# 查看渠道状态
openclaw channels status
```

如果发现问题，可以使用自动修复：
```bash
openclaw doctor --fix
```

**详细诊断指南**：[04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)

---

### 1. 基础配置（单 Agent）

如果你刚开始使用 OpenClaw，建议先配置一个简单的 Agent：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.anthropic.com",
        "apiKey": "${ANTHROPIC_API_KEY}",
        "models": [...]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5"
    }
  }
}
```

**参考**：[01-models-and-providers.md](./01-models-and-providers.md) 第 3 节

---

### 2. 多 Agent 配置

当你需要隔离不同用途的 Agent 时：

```json
{
  "agents": {
    "list": [
      {
        "id": "personal",
        "default": true,
        "workspace": "~/.openclaw/workspace-personal"
      },
      {
        "id": "work",
        "workspace": "~/.openclaw/workspace-work"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "personal",
      "match": { "channel": "feishu", "accountId": "personal" }
    },
    {
      "agentId": "work",
      "match": { "channel": "feishu", "accountId": "work" }
    }
  ]
}
```

**参考**：[02-agents-and-routing.md](./02-agents-and-routing.md) 第 4 节

---

### 3. Agent 协作配置

当你需要 Agent 之间协作完成复杂任务时：

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxSpawnDepth": 2,
        "maxConcurrent": 8
      }
    },
    "list": [
      {
        "id": "orchestrator",
        "model": "anthropic/claude-opus-4-6",
        "subagents": {
          "allowAgents": ["worker-1", "worker-2"]
        }
      },
      {
        "id": "worker-1",
        "model": "anthropic/claude-sonnet-4-5"
      },
      {
        "id": "worker-2",
        "model": "anthropic/claude-sonnet-4-5"
      }
    ]
  }
}
```

**参考**：[03-inter-agent-communication.md](./03-inter-agent-communication.md) 第 5 节

---

## 📖 按场景查找

### 场景 1：我想使用多个 API 提供商

**问题**：
- 官方 API 有时会限流
- 想要备用方案
- 想要比较不同提供商的质量

**解决方案**：
1. 阅读 [01-models-and-providers.md](./01-models-and-providers.md) 第 3 节
2. 配置多个提供商（官方 + 中转）
3. 设置主要模型和备用模型（第 4 节）

---

### 场景 2：我想为工作和个人分别配置 Agent

**问题**：
- 工作和个人消息混在一起
- 想要不同的工作目录
- 想要不同的技能集

**解决方案**：
1. 阅读 [02-agents-and-routing.md](./02-agents-and-routing.md) 第 2-3 节
2. 创建两个 Agent（不同 workspace）
3. 配置路由规则（按账号或渠道）

---

### 场景 3：我想让 Agent 自动分解复杂任务

**问题**：
- 任务太复杂，单个 Agent 难以完成
- 想要并行处理多个子任务
- 想要不同专业的 Agent 协作

**解决方案**：
1. 阅读 [03-inter-agent-communication.md](./03-inter-agent-communication.md) 第 3 节
2. 配置子 Agent 系统
3. 设置 orchestrator + workers 模式（第 5 节）

---

### 场景 4：我想优化成本

**问题**：
- API 调用成本太高
- 简单任务不需要高级模型
- 想要灵活的备用策略

**解决方案**：
1. 阅读 [01-models-and-providers.md](./01-models-and-providers.md) 第 4.4 节
2. 配置不同质量/成本的模型
3. 为不同 Agent 配置不同模型（[03-inter-agent-communication.md](./03-inter-agent-communication.md) 第 2 节）

---

## 🔧 配置文件位置

- **主配置文件**：`~/.openclaw/openclaw.json`
- **环境变量**：`~/.profile` 或 `~/.zshrc`
- **Agent 状态目录**：`~/.openclaw/agents/<agent-id>/`
- **工作目录**：配置中的 `workspace` 路径
- **日志文件**：`~/.openclaw/logs/gateway.log`

---

## 🛠️ 常用命令

### 查看配置
```bash
# 查看完整配置
pnpm openclaw config get

# 查看模型配置
pnpm openclaw config get models

# 查看 Agent 配置
pnpm openclaw config get agents

# 查看路由规则
pnpm openclaw config get bindings
```

### 修改配置
```bash
# 设置配置项
pnpm openclaw config set agents.defaults.model "anthropic/claude-opus-4-6"

# 编辑配置文件
vim ~/.openclaw/openclaw.json
```

### 验证配置
```bash
# 运行 doctor 检查配置
openclaw doctor

# 修复配置问题
openclaw doctor --fix

# 查看 Gateway 状态
openclaw gateway status

# 查看渠道状态
openclaw channels status

# 查看综合状态
openclaw status --deep

# 安全审计
openclaw security audit --deep

# Memory 系统状态
openclaw memory status --deep
```

**详细诊断指南**：[04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)

### 测试
```bash
# 发送测试消息
pnpm openclaw message send --message "Hello" --channel test

# 查看可用模型
pnpm openclaw models list

# 查看会话列表
pnpm openclaw sessions list
```

---

## 📝 配置模板

### 最小配置
```json
{
  "models": {
    "mode": "merge"
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5"
    }
  }
}
```

### 推荐配置（多提供商 + 备用）
```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "anthropic": { /* 官方 */ },
      "openrouter": { /* 中转备用 */ }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5",
        "fallbacks": ["openrouter/anthropic/claude-sonnet-4-5"]
      }
    }
  }
}
```

### 完整配置（多 Agent + 协作）
```json
{
  "models": { /* 多提供商 */ },
  "agents": {
    "defaults": {
      "subagents": {
        "maxSpawnDepth": 2,
        "maxConcurrent": 8
      }
    },
    "list": [
      { "id": "main", "default": true },
      { "id": "orchestrator" },
      { "id": "worker-1" },
      { "id": "worker-2" }
    ]
  },
  "bindings": [ /* 路由规则 */ ]
}
```

---

## ❓ 常见问题

### Q: 配置修改后需要重启吗？

A: 是的，修改配置后需要重启 Gateway：
```bash
pnpm openclaw gateway restart
```

### Q: 如何备份配置？

A: 配置文件在 `~/.openclaw/openclaw.json`，直接复制即可：
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
```

### Q: 配置出错怎么办？

A: 运行 doctor 检查并修复：
```bash
pnpm openclaw doctor --fix
```

### Q: 如何查看日志？

A: 查看 Gateway 日志：
```bash
tail -f ~/.openclaw/logs/gateway.log
```

---

## 📚 更多资源

- **官方文档**：https://docs.openclaw.ai
- **GitHub**：https://github.com/openclaw/openclaw
- **配置示例**：查看各文档中的"完整配置示例"章节

---

## 🎯 下一步

1. **新手**：从 [01-models-and-providers.md](./01-models-and-providers.md) 开始配置模型
2. **进阶**：配置多 Agent，阅读 [02-agents-and-routing.md](./02-agents-and-routing.md)
3. **高级**：实现 Agent 协作，阅读 [03-inter-agent-communication.md](./03-inter-agent-communication.md)
4. **诊断**：遇到问题或需要验证配置，查看 [04-diagnostics-and-troubleshooting.md](./04-diagnostics-and-troubleshooting.md)
5. **深入理解**：想深入了解 API 协议机制，阅读 [05-model-api-deep-dive.md](./05-model-api-deep-dive.md)

---

**提示**：配置完成后，务必运行 `openclaw doctor` 验证配置是否正确！

---

**最后更新**：2026-03-02
**OpenClaw 版本**：2026.2.27
