# OpenClaw 实战：如何“克隆”一个精通业务的 Agent？

> **核心问题**：当我经过长时间的调教、Review 和报错反馈，终于在本地“养”出了一个极其好用的 Agent（不仅懂我的代码，还懂我的脾气和服务器环境）。如果新人入职，我**能不能只通过拷贝这个项目（Workspace）的代码目录，就把这个“满级”的 Agent 完美过继给他？**

**简答版答案**：
**不能完全做到。**
拷贝 Workspace 只能继承 Agent 的“显性知识”和“项目级业务逻辑”。它会丢失一些底层的“潜意识直觉（长期记忆库）”、“全局通用技能”以及“高危执行黑卡（运行时权限）”。

要实现 100% 的完美克隆，必须搞清楚 OpenClaw 里 Agent 的状态是分布在哪些不同维度的。

---

## 一、 如果只拷走 Workspace，你能继承什么？（显性资产）

由于 OpenClaw 遵循“Everything is Markdown”，Workspace 内的文件包揽了该 Agent 面向此项目的绝大部分显性业务逻辑。这部分通过 Git 就可以直接共享给新员工。

新员工克隆项目后，他的 Agent 立即懂得：

1. **项目灵魂与人设 (`SOUL.md` / `IDENTITY.md`)**：知道自己在该项目里是个严谨的架构师，还是个负责写测试的开发。
2. **多 Agent 协作网络 (`AGENTS.md`)**：知道遇到报错该派生（Spawn）哪个特定配置的子代去查。
3. **明文的避坑指南 (`MEMORY.md`)**：只要你在之前的会话里让 Agent 把关键总结沉淀进了显性的 `MEMORY.md` 文本里，新员工的 Agent 在加载时就会立刻知道“这个项目必须用 Pnpm，严禁碰 `sys` 库”。
4. **用户交互底线 (`USER.md`)**：懂得交流时不用繁体字，或者代码只输出 diff 而非全量（如果这些偏好被固化在 `USER.md` 的话）。
5. **项目级专属技能 (`[项目根目录]/.agents/skills/` 或 `skills/`)**：遇到特定的构建发布等动作，Agent 能直接调用项目里内聚存放的 XML/Markdown “技能卡带”执行。

---

## 二、 只拷 Workspace 会丢失什么？（隐性资产与环境特权）

这些是“养”出来的手感，它们与**本机环境、运行时引擎、长期存储**强绑定。一旦换了电脑（甚至仅仅是切了 OpenClaw 全局配置），新 Agent 就会显得“变笨了”。

### 1. 丢失了“潜意识”：本地 SQLite 向量记忆引擎
OpenClaw 默认开启了 `memorySearch.enabled: true`。一旦开启，它会将项目中的文档和长对话提取为区块，并在后台进行向量化，存于本地系统的全局缓存路径中（通常是 `~/.cache/qmd/index.sqlite` 或 `$XDG_CACHE_HOME/qmd/index.sqlite`）。

> **为什么会被丢失？** 虽然 Workspace 的知识是项目维度的，但**底层的向量索引数据库是存放在宿主机的全局/临时目录中的**。

*   **条件化使用**：向量库并非绝对绑定，如果你在全局配置中设定了 `agents.defaults.memorySearch.enabled: false`，则 Agent 完全退化为只读 Markdown 的纯文本模式，此时这部分隐性资产压根不会产生。
*   **Session 与时间记忆**：如果你在配置中开启了实验性的 `experimental: { sessionMemory: true }`，Agent 过去的历史会话（Session Transcripts）也会被灌入这个 SQLite 库中。并且在检索时会叠加时间衰减算法（Temporal Decay），让越新的记忆权重越高。**这部分“时间感知”也是完全强依赖这个处于本地机器缓存里的 `.sqlite` 文件的。**
*   **克隆后的表现**：新员工 `git clone` 到的新仓库里无法带走上一个主机的 `.sqlite` 缓存。当 Agent 面对曾出现过的诡异报错时，它无法通过 `memory_search` 发生语义级“条件反射”直接避险，因为它失去了那颗被时间打磨过的记忆大脑。

### 2. 丢失了“通用内功”：全局与个人技能库
技能加载拥有严格层级（`bundled < managed < personal < project`）。如果你养它的过程中，曾经在全局装了技能（比如存在 `~/.openclaw/skills/` 或 `~/.agents/skills/` 中，为了能跨多个项目通用）：
*   **表现**：新项目的 Agent 突然不知道怎么去连 AWS 服务器了。因为虽然 Workspace 里写了它该干什么，但提供肢体能力的全局对应 `SKILL.md` 没跟着一起拷过来。

### 3. 丢失了“物理黑卡”：Host 容器与配置提权
你的本机之所以好用，是因为随着信任的累加，你在 `~/.openclaw/config.yaml` 或 Gateway 配置里给它开了“绿灯”（比如 `exec: { security: "full", ask: "off" }`）。
*   **表现**：一到新员工的电脑上，原本能连跑 10 条编译指令自动纠错的神级 Agent，现在跑一条 `npm run build` 都在弹窗问人类：“这个修改有风险，请问是否放行？” 或者因为环境退回了 Strict Sandbox，导致它无权去读系统的某个全局配置文件证书。

---

## 三、 终极实战：如何 100% 完美克隆一个 “满级” Agent？

如果你要在团队内推行一个极其强势、顺手的“模板级 Agent Node”，正确的打法是采取 **“代码库共享 + 运行时环境对齐”** 的三步走策略：

### 第一步：最大化显性沉淀（榨干它的价值）
在准备克隆之前，强迫 Agent 将隐性知识转为显性。
在原电脑上对它说：*“请把你最近一个月在 SQLite 记忆库里关于 `[某些特定领域]` 的避坑知识进行总结，并统统写入本项目的 `MEMORY.md` 中；把你常用的解决构建报错的长串 Bash 逻辑，抽离成一个具体的 `.agents/skills/my-build-recover/SKILL.md` 保存到项目里。”*

### 第二步：推送 Workspace（共享显性资产）
使用 Git 提交代码，新开发机器 `git clone`，拿到核心灵魂（各种业务 `*.md` 与项目级 Skills）。

### 第三步：对齐基础设施与授权（使用 Doctor 自动愈合）
在新环境下配置：
1. **统一运行时配置**：确保新员工的 `~/.openclaw/config.yaml` 里，给予了 Agent 同等维度的物理读写权限（开启 `full` security 和合理的无感 `ask` 策略）。
2. **全局依赖拉取**：确保新员工机子上也安装了同样的全局 Skills。
3. **环境自愈与迁移 (`openclaw doctor`)**：跨机拷贝后，绝对路径、授权令牌往往会错乱。**不要手工一个个改**！直接在终端运行：
   ```bash
   # 交互式自动修复环境错乱、路径错配及旧版状态表
   openclaw doctor --fix
   # (或者对于无人值守的服务器，使用 --non-interactive)
   ```
   OpenClaw 内置的 `doctor` 会帮你撸平底层因机器变更带来的物理水土不服。

### 第四步：强制重构向量脑（利用原生指令）
相比于拷贝脆弱且易锁死的 SQLite 文件，我们强烈推荐利用刚刚提纯好的 `MEMORY.md` 结合官方指令，让新机器上的它**主动“逆炼”**出新大脑。
不要去“等”后台慢慢扫描发酵，直接运行：
```bash
# 1. 强制 Agent 立即读取迁移过来的所有记忆 Markdown 并调用 Embedding 模型生成最新的 SQLite 向量图谱
openclaw memory index --force

# 2. 验明正身
openclaw memory status
```
*当看到输出显示 `Index error: false` 且包含 `Indexed X chunks` 时，你的克隆体就真正找回了上一代的所有直觉和“潜意识”。*

---

**总结**：OpenClaw 的 Workspace 设计非常克制，它极力将业务逻辑封装在 Markdown 中实现高度可移植。但这并不意味着 `workspace = Agent 全部`。真正的“满阶智能体”，是 **“高内聚的 Markdown 知识树 + 本机深度的向量记忆池 + 极高信任等级的运行时特权”** 三位一体的产物。
