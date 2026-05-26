# lark-opencode-bridge

[![CI](https://github.com/rorschachachxd/lark-opencode-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/rorschachachxd/lark-opencode-bridge/actions/workflows/ci.yml)

A lightweight bridge that connects **Feishu/Lark** messaging with the
**opencode** AI coding agent, driven by the official [`larksuite/cli`](https://github.com/larksuite/cli)
on the messaging side and [`opencode`](https://opencode.ai) on the AI side.

```
                    ┌──────────────────────────────┐
  Feishu user msg → │ lark-cli event consume       │  (NDJSON stdout stream)
                    │   im.message.receive_v1      │
                    └──────────────┬───────────────┘
                                   │ parsed events
                                   ▼
                    ┌──────────────────────────────┐
                    │ lark-opencode-bridge         │
                    │  - per-chat session map      │
                    │  - slash commands            │
                    │  - allowlist / dedupe        │
                    └──────────────┬───────────────┘
                                   │ HTTP /session/.../prompt
                                   ▼
                    ┌──────────────────────────────┐
                    │ opencode serve (127.0.0.1)   │
                    └──────────────┬───────────────┘
                                   │ reply text
                                   ▼
                    ┌──────────────────────────────┐
       reply to user│ lark-cli im +messages-reply  │
                    └──────────────────────────────┘
```

## Status

`v0.2` — P2P chats, group `@mentions`, **and Lark document comment
`@mentions`**, with:

- **Streaming cards** (`replyStyle: "card"`) — opens an SSE connection to
  opencode's `/event`, accumulates `message.part.updated` deltas (filtered to
  assistant messages only), and PATCHes the Lark card every ~800ms.
- **Attachments** — image/file/post messages have their resources fetched via
  `lark-cli im +messages-mget` + `+messages-resources-download`, then handed
  to opencode as `FilePartInput` (`file://` URLs into
  `~/.lark-opencode-bridge/media/`).
- **Named workspaces** — `/ws save|list|use|rm`, persisted to
  `~/.lark-opencode-bridge/workspaces.json`.
- **Doc comments** — when someone `@mentions` the bot in a comment or comment
  reply on a Lark doc/docx/sheet/bitable/slides/file, the bridge picks it up
  via the Lark
  `drive.notice.comment_add_v1` event, prompts opencode with the comment
  question (and the anchored excerpt if any), and posts a reply back into the
  same comment thread.
- **`/spawn <topic>` — one session per group** — from a P2P chat, the bridge
  can create a dedicated Lark group (`[opencode] <topic>` prefix as the visual
  "opencode tag") and bind it 1:1 to a brand-new opencode session. The group
  description is set to a built-in cheat-sheet (`群内直接发消息无需 @；常用：
  /help /new /status /stop /models /cd /undo`). Inside a spawned group **every
  message goes to opencode without needing to `@mention` the bot** — the
  bridge recognises spawned groups via a persisted set in `sessions.json`, so
  this survives restarts. After the first response, the opencode-generated
  session title is synced back into the group name.

## Functional requirements

- P2P chat messages must create or reuse one opencode session per Lark chat and
  reply in the original message thread.
- Group messages must route to opencode when the bot is mentioned; groups
  created with `/spawn` must route every message without requiring an
  `@mention`.
- Attachments in image/file/post messages must be downloaded locally and passed
  to opencode as file parts.
- Feishu/Lark cloud document comments must be supported: when a user
  `@mentions` the bot in a document comment or comment reply, the bridge must
  fetch the comment thread and anchored excerpt, ask opencode for an answer,
  and post the answer back as a reply in the same comment thread.
- Streaming card mode must show progress in-place and allow the user to stop a
  running opencode request.

Event ingestion is via the `@larksuiteoapi/node-sdk` WebSocket long-connection
(lark-cli's `event consume` only covers IM messages). Outgoing actions still
go through `lark-cli` (`im +messages-send/reply`, `api PATCH` for cards,
`api POST` for comment replies).

## Getting started (step-by-step)

> **First time?** Follow these steps in order. Run **`run` in the foreground** first to scan the QR code — do **not** jump straight to `start`.

### Step 0 — Node.js

Node.js **≥ 20** required:

```bash
node -v
```

### Step 1 — Install opencode

The bridge spawns `opencode serve` locally. Install only — no separate login — but the `opencode` binary must be on `$PATH`.

```bash
# macOS / Linux (recommended)
curl -fsSL https://opencode.ai/install | bash

# or Homebrew
brew install anomalyco/tap/opencode

# or npm
npm install -g opencode-ai@latest
```

Verify in a **new terminal**:

```bash
opencode --version
```

Configure models/API keys per [opencode docs](https://opencode.ai/docs). Without a model, `/help` works but AI prompts may fail — use `/models` in chat to list providers.

### Step 2 — Install Feishu CLI (lark-cli)

Outgoing Lark actions use the official **`@larksuite/cli`** package (the unrelated npm package named `lark-cli` is **not** this tool):

```bash
npm install -g @larksuite/cli@latest
lark-cli --version
```

The bridge can auto-install lark-cli on first `run`, but manual install avoids PATH surprises.

### Step 3 — Install lark-opencode-bridge

```bash
npm install -g lark-opencode-bridge@latest
lark-opencode-bridge doctor
```

Both `lark-cli` and `opencode` should show **ok**.

### Step 4 — First foreground run + QR setup

```bash
lark-opencode-bridge run
```

Scan with the Feishu app → create/select an app → credentials land in `~/.lark-cli/config.json` and `~/.lark-opencode-bridge/secrets.json`. Wait for **`bridge ready — listening for IM messages`** before testing.

### Step 5 — Feishu developer console (required)

In https://open.feishu.cn/app/<app_id>:

1. Enable **Bot** capability
2. **Permissions** — import JSON from `lark-opencode-bridge scopes --copy`
3. **Events** — use **persistent connection** (not webhook); subscribe to `im.message.receive_v1`
4. **Publish a version** (most commonly missed step)

Optional: `lark-opencode-bridge configure` to auto-set events and card callbacks.

### Step 6 — Verify in Feishu

1. Open a **P2P** chat with your bot
2. Send `/help` — should reply immediately
3. In normal groups, **`@mention` the bot**; or use `/spawn <topic>` for a dedicated workspace group

### Step 7 — Background daemon (optional)

Only after Step 6 works (macOS / Linux):

```bash
lark-opencode-bridge start
lark-opencode-bridge status
```

Requires **global** install (`npm i -g`), not `npx`.

See also [README.zh.md](./README.zh.md) for the Chinese walkthrough.

## Prerequisites

- Node.js ≥ 20
- [`opencode`](https://opencode.ai) on `$PATH`
- Feishu CLI: [`@larksuite/cli`](https://www.npmjs.com/package/@larksuite/cli)
- A Feishu/Lark app (created via QR wizard on first `run`)

### Quick start (already set up)

```bash
npm i -g lark-opencode-bridge@latest
lark-opencode-bridge run
```

### Manual / advanced setup

If you prefer not to use the wizard:

```bash
npx @larksuite/cli@latest install
# 或
npm install -g @larksuite/cli@latest
```

### Lark developer console settings (one-time)

For each Lark app the bridge talks to, in https://open.feishu.cn/app/<app_id>:

1. **应用能力 → 机器人** — enable bot capability.
2. **权限管理** — request and enable:
   - `im:message.p2p_msg:readonly` (receive P2P messages)
   - `im:message:send_as_bot` (send messages as bot)
   - `im:chat` (create / update groups — required for `/spawn`)
   - `im:message.group_at_msg:readonly` (receive **@-bot** group messages — minimum for normal group use)
   - **`im:message.group_msg`** — **sensitive permission**, "获取群组中所有消息". Required for `/spawn`'s "no @ needed" behaviour: without it Lark simply doesn't deliver non-mention messages to the bot, no matter what the bridge does. Note the name has **no `:readonly` suffix** — that historical/typo'd form (`im:message.group_msg:readonly`) doesn't exist in the current Lark console. Sensitive permissions require **tenant admin approval** in addition to publishing a new app version.
   - `drive:file:read` *and* `drive:file.comment:create` for comment support.
3. **事件订阅 → 订阅方式** — must be **长连接 (persistent connection)**, not
   webhook. Then subscribe to:
   - `接收消息 v2.0` (`im.message.receive_v1`)
   - `文档新增评论` (`drive.notice.comment_add_v1`) — only if you want
     comment support.
4. **版本管理与发布** — publish a version (this is the most commonly missed
   step). Self-built apps additionally need admin approval.

## Build & run

```bash
npm install
npm run build

# first time: auto QR wizard if no app configured
npm start
# or
node ./bin/lark-opencode-bridge.mjs run --cwd /path/to/your/repo

node ./bin/lark-opencode-bridge.mjs doctor
```

Logs go to stdout/stderr. State is persisted under
`~/.lark-opencode-bridge/` (override with `LARK_OPENCODE_HOME`).

## In-chat commands

The bridge mirrors opencode's TUI command names where possible (see
[`opencode TUI docs`](https://opencode.ai/docs/tui/)). Older bridge-style
singular forms (`/model`, `/agent`, `/ws`) and opencode's own aliases
(`/clear`, `/summarize`, `/resume`, `/continue`) all map to the canonical
command below.

Opencode-aligned:

| Command | Aliases | Effect |
|---|---|---|
| `/help` | — | List commands |
| `/new` | `/clear` | Reset the opencode session for this chat |
| `/init` | — | Analyse the project and create/refresh `AGENTS.md` |
| `/sessions` | `/resume`, `/continue` | List opencode sessions |
| `/models` | `/model` | List providers + models opencode has available |
| `/models <provider/model>` | `/model <provider/model>` | Switch model for this chat (auto-completes the provider prefix when unambiguous) |
| `/agents` | `/agent` | List available agents |
| `/agents <name>` | `/agent <name>` | Switch agent (`build` / `plan` / …) for this chat |
| `/compact` | `/summarize` | Compact the current session (opencode `summarize`) |
| `/share` | — | Create a public share link for this session |
| `/unshare` | — | Revoke the share link |
| `/undo` | — | Revert the last user message and its file changes |
| `/redo` | — | Restore the previously reverted messages |

Bridge-specific (no opencode TUI counterpart):

| Command | Aliases | Effect |
|---|---|---|
| `/cd <path>` | — | Set the working directory for this chat |
| `/status` | — | Show current session id, cwd, agent, model |
| `/stop` | — | Abort the in-flight prompt |
| `/spawn <topic>` | `/group`, `/拉群` | **P2P only.** Create a new Lark group named `[opencode] <topic>` with a cheat-sheet in its description, invite you and the bot, bind a fresh opencode session. Inside the spawned group **every message is auto-routed to opencode (no `@mention` needed)**; the opencode-generated session title syncs back into the group name after the first response. |
| `/workspaces list` | `/ws list` | List saved workspaces |
| `/workspaces save <name> [path]` | `/ws save …` | Save the chat's current cwd (or an explicit path) as a workspace |
| `/workspaces use <name>` | `/ws use …` | Switch this chat to a saved workspace (resets the session) |
| `/workspaces rm <name>` | `/ws rm …` | Delete a workspace |

## Config

`~/.lark-opencode-bridge/config.json`:

```json
{
  "opencodePort": 4096,
  "opencodeHost": "127.0.0.1",
  "manageOpencodeServer": true,
  "larkIdentity": "bot",
  "allowedSenderOpenIds": [],
  "allowedChatIds": [],
  "replyStyle": "reply",
  "handleDocComments": true,
  "defaultCwd": "/Users/me/repo",
  "agent": "build",
  "model": "anthropic/claude-3-5-sonnet"
}
```

The `model` string is split on the first `/` into `{providerID, modelID}` (the
shape opencode expects). `defaultCwd` becomes the `directory` parameter when
the bridge creates a session for a chat. Changing cwd via `/cd` resets the
chat's session because opencode binds `directory` at session creation.

- `allowedSenderOpenIds` — empty array = anyone may talk to the bot. Add
  `ou_…` ids to restrict.
- `allowedChatIds` — same idea for `oc_…` chats.
- `replyStyle` — `"reply"` (threaded markdown) or `"card"` (interactive card).
- `handleDocComments` — `true` enables Feishu/Lark document comment
  `@mention` handling and posts opencode's answer back into the same comment
  thread.
- `manageOpencodeServer` — if `false`, point the bridge at an externally
  running `opencode serve`.

## How it works

1. `bridge run` starts `opencode serve` as a child process (skip with
   `--no-manage-server`) and waits for it to listen on
   `http://127.0.0.1:4096`.
2. It then spawns `lark-cli event consume im.message.receive_v1 --as bot
   --quiet` and reads NDJSON from stdout.
3. Each parsed event is deduped by `event_id`, filtered against the
   allowlist, and (for group chats) requires an `@_user_` mention marker in
   the message content.
4. Slash commands (see table above) short-circuit; everything else becomes an
   opencode prompt:
   - Lazy-create one opencode session per `chat_id` (persisted to disk).
   - `POST /session/{id}/prompt` with `parts: [{type:"text", text}]`,
     optionally setting `agent`, `model`, `cwd`.
   - Concatenate `text`-typed parts from the response.
5. Reply via `lark-cli im +messages-reply --message-id <om_…> --markdown
   …`.

## Layout

```
src/
├── cli.ts                # commander entry (`run`, `doctor`, `config`)
├── config.ts             # ~/.lark-opencode-bridge/config.json schema
├── session.ts            # chat_id → opencode session_id store
├── workspace.ts          # named workspace store (/ws)
├── slash.ts              # in-chat slash-command parser
├── paths.ts              # filesystem paths
├── log.ts                # tiny scoped logger
├── core/bridge.ts        # orchestrator (slash, prompt, streaming, attach,
│                           comment handler)
├── lark/
│   ├── ws-consumer.ts    # @larksuiteoapi/node-sdk WSClient (IM + comments)
│   ├── sender.ts         # `+messages-send`, `+messages-reply`, card PATCH
│   ├── attach.ts         # mget + resources-download for image/file/post
│   ├── comments.ts       # fetch comment thread + post comment reply
│   ├── credentials.ts    # resolve app_id (lark-cli config) + app_secret
│   │                       (env / keychain)
│   └── types.ts          # event schema (message + comment)
├── opencode/
│   ├── server.ts         # spawns `opencode serve`, waits for ready
│   ├── client.ts         # HTTP client (`POST /session/:id/message`)
│   └── events.ts         # SSE client for `/event`, role-aware normalization
└── card/
    ├── state.ts          # streaming CardState (text/reasoning/tool/status)
    └── render.ts         # static card builder (unused by default; kept for
                          # explicit one-shot card sends)
```

## Reply modes

`config.replyStyle`:

- **`reply`** (default) — bridge waits for the prompt to complete and posts a
  single threaded markdown reply.
- **`card`** — bridge immediately sends an interactive card showing
  "thinking…", subscribes to opencode's SSE event stream, and PATCHes the
  card every ~800ms with the latest accumulated text. Only parts that belong
  to messages with `role: assistant` are rendered, so the user's own message
  is never echoed.

## Doc comment flow

```
飞书文档评论/回复 @机器人 ───► drive.notice.comment_add_v1 (WSClient)
                          │ is_mentioned: true
                          ▼
                       fetchThread()  (GET /open-apis/drive/v1/files/.../comments/...)
                          │ extract triggering reply text + anchored quote
                          ▼
                       opencode session keyed by `doc:<file_token>`
                          │ session.prompt with question + quote + doc URL
                          ▼
                       postReply()  (POST .../comments/{commentId}/replies)
```

- Each document gets its own opencode session (keyed by `file_token`),
  so a thread of follow-up comments retains context.
- Replies are plain text capped at 2000 chars (Lark comment limit).
- Set `config.handleDocComments = false` to disable this branch (e.g. when
  the app isn't subscribed to the event yet).

## Attachments

When a Lark message of type `image`, `file`, or `post` arrives:

1. `lark-cli im +messages-mget --message-ids <om_…>` fetches the structured
   message (event payload `content` is pre-rendered text and doesn't expose
   `image_key` / `file_key`).
2. For each `image_key` / `file_key` found, `lark-cli im
   +messages-resources-download --type image|file --output <relative>`
   downloads the resource into `~/.lark-opencode-bridge/media/<message_id>/`.
3. The local file is handed to opencode as a `FilePartInput` with a
   `file://` URL and a best-effort `mime` derived from the extension.

`post` messages are walked recursively for `tag: img` and `tag: file` nodes.

## Background service

macOS (launchd) and Linux (systemd user unit) — **not supported on Windows** (use foreground `run`):

```bash
npm i -g lark-opencode-bridge   # required for stable daemon paths
lark-opencode-bridge start      # install + start
lark-opencode-bridge stop
lark-opencode-bridge restart
lark-opencode-bridge unregister
lark-opencode-bridge status
```

In-chat preferences: send `/config` for an interactive settings card (reply style, access control, group @ policy, etc.).

## Known limitations

- **Windows**: no background daemon; use `lark-opencode-bridge run`.
- **Group @ detection**: relies on `@_user_` markers in rendered content until explicit `mentions[]` is available from lark-cli.
- **Permission prompts**: card mode auto-approves opencode tool permissions for non-interactive Lark runs.

## License

MIT — see [LICENSE](./LICENSE).
