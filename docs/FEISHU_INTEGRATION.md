# OpenClaw 飞书插件接入指南

本文档记录 OpenClaw 飞书插件的配置流程，支持单 Gateway 多 Agent 对接多个飞书机器人。

## 1. 环境要求

- OpenClaw 已安装
- Node.js 环境
- 飞书开放平台创建的机器人应用（已获取 App ID 和 App Secret）

## 2. 安装插件依赖

飞书插件需要安装依赖包，进入插件目录执行：

```bash
cd ~/.openclaw/extensions/feishu

# 移除 workspace 依赖（如果存在）
sed -i '/"openclaw": "workspace:\*"/d' package.json

# 安装依赖
npm install --legacy-peer-deps
```

依赖包包括：
- `@larksuiteoapi/node-sdk` - 飞书官方 SDK
- `@sinclair/typebox` - 类型校验
- `zod` - Schema 验证

## 3. 飞书应用配置

### 3.1 创建飞书应用

1. 登录 [飞书开放平台](https://open.feishu.cn/)
2. 创建企业自建应用
3. 记录 **App ID** 和 **App Secret**

### 3.2 配置应用权限

在飞书开放平台，为应用开启必要的权限：

**基础权限：**
- `im:message` - 获取与发送单聊、群组消息
- `im:message:receive_as_bot` - 接收群聊中@机器人消息

**可选权限（用于文档/网盘等工具）：**
- `docs:doc` - 文档操作
- `wiki:wiki` - 知识库操作
- `drive:drive` - 云空间操作

### 3.3 发布应用

配置完成后，发布应用版本并添加到企业可用应用列表。

## 4. OpenClaw 配置

### 4.1 单账号配置（基础模式）

编辑 `~/.openclaw/openclaw.json`：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxxxxxxxxxxxxx",
      "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "encryptKey": "xxx",
      "verificationToken": "xxx",
      "connectionMode": "websocket",
      "dmPolicy": "pairing"
    }
  }
}
```

### 4.2 多账号配置（推荐）

支持在单个 Gateway 中对接多个飞书机器人：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "connectionMode": "websocket",
      "accounts": {
        "openclaw": {
          "name": "openclaw",
          "appId": "cli_a91ed1b15278dccb",
          "appSecret": "RQazpCDOwMbsrck1tzTwTdLkLQ8jn1Ak"
        },
        "opengl": {
          "name": "opengl",
          "appId": "cli_a91d080d93b85bd6",
          "appSecret": "e8DarTtxDZoprbHALOI35dQNP070ad8Q"
        }
      }
    }
  }
}
```

### 4.3 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | boolean | 否 | 是否启用，默认 true |
| `connectionMode` | string | 否 | 连接模式：`websocket`（推荐）或 `webhook` |
| `domain` | string | 否 | 域名：`feishu` 或 `lark`，默认 `feishu` |
| `dmPolicy` | string | 否 | 私聊策略：`open`/`pairing`/`allowlist`，默认 `pairing` |
| `groupPolicy` | string | 否 | 群聊策略：`open`/`allowlist`/`disabled`，默认 `allowlist` |
| `requireMention` | boolean | 否 | 群聊是否需要@机器人，默认 true |

**账号级配置（accounts 内）：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 否 | 显示名称 |
| `appId` | string | 是 | 飞书应用 App ID |
| `appSecret` | string | 是 | 飞书应用 App Secret |
| `encryptKey` | string | 否 | 加密 Key |
| `verificationToken` | string | 否 | 验证 Token |

## 5. 连接模式选择

### 5.1 WebSocket 模式（推荐）

- 无需公网 IP
- 无需配置回调地址
- 自动保持长连接
- 适合内网环境

```json
{
  "connectionMode": "websocket"
}
```

### 5.2 Webhook 模式

- 需要公网 IP 或内网穿透
- 需要在飞书开放平台配置事件订阅地址
- 适合有固定公网地址的服务器

```json
{
  "connectionMode": "webhook",
  "webhookPath": "/feishu/events",
  "webhookPort": 8080
}
```

## 6. 验证配置

### 6.1 检查配置状态

```bash
openclaw channels list
```

期望输出：
```
Chat channels:
- Feishu openclaw (openclaw): configured, enabled
- Feishu opengl (opengl): configured, enabled
```

### 6.2 重启 Gateway

```bash
# 查找并停止现有进程
pkill -f openclaw-gateway

# 启动 Gateway
openclaw-gateway &

# 确认运行状态
ps aux | grep openclaw-gateway
```

## 7. 使用指南

### 7.1 私聊配对

默认 `dmPolicy: pairing` 模式下，用户首次私聊机器人需要进行配对：

1. 用户私聊机器人发送任意消息
2. 机器人返回配对码
3. 管理员在终端执行配对确认命令

### 7.2 开放私聊

如需开放给所有用户私聊：

```json
{
  "channels": {
    "feishu": {
      "dmPolicy": "open",
      "allowFrom": ["*"]
    }
  }
}
```

### 7.3 群聊配置

配置允许的群组：

```json
{
  "channels": {
    "feishu": {
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["oc_xxxxxxxxxxxxxxxx"],
      "groups": {
        "oc_xxxxxxxxxxxxxxxx": {
          "enabled": true,
          "requireMention": true
        }
      }
    }
  }
}
```

## 8. 多 Agent 绑定

可以为不同飞书账号或用户绑定不同的 Agent：

```json
{
  "agents": {
    "list": [
      { "id": "agent-openclaw", "workspace": "/root/.openclaw/workspace-openclaw" },
      { "id": "agent-opengl", "workspace": "/root/.openclaw/workspace-opengl" }
    ]
  },
  "bindings": [
    {
      "agentId": "agent-openclaw",
      "match": {
        "channel": "feishu",
        "account": "openclaw"
      }
    },
    {
      "agentId": "agent-opengl",
      "match": {
        "channel": "feishu",
        "account": "opengl"
      }
    }
  ]
}
```

## 9. 故障排查

### 9.1 插件加载失败

**错误：** `Cannot find module '@sinclair/typebox'`

**解决：**
```bash
cd ~/.openclaw/extensions/feishu
npm install --legacy-peer-deps
```

### 9.2 连接失败

1. 检查 App ID 和 App Secret 是否正确
2. 检查飞书应用是否已发布
3. 检查应用权限配置
4. 查看 Gateway 日志

### 9.3 消息无响应

1. 确认 Gateway 正在运行
2. 检查私聊配对状态
3. 检查群聊是否在 allowlist 中
4. 确认群聊消息是否@了机器人

## 10. 参考配置示例

完整的 `openclaw.json` 配置示例：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "connectionMode": "websocket",
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist",
      "requireMention": true,
      "streaming": true,
      "renderMode": "auto",
      "tools": {
        "doc": true,
        "wiki": true,
        "drive": true,
        "perm": false
      },
      "accounts": {
        "openclaw": {
          "name": "openclaw",
          "appId": "cli_a91ed1b15278dccb",
          "appSecret": "RQazpCDOwMbsrck1tzTwTdLkLQ8jn1Ak"
        },
        "opengl": {
          "name": "opengl",
          "appId": "cli_a91d080d93b85bd6",
          "appSecret": "e8DarTtxDZoprbHALOI35dQNP070ad8Q"
        }
      }
    }
  }
}
```

---

**文档版本：** 2026.2.20  
**OpenClaw 版本：** 2026.2.17
