# lark-opencode-bridge

[![CI](https://github.com/rorschachachxd/lark-opencode-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/rorschachachxd/lark-opencode-bridge/actions/workflows/ci.yml)

把飞书 / Lark 消息和本地 **opencode** CLI 打通的轻量 bot：一条命令起服务，扫码绑应用，在飞书里和 AI 对话、让它读图 / 改代码。

[English README](./README.md)

## 从零开始（推荐操作顺序）

> **给第一次用的人**：请严格按下面顺序做，**不要跳步**。  
> 第一次必须在前台跑 `run` 完成扫码；**不要**一上来就 `start` 开后台。

### 第 0 步：安装 Node.js

需要 **Node.js ≥ 20**。

```bash
node -v   # 应显示 v20.x 或更高
```

没有 Node？去 [nodejs.org](https://nodejs.org/) 下载 LTS，或用 Homebrew：`brew install node`。

---

### 第 1 步：安装 opencode

bridge 会调用本机的 `opencode` 命令（会自动起 `opencode serve`），**无需单独登录**，但必须先装好并在 PATH 里。

**方式 A — 官方脚本（macOS / Linux 推荐）**

```bash
curl -fsSL https://opencode.ai/install | bash
```

**方式 B — Homebrew**

```bash
brew install anomalyco/tap/opencode
```

**方式 C — npm**

```bash
npm install -g opencode-ai@latest
```

装完后**新开一个终端**，验证：

```bash
opencode --version
```

还要配置好 opencode 的模型 / API（按 [opencode 文档](https://opencode.ai/docs) 操作）。没配模型的话，机器人能收到消息但 AI 可能不回或报错；私聊发 `/models` 可查看可用模型。

---

### 第 2 步：安装飞书 CLI（lark-cli）

发消息、回消息、调开放平台 API 都靠它。请装官方包 **`@larksuite/cli`**（npm 上另一个叫 `lark-cli` 的包**不是**这个）。

```bash
npm install -g @larksuite/cli@latest
```

验证：

```bash
lark-cli --version
```

> 首次 `lark-opencode-bridge run` 时也会尝试自动安装/升级 lark-cli，但**建议先手动装好**，少踩 PATH 的坑。

---

### 第 3 步：安装 lark-opencode-bridge

```bash
npm install -g lark-opencode-bridge@latest
```

验证：

```bash
lark-opencode-bridge --version
lark-opencode-bridge doctor
```

`doctor` 里 `lark-cli`、`opencode` 都应是 **ok**。

---

### 第 4 步：首次前台启动 + 扫码绑应用

```bash
lark-opencode-bridge run
# 等价于直接敲：
lark-opencode-bridge
```

终端会出现二维码（macOS 还会尝试打开浏览器）：

1. 用**飞书 App** 扫码
2. 选择或创建一个应用（PersonalAgent 流程）
3. 向导会把凭据写入 `~/.lark-cli/config.json`，app secret 写入 `~/.lark-opencode-bridge/secrets.json`
4. 终端会提示打开**权限管理**页，并复制推荐权限 JSON

看到日志里出现 **`bridge ready — listening for IM messages`** 表示 bridge 已跑起来。**这个终端窗口先不要关**，方便排查。

---

### 第 5 步：飞书开放平台 — 权限、事件、发布（必做）

打开 [飞书开放平台](https://open.feishu.cn/) → 进入扫码创建的那个应用：

| 要做的事 | 说明 |
|---|---|
| **应用能力 → 机器人** | 开启机器人 |
| **权限管理** | 终端向导会复制 JSON；也可本地执行 `lark-opencode-bridge scopes --copy`，在后台「批量导入」 |
| **事件订阅** | 订阅方式选 **「使用长连接接收事件」**（不是 Webhook） |
| **版本管理与发布** | 创建新版本并**发布**（自建应用还需管理员审批） |

也可在本机补跑（自动配事件 + 卡片回调）：

```bash
lark-opencode-bridge configure
```

> **最容易漏的一步**：只加了权限但没**发布版本**，机器人会「在线但不收消息」。

---

### 第 6 步：在飞书里验证

1. 在飞书里找到你的机器人，**先私聊**
2. 发送：`/help`  
   - 有回复 → 链路通了  
   - 没回复 → 看下面「常见问题」，或终端 / `~/.lark-opencode-bridge/logs/` 里的日志
3. **普通群**里说话必须 **`@机器人`**；或用 `/spawn 主题` 建专用工作群（群内不用 @）

---

### 第 7 步（可选）：后台常驻

**确认第 6 步私聊 `/help` 正常后**，再开后台（仅 macOS / Linux）：

```bash
# 先 Ctrl+C 停掉前台 run
lark-opencode-bridge start
lark-opencode-bridge status
```

日志：`~/.lark-opencode-bridge/logs/`

> 后台服务必须用 **`npm i -g` 全局安装**，不要用 `npx`。

---

### 常见问题

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 私聊 `/help` 无回复 | 应用版本未发布 / 事件未开长连接 | 重做第 5 步 |
| 群里说话不回 | 没 @ 机器人 | 群里 `@机器人` 再发，或 `/spawn` 建群 |
| `start` 后仍不回 | 没先完成扫码，或 PATH 缺 opencode | 先 `run` 扫码；`doctor` 检查 |
| AI 不回但 `/help` 正常 | opencode 未配模型 | 配 API；私聊发 `/models` |
| `doctor` 报 opencode / lark-cli MISSING | 未安装或不在 PATH | 重做第 1、2 步，**新开终端**再试 |

---

## 能干什么

- 私聊直接发；群里 `@bot`（`/spawn` 拉群后群内无需 @）
- **流式卡片**：opencode 输出实时更新到同一张卡片
- **会话延续**：每个 chat 独立 session
- **多工作空间**：`/workspaces`（别名 `/ws`）切换项目
- **图片 / 文件 / 富文本**：下载到本地后交给 opencode
- **云文档评论**：在文档评论里 `@bot` 可提问并自动回帖

> 详细安装与操作顺序见上文 **[从零开始（推荐操作顺序）](#从零开始推荐操作顺序)**。

## 命令速查

### 宿主 CLI

```
lark-opencode-bridge              前台启动（默认 = run）
lark-opencode-bridge run          前台启动 bot
lark-opencode-bridge start        安装并启动后台 daemon（macOS / Linux）
lark-opencode-bridge stop         停止后台 daemon
lark-opencode-bridge restart      重启后台 daemon
lark-opencode-bridge unregister   撤销后台服务注册
lark-opencode-bridge setup        仅跑扫码向导（不启动 bridge）
lark-opencode-bridge configure    自动配置应用名、长连接事件、卡片回调
lark-opencode-bridge scopes       打印推荐权限 JSON
lark-opencode-bridge doctor       依赖与配置自检
lark-opencode-bridge ps           列出本机 bridge 进程
lark-opencode-bridge status       查看后台服务与配置摘要
lark-opencode-bridge service install|start|stop|uninstall  （与 start/stop 等价）
```

> 后台服务仅支持 **macOS / Linux**。`start` 等命令需 **全局安装**（`npm i -g`），勿用 npx。

### 飞书内斜杠命令

见 [English README — In-chat commands](./README.md#in-chat-commands)。常用：`/help` `/new` `/status` `/stop` `/spawn` `/workspaces` **`/config`** `/doctor` `/reconnect` `/timeout`。

## 数据目录

| 路径 | 内容 |
|---|---|
| `~/.lark-opencode-bridge/config.json` | bridge 运行配置（端口、cwd、replyStyle 等） |
| `~/.lark-opencode-bridge/secrets.json` | app secret（setup 写入，权限 600） |
| `~/.lark-opencode-bridge/sessions.json` | chat → opencode session 映射 |
| `~/.lark-opencode-bridge/workspaces.json` | 命名工作空间 |
| `~/.lark-opencode-bridge/media/` | 下载的附件 |
| `~/.lark-opencode-bridge/logs/` | 运行日志 |

`~/.lark-cli/config.json` 由 lark-cli / 扫码向导维护（app id、profile 等）。

## 许可

[MIT](./LICENSE)
