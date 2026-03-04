# 配置诊断与故障排查

本文档介绍如何使用 OpenClaw 内置的诊断工具来验证配置、排查问题和修复常见故障。

---

## 目录

1. [核心诊断命令](#核心诊断命令)
2. [openclaw doctor - 全面健康检查](#openclaw-doctor---全面健康检查)
3. [Gateway 诊断](#gateway-诊断)
4. [渠道诊断](#渠道诊断)
5. [安全审计](#安全审计)
6. [Memory 系统诊断](#memory-系统诊断)
7. [综合状态查看](#综合状态查看)
8. [常见问题排查流程](#常见问题排查流程)
9. [日志查看与分析](#日志查看与分析)

---

## 核心诊断命令

OpenClaw 提供了一套完整的诊断工具，**强烈建议使用这些内置命令而不是手动检查配置文件**：

| 命令 | 用途 | 适用场景 |
|------|------|----------|
| `openclaw doctor` | 全面健康检查 | 配置后验证、定期检查、问题排查 |
| `openclaw doctor --fix` | 自动修复问题 | 发现问题后快速修复 |
| `openclaw gateway status` | Gateway 状态 | Gateway 连接问题 |
| `openclaw channels status` | 渠道状态 | 消息收发问题 |
| `openclaw security audit` | 安全审计 | 安全配置检查 |
| `openclaw memory status` | Memory 系统状态 | 记忆功能问题 |
| `openclaw status` | 综合状态 | 快速了解整体状态 |

---

## openclaw doctor - 全面健康检查

### 基本用法

```bash
# 运行完整健康检查
openclaw doctor

# 自动修复发现的问题
openclaw doctor --fix

# 强制修复（覆盖自定义配置）
openclaw doctor --fix --force

# 非交互模式（适合脚本）
openclaw doctor --non-interactive
```

### 检查项目

`openclaw doctor` 会检查以下方面：

#### 1. 配置与状态
- ✅ 配置文件格式验证
- ✅ 配置版本迁移
- ✅ 状态目录完整性
- ✅ Session 锁健康状态
- ✅ 工作区备份建议
- ✅ Memory 系统配置建议

#### 2. Gateway 与服务
- ✅ Gateway 模式配置（local/remote）
- ✅ Gateway 认证设置（token/password）
- ✅ Gateway 健康探测和可达性
- ✅ Gateway Memory 状态
- ✅ Gateway 守护进程状态（systemd/launchd/Windows Task Scheduler）
- ✅ 额外 Gateway 服务检测
- ✅ macOS LaunchAgent 环境变量覆盖

#### 3. 安全与认证
- ✅ 认证配置健康（Anthropic OAuth、废弃的 CLI profiles）
- ✅ 安全警告
- ✅ 废弃的环境变量

#### 4. 渠道与集成
- ✅ Sandbox 镜像验证
- ✅ Sandbox 作用域警告
- ✅ Shell 补全设置

#### 5. 模型与 Hooks
- ✅ Hooks Gmail 模型验证
- ✅ 模型目录检查

#### 6. 平台特定
- ✅ systemd user linger（Linux）
- ✅ macOS launch agent 问题
- ✅ 源码安装问题

### 自动修复功能

使用 `--fix` 标志时，doctor 会自动修复以下问题：

- 🔧 生成并配置缺失的 Gateway token
- 🔧 自动迁移旧版状态
- 🔧 修复 Anthropic OAuth profile ID
- 🔧 移除废弃的 CLI 认证配置
- 🔧 修复 Sandbox Docker 镜像
- 🔧 修复 Gateway 服务配置
- 🔧 修复 UI 协议新鲜度
- 🔧 迁移配置兼容性问题
- 🔧 移除过期的 session 锁
- 🔧 应用安全的配置迁移

**注意**：修复操作会自动备份配置文件（`.bak` 后缀）。

### 输出示例

```bash
$ openclaw doctor

◇  OpenClaw Doctor
│
◆  Configuration & State
│  ✔ Config file valid
│  ✔ State directory healthy
│  ⚠ Found 2 stale session locks (older than 7 days)
│
◆  Gateway & Services
│  ✔ Gateway running (PID 86909, port 18789)
│  ✔ Gateway reachable
│  ✔ Gateway memory enabled
│  ⚠ macOS LaunchAgent has custom PATH override
│
◆  Security & Auth
│  ✔ Gateway token configured
│  ✔ No deprecated environment variables
│
◆  Channels & Integrations
│  ✔ All channels healthy
│
◆  Summary
│  Found 2 warnings, 0 errors
│  Run `openclaw doctor --fix` to apply automatic repairs
│
└  Health check complete
```

### 最佳实践

1. **定期运行**：建议每周运行一次 `openclaw doctor`
2. **配置后验证**：每次修改配置后运行 `openclaw doctor`
3. **问题排查第一步**：遇到任何问题时，首先运行 `openclaw doctor`
4. **自动修复前备份**：虽然 doctor 会自动备份，但重要配置建议手动备份
5. **查看修复内容**：使用 `--fix` 后，检查 `.bak` 文件了解修改内容

---

## Gateway 诊断

### Gateway 状态检查

```bash
# 基本状态
openclaw gateway status

# 包含可达性探测
openclaw gateway probe

# 获取健康快照
openclaw gateway health

# 发现本地/广域 Gateway
openclaw gateway discover
```

### 输出示例

```bash
$ openclaw gateway status

Gateway Status
├─ Mode: local
├─ Address: http://127.0.0.1:18789
├─ Status: running (PID 86909)
├─ Uptime: 2h 15m
└─ Health: ✔ healthy

$ openclaw gateway probe

Gateway Probe
├─ Reachability: ✔ reachable
├─ Response time: 45ms
├─ Version: 2026.2.17
├─ Memory: enabled
└─ Channels: 3 active
```

### 常见 Gateway 问题

#### 问题 1：Gateway 无法连接

```bash
# 检查 Gateway 是否运行
openclaw gateway status

# 如果未运行，启动 Gateway（macOS）
# 通过 OpenClaw Mac 应用启动
# 或使用脚本
./scripts/restart-mac.sh

# 验证端口监听
ss -ltnp | grep 18789  # Linux
lsof -i :18789         # macOS
```

#### 问题 2：Gateway token 错误

```bash
# 使用 doctor 自动修复
openclaw doctor --fix

# 或手动重新生成 token
openclaw gateway token --regenerate
```

#### 问题 3：Gateway 模式配置错误

```bash
# 检查当前模式
openclaw config get gateway.mode

# 设置为本地模式
openclaw config set gateway.mode local

# 设置为远程模式
openclaw config set gateway.mode remote
openclaw config set gateway.url https://your-gateway.example.com
```

---

## 渠道诊断

### 渠道状态检查

```bash
# 查看所有渠道状态
openclaw channels status

# 运行渠道凭证探测
openclaw channels status --probe

# 查看渠道能力
openclaw channels capabilities

# 列出配置的渠道
openclaw channels list
```

### 输出示例

```bash
$ openclaw channels status

Channels Status
├─ Discord: ✔ connected (bot mode)
├─ Telegram: ✔ connected
├─ Feishu: ✔ connected
├─ Signal: ⚠ not configured
└─ WhatsApp: ⚠ not configured

$ openclaw channels status --probe

Channels Probe
├─ Discord
│  ├─ Auth: ✔ valid token
│  ├─ API: ✔ reachable
│  └─ Bot user: @openclaw#1234
├─ Telegram
│  ├─ Auth: ✔ valid token
│  ├─ API: ✔ reachable
│  └─ Bot user: @openclaw_bot
└─ Feishu
   ├─ Auth: ✔ valid credentials
   ├─ API: ✔ reachable
   └─ App ID: cli_a1b2c3d4e5f6
```

### 常见渠道问题

#### 问题 1：消息无法发送

```bash
# 1. 检查渠道状态
openclaw channels status --probe

# 2. 检查 Gateway 连接
openclaw gateway status

# 3. 查看最近的错误日志
tail -n 100 ~/.openclaw/logs/gateway.log | grep ERROR

# 4. 运行完整诊断
openclaw doctor
```

#### 问题 2：渠道认证失败

```bash
# 检查环境变量
env | grep -E 'DISCORD|TELEGRAM|FEISHU'

# 重新配置渠道（以 Discord 为例）
openclaw config set discord.token "your-bot-token"

# 验证配置
openclaw channels status --probe
```

#### 问题 3：Allowlist 配置问题

```bash
# 检查当前 allowlist
openclaw config get discord.allowlist
openclaw config get telegram.allowlist
openclaw config get feishu.allowlist

# 添加用户到 allowlist（以飞书为例）
openclaw config set feishu.allowlist '["ou_fe4581b657a9f031701969d9631541d4"]'

# 验证配置
openclaw doctor
```

---

## 安全审计

### 安全审计命令

```bash
# 基本安全审计
openclaw security audit

# 深度审计（包含 Gateway 探测）
openclaw security audit --deep

# 自动修复安全问题
openclaw security audit --fix

# JSON 输出（适合自动化）
openclaw security audit --json
```

### 审计项目

安全审计会检查以下方面：

1. **文件权限**
   - 状态目录权限
   - 配置文件权限
   - 凭证文件权限

2. **Gateway 认证**
   - Token 配置
   - 密码强度
   - 认证方式

3. **危险配置标志**
   - 不安全的选项
   - 调试模式
   - 开发模式标志

4. **配置中的密钥**
   - 明文密码
   - API token
   - 敏感信息

5. **Sandbox 安全**
   - Docker 配置
   - 容器隔离
   - 资源限制

6. **Plugin/Skill 代码安全**
   - 代码签名
   - 权限检查
   - 依赖安全

7. **渠道安全**
   - 认证配置
   - Allowlist 设置
   - 权限范围

8. **模型卫生**
   - 模型来源
   - API 密钥管理
   - 配额设置

9. **攻击面分析**
   - 暴露的端口
   - 网络绑定
   - 外部访问

### 输出示例

```bash
$ openclaw security audit --deep

Security Audit
├─ File Permissions
│  ✔ State directory: 700 (secure)
│  ✔ Config file: 600 (secure)
│  ✔ Credentials: 600 (secure)
│
├─ Gateway Auth
│  ✔ Token configured
│  ✔ Binding: loopback only
│  ⚠ Consider enabling password auth for additional security
│
├─ Configuration
│  ✔ No dangerous flags
│  ✔ No secrets in config
│
├─ Sandbox
│  ✔ Docker isolation enabled
│  ✔ Resource limits configured
│
├─ Channels
│  ✔ All channels use allowlist
│  ⚠ Feishu allowlist has 1 entry (consider reviewing)
│
└─ Summary
   Found 2 recommendations, 0 critical issues
   Run `openclaw security audit --fix` to apply safe remediations
```

### 安全最佳实践

1. **定期审计**：每月运行一次 `openclaw security audit --deep`
2. **最小权限原则**：只授予必要的渠道访问权限
3. **使用 Allowlist**：所有生产渠道都应配置 allowlist
4. **保护凭证**：
   - 使用环境变量存储敏感信息
   - 不要在配置文件中明文存储密码
   - 定期轮换 API token
5. **Gateway 绑定**：
   - 本地使用时绑定到 loopback（127.0.0.1）
   - 远程访问时使用 HTTPS + 强密码
6. **文件权限**：
   - `~/.openclaw/` 目录：700
   - `openclaw.json`：600
   - 凭证文件：600

---

## Memory 系统诊断

### Memory 状态检查

```bash
# 基本状态
openclaw memory status

# 深度探测（包含向量和嵌入）
openclaw memory status --deep

# 重建索引
openclaw memory status --index

# JSON 输出
openclaw memory status --json
```

### 输出示例

```bash
$ openclaw memory status --deep

Memory Status
├─ Provider: anthropic
├─ Model: claude-opus-4-6
├─ Indexed Files: 1,234
├─ Indexed Chunks: 5,678
├─ Dirty State: no
├─ Store Path: ~/.openclaw/memory/
├─ Workspace Path: ~/projects/
│
├─ Sources
│  ├─ Memory files: 1,234
│  └─ Session logs: 456
│
├─ Embeddings
│  ✔ Available
│  ├─ Provider: anthropic
│  └─ Model: voyage-2
│
└─ Vector Store
   ✔ Healthy
   ├─ Dimensions: 1024
   └─ Index size: 45.6 MB
```

### 常见 Memory 问题

#### 问题 1：Memory 搜索不工作

```bash
# 1. 检查 Memory 状态
openclaw memory status --deep

# 2. 检查 Gateway Memory 配置
openclaw config get gateway.memory.enabled

# 3. 启用 Memory
openclaw config set gateway.memory.enabled true

# 4. 重建索引
openclaw memory status --index

# 5. 验证
openclaw doctor
```

#### 问题 2：索引过期或损坏

```bash
# 重建索引
openclaw memory status --index

# 清除并重建
rm -rf ~/.openclaw/memory/index/
openclaw memory status --index
```

#### 问题 3：嵌入模型配置错误

```bash
# 检查当前配置
openclaw config get gateway.memory.embeddings

# 配置嵌入模型
openclaw config set gateway.memory.embeddings.provider anthropic
openclaw config set gateway.memory.embeddings.model voyage-2

# 验证
openclaw memory status --deep
```

---

## 综合状态查看

### 状态命令

```bash
# 快速状态概览
openclaw status

# 深度状态（包含健康探测）
openclaw status --deep

# 完整详细状态（表格格式）
openclaw status --all

# 包含使用量/配额快照
openclaw status --usage

# JSON 输出
openclaw status --json
```

### 输出示例

```bash
$ openclaw status --deep

OpenClaw Status
├─ Version: 2026.2.17
├─ Config: ~/.openclaw/openclaw.json
├─ State: ~/.openclaw/
│
├─ Gateway
│  ✔ Running (local mode)
│  ├─ Address: http://127.0.0.1:18789
│  ├─ PID: 86909
│  ├─ Uptime: 2h 15m
│  └─ Health: ✔ healthy
│
├─ Channels (3 active)
│  ✔ Discord: connected
│  ✔ Telegram: connected
│  ✔ Feishu: connected
│
├─ Agents (2 configured)
│  ✔ main (default)
│  ✔ xiaolaoer
│
├─ Models (3 providers)
│  ✔ anthropic: 2 models
│  ✔ openai: 1 model
│  ✔ deepseek: 1 model
│
├─ Memory
│  ✔ Enabled
│  ├─ Indexed: 1,234 files
│  └─ Store: 45.6 MB
│
└─ Security
   ✔ No critical issues
   └─ Last audit: 2h ago
```

---

## 常见问题排查流程

### 流程图

```
遇到问题
    ↓
运行 openclaw doctor
    ↓
发现问题？
    ├─ 是 → 运行 openclaw doctor --fix
    │         ↓
    │      问题解决？
    │         ├─ 是 → 完成 ✓
    │         └─ 否 → 继续下一步
    │
    └─ 否 → 根据问题类型选择专项诊断
              ↓
         ┌────┴────┬────────┬────────┐
         ↓         ↓        ↓        ↓
    Gateway    Channels  Security  Memory
    诊断       诊断      审计      诊断
         ↓         ↓        ↓        ↓
         └────┬────┴────────┴────────┘
              ↓
         查看日志分析
              ↓
         问题解决？
              ├─ 是 → 完成 ✓
              └─ 否 → 寻求社区帮助
```

### 具体场景

#### 场景 1：Agent 无响应

```bash
# 1. 运行完整诊断
openclaw doctor

# 2. 检查 Gateway 状态
openclaw gateway status

# 3. 检查渠道状态
openclaw channels status --probe

# 4. 查看最近日志
tail -n 200 ~/.openclaw/logs/gateway.log

# 5. 检查 Agent 配置
openclaw config get agents.list
openclaw config get bindings

# 6. 验证路由规则
# 确保消息能路由到正确的 Agent
```

#### 场景 2：模型调用失败

```bash
# 1. 检查模型配置
openclaw config get models.providers

# 2. 验证 API 密钥
env | grep -E 'ANTHROPIC|OPENAI|DEEPSEEK'

# 3. 测试模型连接
# （通过发送测试消息）

# 4. 查看错误日志
grep -i "model\|api\|error" ~/.openclaw/logs/gateway.log | tail -n 50

# 5. 运行安全审计
openclaw security audit --deep
```

#### 场景 3：配置修改后不生效

```bash
# 1. 验证配置语法
openclaw doctor

# 2. 重启 Gateway（macOS）
./scripts/restart-mac.sh

# 3. 验证配置已加载
openclaw config get <your-config-key>

# 4. 检查 Gateway 日志
tail -n 100 ~/.openclaw/logs/gateway.log

# 5. 确认配置文件位置
openclaw config path
```

#### 场景 4：子 Agent 无法创建

```bash
# 1. 检查 Agent 配置
# 子 Agent 系统由 maxSpawnDepth 控制（≥1 即启用）
openclaw config get agents.defaults.subagents.maxSpawnDepth
openclaw config get agents.defaults.subagents.maxConcurrent

# 2. 检查 Gateway Memory
openclaw memory status --deep

# 3. 查看 Agent 状态目录
ls -la ~/.openclaw/agents/

# 4. 检查日志中的 spawn 错误
grep -i "spawn\|subagent" ~/.openclaw/logs/gateway.log | tail -n 50

# 5. 运行完整诊断
openclaw doctor --fix
```

---

## 日志查看与分析

### 日志位置

```bash
# Gateway 日志
~/.openclaw/logs/gateway.log

# 当天日志
/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# macOS 系统日志（需要 sudo）
./scripts/clawlog.sh
./scripts/clawlog.sh --follow
./scripts/clawlog.sh --tail 100
```

### 日志分析技巧

#### 1. 查看最近错误

```bash
# 最近 100 行错误
grep -i error ~/.openclaw/logs/gateway.log | tail -n 100

# 最近 50 行警告和错误
grep -iE "error|warn" ~/.openclaw/logs/gateway.log | tail -n 50

# 特定时间段的错误（最近 1 小时）
find ~/.openclaw/logs/ -name "*.log" -mmin -60 -exec grep -i error {} +
```

#### 2. 按组件过滤

```bash
# Gateway 相关
grep -i gateway ~/.openclaw/logs/gateway.log | tail -n 50

# 渠道相关
grep -iE "discord|telegram|feishu" ~/.openclaw/logs/gateway.log | tail -n 50

# 模型调用相关
grep -iE "model|anthropic|openai" ~/.openclaw/logs/gateway.log | tail -n 50

# Agent 相关
grep -iE "agent|spawn|session" ~/.openclaw/logs/gateway.log | tail -n 50
```

#### 3. 实时监控

```bash
# 实时查看所有日志
tail -f ~/.openclaw/logs/gateway.log

# 实时查看错误
tail -f ~/.openclaw/logs/gateway.log | grep -i error

# macOS 系统日志实时监控
./scripts/clawlog.sh --follow
```

#### 4. 日志级别

OpenClaw 日志通常包含以下级别：

- `ERROR`: 错误，需要立即处理
- `WARN`: 警告，可能影响功能
- `INFO`: 信息，正常运行日志
- `DEBUG`: 调试信息，详细执行流程

```bash
# 只看错误
grep "ERROR" ~/.openclaw/logs/gateway.log

# 错误和警告
grep -E "ERROR|WARN" ~/.openclaw/logs/gateway.log

# 调试信息（通常很多）
grep "DEBUG" ~/.openclaw/logs/gateway.log | tail -n 100
```

### macOS 系统日志（clawlog.sh）

```bash
# 基本用法
./scripts/clawlog.sh

# 实时跟踪
./scripts/clawlog.sh --follow

# 最近 N 行
./scripts/clawlog.sh --tail 200

# 按类别过滤
./scripts/clawlog.sh --category gateway
./scripts/clawlog.sh --category channels

# 组合使用
./scripts/clawlog.sh --follow --tail 50 | grep -i error
```

**注意**：`clawlog.sh` 需要 passwordless sudo 权限来访问 `/usr/bin/log`。

---

## 诊断工具速查表

| 问题类型 | 首选命令 | 备选命令 |
|---------|---------|---------|
| 不确定问题 | `openclaw doctor` | `openclaw status --deep` |
| Gateway 连接 | `openclaw gateway status` | `openclaw gateway probe` |
| 消息收发 | `openclaw channels status --probe` | `openclaw doctor` |
| 安全配置 | `openclaw security audit --deep` | `openclaw doctor` |
| Memory 功能 | `openclaw memory status --deep` | `openclaw doctor` |
| 配置验证 | `openclaw doctor` | `openclaw config get <key>` |
| 性能问题 | `openclaw status --usage` | 查看日志 |
| 日志分析 | `tail -f ~/.openclaw/logs/gateway.log` | `./scripts/clawlog.sh` |

---

## 最佳实践总结

1. **预防性诊断**
   - 每周运行 `openclaw doctor`
   - 每月运行 `openclaw security audit --deep`
   - 配置修改后立即运行 `openclaw doctor`

2. **问题排查顺序**
   - 先运行 `openclaw doctor`
   - 再运行专项诊断命令
   - 最后查看日志分析

3. **自动修复**
   - 优先使用 `--fix` 标志
   - 修复前检查 `.bak` 备份
   - 重要配置手动备份

4. **日志管理**
   - 定期清理旧日志
   - 保留最近 30 天的日志
   - 重要错误截图保存

5. **安全意识**
   - 不在日志中记录敏感信息
   - 分享日志前脱敏处理
   - 定期审计安全配置

6. **寻求帮助**
   - 提供完整的诊断输出
   - 附上相关日志片段
   - 说明复现步骤

---

## 相关文档

- [模型与 Provider 配置](./01-models-and-providers.md)
- [Agent 与路由配置](./02-agents-and-routing.md)
- [Agent 间通信与协作](./03-inter-agent-communication.md)
- [返回文档首页](./README.md)

---

**提示**：本文档持续更新中。如果遇到未覆盖的问题，请运行 `openclaw doctor` 并查看输出建议，或访问 [OpenClaw 社区](https://github.com/openclaw/openclaw) 寻求帮助。
