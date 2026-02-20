# OpenClaw 最佳实践指南

本文档整理了 OpenClaw 的配置、安全、性能和运维方面的最佳实践。

---

## 目录

1. [安全配置](#1-安全配置)
2. [多 Agent 配置](#2-多-agent-配置)
3. [内存管理](#3-内存管理)
4. [Skills 使用](#4-skills-使用)
5. [性能优化](#5-性能优化)
6. [运维监控](#6-运维监控)
7. [故障排查](#7-故障排查)

---

## 1. 安全配置

### 1.1 安全基线配置

**60 秒安全加固**：

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "使用长随机字符串" },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

### 1.2 访问控制原则

**核心原则：身份优先，范围其次，模型最后**

| 层级 | 控制点 | 配置 |
|------|--------|------|
| 身份 | 谁能发消息 | `dmPolicy: "pairing"` 或 `allowlist` |
| 范围 | 能做什么 | `tools.allow/deny`, `sandbox` |
| 模型 | 如何处理 | 选择经过指令加固的模型 |

### 1.3 安全审计

**定期运行**：

```bash
# 基础审计
openclaw security audit

# 深度审计（包含 Gateway 探测）
openclaw security audit --deep

# 自动修复
openclaw security audit --fix
```

### 1.4 敏感配置检查清单

| 检查项 | 严重性 | 修复方式 |
|--------|--------|----------|
| Gateway 无认证暴露 | Critical | 设置 `gateway.auth.token` |
| 公网暴露 (Tailscale Funnel) | Critical | 设置 `gateway.tailscale.mode: "off"` |
| DM 策略开放 + 工具可用 | Critical | 使用 `pairing` 或 `allowlist` |
| 配置文件权限过宽 | Critical | `chmod 600 ~/.openclaw/openclaw.json` |
| Control UI 不安全认证 | Critical | 使用 HTTPS 或 localhost |

### 1.5 凭证存储位置

```
~/.openclaw/
├── openclaw.json                    # 主配置（包含敏感信息）
├── credentials/
│   ├── whatsapp/<accountId>/        # WhatsApp 认证
│   ├── <channel>-allowFrom.json     # 配对白名单
│   └── oauth.json                   # OAuth 凭证（旧版）
└── agents/<agentId>/agent/
    └── auth-profiles.json           # 模型认证配置
```

**权限设置**：

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod -R 700 ~/.openclaw/credentials
```

---

## 2. 多 Agent 配置

### 2.1 Agent 概念

一个 **Agent** 是一个完全独立的大脑，拥有：

- **Workspace**：文件、AGENTS.md、SOUL.md、USER.md
- **AgentDir**：认证配置、模型注册、会话存储
- **Sessions**：聊天历史和路由状态

**关键点**：认证配置是 **per-agent** 的，默认不共享。

### 2.2 多 Agent 配置示例

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        workspace: "~/.openclaw/workspace",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "coding",
        workspace: "~/.openclaw/workspace-coding",
        agentDir: "~/.openclaw/agents/coding/agent",
        model: "anthropic/claude-opus-4-6",
        tools: {
          allow: ["exec", "read", "write", "edit"],
          deny: ["gateway", "cron"],
        },
      },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "coding", match: { channel: "telegram", accountId: "coding" } },
  ],
}
```

### 2.3 路由规则（优先级从高到低）

1. `peer` 精确匹配（DM/群组/频道 ID）
2. `parentPeer` 匹配（线程继承）
3. `guildId + roles`（Discord 角色路由）
4. `guildId`（Discord）
5. `teamId`（Slack）
6. `accountId` 匹配
7. `accountId: "*"`（频道级别）
8. 回退到默认 Agent

### 2.4 单号码多人使用

使用 `peer.kind: "direct"` 按发送者路由：

```json5
{
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
}
```

### 2.5 最佳实践

- **每个 Agent 使用独立的 `agentDir`**，避免会话冲突
- **共享凭证需要显式复制** `auth-profiles.json`
- **为群聊设置提及模式**，避免噪音
- **使用 `openclaw agents add <name>` 创建新 Agent**

---

## 3. 内存管理

### 3.1 内存文件结构

```
workspace/
├── MEMORY.md              # 长期记忆（仅主会话加载）
├── memory/
│   └── YYYY-MM-DD.md      # 每日日志
├── AGENTS.md              # Agent 行为规则
├── SOUL.md                # 人格设定
└── USER.md                # 用户信息
```

### 3.2 内存分层

| 文件 | 用途 | 加载时机 |
|------|------|----------|
| `MEMORY.md` | 精选长期记忆 | 仅主会话 |
| `memory/YYYY-MM-DD.md` | 每日日志 | 今天 + 昨天 |
| `AGENTS.md` | 行为规则 | 每次会话 |
| `SOUL.md` | 人格 | 每次会话 |

### 3.3 内存搜索配置

**向量搜索后端选择**：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",           // openai | gemini | voyage | local
        model: "text-embedding-3-small",
        extraPaths: ["../team-docs"], // 额外索引路径
      },
    },
  },
}
```

### 3.4 自动内存刷新（压缩前）

当会话接近自动压缩时，OpenClaw 会触发静默提示让模型写入持久化记忆：

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
        },
      },
    },
  },
}
```

### 3.5 最佳实践

- **重要决策写入 `MEMORY.md`**
- **日常记录写入 `memory/YYYY-MM-DD.md`**
- **告诉用户"请记录这个"来触发内存写入**
- **定期清理过期的每日日志**

---

## 4. Skills 使用

### 4.1 Skill 加载优先级

```
<workspace>/skills  (最高)
    ↓
~/.openclaw/skills  (共享)
    ↓
bundled skills  (内置)
```

### 4.2 安装 Skills

```bash
# 从 ClawHub 安装
clawhub install <skill-slug>

# 更新所有 skills
clawhub update --all

# 同步到 ClawHub
clawhub sync --all
```

### 4.3 Skill 配置

```json5
{
  skills: {
    entries: {
      "skill-name": {
        enabled: true,
        apiKey: "YOUR_API_KEY",
        env: {
          CUSTOM_VAR: "value",
        },
      },
    },
  },
}
```

### 4.4 Skill 格式（SKILL.md）

```markdown
---
name: my-skill
description: 简短描述
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["required-cli"], "env": ["API_KEY"] },
        "primaryEnv": "API_KEY",
      },
  }
---

## 使用说明

这里是具体的技能使用指南...
```

### 4.5 安全注意事项

- **第三方 skills 视为不可信代码**，使用前先阅读
- **敏感环境变量**通过 `skills.entries.*.env` 注入
- **沙箱运行**不可信输入：`sandbox.mode: "all"`

---

## 5. 性能优化

### 5.1 会话压缩配置

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard",
        reserveTokensFloor: 20000,
      },
    },
  },
}
```

### 5.2 并发控制

```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: {
        maxConcurrent: 8,
      },
    },
  },
}
```

### 5.3 图片尺寸优化

降低图片尺寸以减少 token 消耗：

```json5
{
  agents: {
    defaults: {
      imageMaxDimensionPx: 800,  // 默认 1200
    },
  },
}
```

### 5.4 Skills Token 开销

每个启用的 skill 约 97 字符（~24 tokens）+ 名称和描述长度。

**计算公式**：
```
total_tokens ≈ (195 + Σ(97 + len(name) + len(desc))) / 4
```

### 5.5 模型选择建议

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 日常对话 | Claude Sonnet / GPT-4o-mini | 速度快，成本低 |
| 深度推理 | Claude Opus / GPT-4 | 推理能力强 |
| 代码生成 | Claude Sonnet | 代码能力好 |
| 多语言 | GLM-5 | 中文优化 |

---

## 6. 运维监控

### 6.1 健康检查

```bash
# Gateway 状态
openclaw gateway status

# 通道状态
openclaw channels status --probe

# 诊断问题
openclaw doctor

# 自动修复
openclaw doctor --fix
```

### 6.2 日志配置

```json5
{
  logging: {
    level: "info",
    redactSensitive: true,
  },
}
```

### 6.3 Heartbeat 配置

定期检查任务：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last",  // 发送到最后一个活跃的通道
      },
    },
  },
}
```

### 6.4 Cron 任务

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 2,
    sessionRetention: "24h",
  },
}
```

### 6.5 配置热重载

```json5
{
  gateway: {
    reload: {
      mode: "hybrid",  // hot | hybrid | restart | off
      debounceMs: 300,
    },
  },
}
```

**热重载 vs 重启**：

| 配置类别 | 是否热重载 |
|----------|------------|
| channels, agents, models | ✅ 是 |
| hooks, cron, heartbeat | ✅ 是 |
| tools, skills, sessions | ✅ 是 |
| gateway.port/auth/bind | ❌ 需重启 |

---

## 7. 故障排查

### 7.1 常用诊断命令

```bash
# 查看日志
openclaw logs --tail 100

# 检查配置
openclaw doctor

# 验证配置格式
python3 -m json.tool ~/.openclaw/openclaw.json

# 检查 Gateway 状态
openclaw gateway status

# 检查通道
openclaw channels list
openclaw channels status --probe
```

### 7.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Gateway 无法启动 | 配置格式错误 | 运行 `openclaw doctor` |
| 飞书/Telegram 无法连接 | Token 无效 | 检查 credentials 目录 |
| 消息无响应 | DM 策略限制 | 检查 `dmPolicy` 和 `allowFrom` |
| 工具执行失败 | 权限限制 | 检查 `tools.allow/deny` |
| 内存不生效 | 文件不存在 | 创建 `MEMORY.md` |

### 7.3 重置步骤

```bash
# 重置 Gateway
openclaw gateway restart

# 完全重置（危险！）
openclaw reset --all

# 重新配置
openclaw onboard
```

---

## 附录：配置文件模板

### 最小配置

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { dmPolicy: "pairing" } },
}
```

### 多 Agent 配置

```json5
{
  agents: {
    list: [
      { id: "main", default: true, workspace: "~/.openclaw/workspace" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "whatsapp" } },
    { agentId: "work", match: { channel: "telegram" } },
  ],
}
```

### 安全加固配置

```json5
{
  gateway: {
    bind: "loopback",
    auth: { mode: "token", token: "长随机字符串" },
  },
  session: { dmScope: "per-channel-peer" },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "gateway", "cron"],
    exec: { security: "deny" },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing" },
    telegram: { dmPolicy: "allowlist", allowFrom: ["tg:123456"] },
  },
}
```

---

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent)
- [Security](https://docs.openclaw.ai/gateway/security)
- [Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [ClawHub Skills 市场](https://clawhub.com)
