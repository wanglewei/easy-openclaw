# 第一层：基础推荐层（第 1 轮）

## 备份（全量备份，推荐）

备份目标：整个 `~/.openclaw/`（包含配置、workspace、agents、cron、extensions 等）。

备份保存目录：`~/openclaw-backups/`（强制放在 `~/.openclaw/` 之外，避免下次备份把旧备份再次打包导致“滚雪球”）

命名规则：`backup-openclaw-all-YYYYMMDD-HHMMSS.zip`

执行命令：
```bash
BACKUP_DIR="$HOME/openclaw-backups"
mkdir -p "$BACKUP_DIR" 2>/dev/null || BACKUP_DIR="$HOME/.openclaw/backups"
mkdir -p "$BACKUP_DIR"
cd ~ && zip -r "$BACKUP_DIR/backup-openclaw-all-$(date +%Y%m%d-%H%M%S).zip" .openclaw/ -x ".openclaw/backups/*"
```

## 1) 流式消息

推荐写法：
```json
"channels": {
  "discord": {
    "streaming": "partial"
  }
}
```

合法值：
- `"off"`
- `"partial"`（推荐）
- `"block"`
- `"progress"`

硬规则：
- `streaming` 只能写以上合法枚举值。
- 当前 skill 的默认写法一律是 `"partial"`；禁止写成布尔值 `true/false`。

可配合：
```json
"agents": {
  "defaults": {
    "blockStreamingDefault": "on",
    "blockStreamingBreak": "text_end"
  }
}
```

## 2) 记忆功能（记忆增强 + 可选每天归档）

### 2.1 记忆增强（强制刷新，推荐）

```json
"agents": {
  "defaults": {
    "compaction": {
      "memoryFlush": {
        "enabled": true,
        "softThresholdTokens": 40000
      }
    }
  }
}
```

注意：
- `compaction` 必须深度合并，保留原有字段。
- 不再主动写 `memorySearch`、`provider`、`modelPath` 等向量检索配置，交由 OpenClaw 新版默认能力接管。

### 2.2 每日记忆自动归档（定时总结，可选）

目标：每天定时让 AI 把“当天会话要点”整理成一份归档，写入目标 agent 的 `memory/YYYY-MM-DD.md`。

关键说明（避免误解）：
- 这是 **OpenClaw 内置 Cron（Gateway 调度器）**，不是系统 `crontab`。
- **不会自动覆盖所有 agent**：每个 agent 都需要单独创建一条 cron 任务；只创建 main 的任务不会更新 `workspace-amadeus`、`workspace-workflow` 等其他 workspace。
- 推荐在 `sessionTarget="isolated"` 的独立会话中执行，避免污染主对话上下文。
- 默认不做通知：`delivery.mode="none"`（需要通知时再走“announce + channel/to”的链路）。

#### 1) 查看 cron 任务列表 / 获取真实 job id

```bash
openclaw cron list
```

若需要从文件侧排查，可查看：`~/.openclaw/cron/jobs.json`。

#### 2) 创建“每日记忆自动归档”任务（示例）

下面示例以“每天 23:00（Asia/Shanghai）”为例；`agentId`、`model` 可按你的环境调整。

若可直接调用 cron 工具（推荐）：

```json
{
  "action": "add",
  "job": {
    "name": "每日记忆自动归档",
    "agentId": "<target-agent-id>",
    "schedule": { "expr": "0 23 * * *", "kind": "cron", "tz": "Asia/Shanghai" },
    "sessionTarget": "isolated",
    "payload": {
      "kind": "agentTurn",
      "message": "请执行每日记忆归档：\n1) 汇总今天（按当前时区）的会话要点\n2) 输出一个简短摘要 + 待办/结论\n3) 写入 memory/YYYY-MM-DD.md（YYYY-MM-DD 为今天日期）\n完成后回复 NO_REPLY（若不支持则简短回复完成）",
      "model": "<optional-model-id>"
    },
    "delivery": { "mode": "none" }
  }
}
```

#### 3) 手动触发一次（用于验收）

强制要求：手动触发前，必须先从 `cron list` 或 `jobs.json` 解析到 **真实 `job id`**（不要把任务名当成 id）。

```bash
openclaw cron run <job-id>
```

验收口径（建议）：
- 目标 workspace 下出现 `memory/YYYY-MM-DD.md`
- 文件内容包含“今日摘要/要点/结论或待办”

## 3) 消息回执

```json
"messages": {
  "ackReactionScope": "all"
}
```

## 4) 联网搜索

目标：不给用户强绑 Brave API，也不要求本地部署搜索服务；优先用网页正文读取服务，再用隔离浏览器兜底抓取页面内容。

推荐策略（按优先级）：
1) `https://defuddle.md/https://目标网址`（主力正文提取）
2) `https://r.jina.ai/http://目标网址`（备用正文提取）
3) 浏览器隔离抓取兜底（`browser.defaultProfile="openclaw"`）

`openclaw.json` 建议写入（浏览器兜底）：

```json
"browser": {
  "enabled": true,
  "defaultProfile": "openclaw"
}
```

说明：
- `defaultProfile: "chrome"` 会走扩展 relay 和用户日常浏览器上下文，不适合作为默认自动化通道。
- `defaultProfile: "openclaw"` 使用 OpenClaw 托管隔离 profile，更安全也更符合开箱路径。
- 若环境是 Docker 容器、无桌面 VPS、纯命令行虚拟机等无图形界面场景，浏览器兜底可能不可用；此时只依赖前两层正文提取。

写入 `~/.openclaw/workspace/TOOLS.md`（告知 agent 的操作偏好）：

```markdown
## 搜索服务（默认）
- 默认不部署本地搜索服务，也不强依赖 Brave API。
- 第 1 优先：`https://defuddle.md/https://目标网址`
- 第 2 优先：`https://r.jina.ai/http://目标网址`
- 第 3 优先：若前两者失败，再用 `browser`（profile=`openclaw`）打开网页并 snapshot。
- 若环境无图形界面，浏览器兜底可能不可用，不作为联网搜索成功前提。
```

正文提取连通性检查：

```bash
curl -L -s "https://defuddle.md/https://example.com" | head
curl -L -s "https://r.jina.ai/http://example.com" | head
```

## 5) 权限模式（强烈建议保持 coding，可选）

目标：默认不做“权限收紧”，而是先探测当前 OpenClaw 权限状态，再由用户选择保持 `coding`、完全放开 `full`，或切到“纯聊天机器人”形态的 `minimal`。

关键结论（避免误解）：
- `minimal` profile 下，`exec` 工具组通常会被禁用（很多基础命令都跑不了），因此 **审批机制也没有触发机会**。
- 只有在 `coding` / `full` 这种 **仍允许执行工具** 的 profile 下，再叠加 `exec` 审批（`security=allowlist + ask=on-miss`）才有意义：不在 allowlist 的命令才会弹出审批。
- 用户是否启用审批，不在第 1 轮收集；统一放到第 2 轮的“Exec 高危操作审批”里收集。

推荐顺序：
1) 维持现状（默认，强烈推荐：`coding`）
2) 完全开放（`full`，高风险）
3) 最小安全（`minimal` + 沙箱，最严格，接近聊天模式）

探测命令（固定）：

```bash
openclaw config get tools.profile
openclaw config get agents.defaults.sandbox.mode
command -v docker || echo docker_missing
```

判定：
- 若当前是 `tools.profile="coding"`：可直接继续执行本 skill，大多数基础功能不受影响。
- 若当前是 `tools.profile="minimal"` 或其他明显受限状态：先提示用户手动执行 `openclaw config set tools.profile coding`，再继续本 skill。`minimal` 下很多列目录、读文件、执行动作都会被拦住，审批链路也很难触发。
- 若当前已是 `tools.profile="full"`：说明环境处于完全放开模式，应额外提醒用户风险。

### A. 维持现状（默认）

推荐给大多数用户：

```json
"tools": {
  "profile": "coding"
},
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "off"
    }
  }
}
```

说明：
- `coding` 权限通常已经足够，是当前这套 skill 的原生体验，强烈建议保持。
- 能读写配置、执行常规命令、完成大部分自动化修改。
- 若要启用审批，建议保持 `coding`，并在第 2 轮再叠加“exec 高危操作审批”。
- 重要：仅保留 `coding` 不代表“删除文件 / sudo 一定会触发审批”。OpenClaw 新版里，`tools.profile` 只决定工具集合，不直接等于 exec 审批策略。
- 若要验证审批，优先测试 `exec` 审批路径：确保 `exec-approvals.json` 已启用 `security=allowlist + ask=on-miss`，并执行一个“不在 allowlist 中”的命令来触发审批；不要把“rm 没弹审批”直接理解成 Telegram / Discord 审批联动失效。
- 默认不额外改动用户现有权限配置；若探测到已是 `coding`，可直接维持。

若当前权限太低，提示用户执行：

```bash
openclaw config set tools.profile coding
```

### B. 完全开放（`full`，高风险）

适用场景：追求“贾维斯狂奔模式”，希望 agent 尽量少受限制。

```json
"tools": {
  "profile": "full"
},
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "off"
    }
  }
}
```

强提醒：
- 这是完全放开模式，风险最高。
- 不建议在生产环境、主力电脑、重要账号环境下长期启用。
- 必须明确提醒用户：当前 agent 的执行自由度明显更高。

### C. 最小安全（`minimal` + 沙箱）

适用场景：把 OpenClaw 尽量收回到“偏聊天机器人”的状态。此模式非常严格，接近日常聊天，不适合拿来测审批联动。

```json
"tools": {
  "profile": "minimal"
},
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "all"
    }
  }
}
```

关键前提：
- 只有系统已安装 Docker，才允许写入 `sandbox.mode="all"`。
- 若未安装 Docker，直接开启沙箱会导致 agent 启动失败，并出现 `Sandbox mode requires Docker` / `docker command was not found in PATH`。

处理规则：
- 若用户选择最小安全，先检查 `docker` 是否可用。
- 若 `docker` 缺失：先提示安装 Docker；安装前不得宣告“最小安全模式已完成”。
- 若环境可自动修复（如 Debian/Ubuntu 且有 `sudo`）：可在用户同意后先安装 Docker。
- 若无法自动修复：明确告知用户先装 Docker，随后再开启 `minimal + sandbox`。

### Exec 高危操作审批（机制说明；实际收集在第 2 轮，仅 coding/full 有效）

设计目标：
- **自动执行（allowlist）**：常用低风险命令尽量直接放行，避免 `ls/cat/curl/head` 这类日常操作也频繁弹窗
- **必须审批（不在 allowlist）**：少见或高风险操作通过“缺省拒绝 + ask=on-miss”触发审批

核心机制：
- `security=allowlist`：只有命中 allowlist 的命令才会自动执行
- `ask=on-miss`：未命中 allowlist 时弹出审批
- `askFallback=deny`：无法触发审批或超时等情况默认拒绝

实用策略（当前仓库默认推荐）：
- 默认采用“宽 allowlist，少打扰”的方式，优先覆盖大家最常用的读写/检索/开发命令：
  - `ls` / `cat` / `grep` / `rg` / `cp` / `find` / `pwd` / `echo` / `whoami` / `sed` / `head` / `tail`
  - `mkdir` / `mv` / `touch` / `tree` / `which` / `jq` / `curl`
  - `openclaw` / `git` / `python` / `python3` / `pip` / `pip3` / `npm` / `bun` / `pytest` / `uv`
  - 若后续会装 `Agent Reach` / `Youtube Clipper`：再加 `yt-dlp` / `agent-reach`
- 这样做的目标是：正常读取文件、拉 README、安装依赖、跑常规开发命令不该每一步都审批。
- 当前 OpenClaw CLI（`2026.2.24`）只有 `allowlist add/remove` 与整份 `set/get`，**没有独立的 denylist 字段**。所以：
  - 如果你整条放行 `git` / `npm` / `bun` 二进制，那么 `git push/pull/reset/clean`、`npm publish`、`bun publish` 这类子命令也可能被一起放行。
  - 本仓库当前采用“实用优先”默认值，先解决“什么都要审批”的问题；只有用户明确要求更细粒度拦截时，才再缩小 pattern。

`~/.openclaw/exec-approvals.json` 示例（模板，按实际路径调整）：

```json
{
  "version": 1,
  "defaults": { "security": "allowlist", "ask": "on-miss", "askFallback": "deny", "autoAllowSkills": true },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        { "pattern": "/usr/bin/ls" },
        { "pattern": "/bin/pwd" },
        { "pattern": "/usr/bin/cat" },
        { "pattern": "/usr/bin/grep" },
        { "pattern": "/bin/cp" },
        { "pattern": "/usr/bin/find" },
        { "pattern": "/usr/bin/rg" },
        { "pattern": "/usr/bin/echo" },
        { "pattern": "/usr/bin/whoami" },
        { "pattern": "/usr/bin/sed" },
        { "pattern": "/usr/bin/head" },
        { "pattern": "/usr/bin/tail" },
        { "pattern": "/bin/mkdir" },
        { "pattern": "/bin/mv" },
        { "pattern": "/usr/bin/touch" },
        { "pattern": "/opt/homebrew/bin/tree" },
        { "pattern": "/usr/bin/which" },
        { "pattern": "/usr/bin/jq" },
        { "pattern": "/usr/bin/curl" },
        { "pattern": "/usr/bin/openclaw" },
        { "pattern": "/opt/homebrew/bin/openclaw" },
        { "pattern": "/usr/bin/git" },
        { "pattern": "/opt/homebrew/bin/git" },
        { "pattern": "/usr/bin/python" },
        { "pattern": "/usr/bin/python3" },
        { "pattern": "/usr/bin/pip" },
        { "pattern": "/usr/bin/pip3" },
        { "pattern": "/usr/bin/npm" },
        { "pattern": "/opt/homebrew/bin/bun" },
        { "pattern": "/usr/bin/pytest" },
        { "pattern": "/opt/homebrew/bin/uv" }
      ]
    }
  }
}
```

写入方式（强制）：
- 不要直接照抄仓库里的示例路径；写入前必须先用 `command -v ls pwd cat grep rg cp find echo whoami sed head tail mkdir mv touch tree which jq curl openclaw git python python3 pip pip3 npm bun pytest uv` 获取当前机器的真实路径。
- 若本轮或后续会使用 `Agent Reach` / `Youtube Clipper`，建议把 `yt-dlp` / `agent-reach` 也纳入默认低风险 allowlist。
- 只把“当前机器上实际存在”的路径放进 allowlist。
- 再将生成后的 JSON 通过 `openclaw approvals set --stdin --json` 写入。

审批命令（在接收审批的窗口内执行）：
- `/approve <id> allow-once`
- `/approve <id> allow-always`
- `/approve <id> deny`

注意：
- OpenClaw `2026.2.24` 下，`tools.exec.askFallback` 不是合法键，`askFallback` 要放在 `exec-approvals.json`。
- allowlist 建议用实际二进制路径（可用 `command -v ls`、`command -v git` 获取）；macOS Homebrew 常见路径是 `/opt/homebrew/bin/*`。
- 当前默认策略是“先别让常用命令一直打扰用户”；因此 `curl`、`pip3`、`yt-dlp`、`agent-reach` 这类常见读取/安装命令应优先被视为低风险放行对象。
- 如果只允许 `{ "pattern": "/usr/bin/git" }` 这类“放行整个二进制”，那么 `git push/pull/reset/clean` 也会一并被自动放行；`npm` / `bun` 同理。若你想让这些高危子命令单独走审批，需要使用更细的 pattern（按子命令/参数）或干脆先不默认放行整条二进制。
- 若开启审批后要做读取/验收，优先一条命令一条命令执行；不要默认用 `ls && cat`、长管道或复合 shell 命令，否则更容易连续触发多条审批。
- 若审批消息里的“Reply with”示例缺少 `<id>`，仍应以消息中展示的**完整审批 ID** 为准，执行 `/approve <full-id> allow-once` / `allow-always` / `deny`。
- `allow-once|allow-always|deny` 是“三选一占位符”，不能整串原样输入；正确命令必须只保留其中一个动作。
- 若用户一时看不懂，agent 应直接回一条可复制的完整命令，例如：`/approve 49a500da-bd57-4458-a320-f1e65281f0d5 allow-once`。
- 一旦出现一条待审批命令，agent 应暂停继续发新命令，先等待这条审批解决；不要在同一轮里连续堆出多个 pending approval。

### 审批提示投递（`approvals.exec`，一次性配置）

上面 `exec-approvals.json` 解决的是“哪些命令会触发审批”。而 **审批提示发到哪里**（Discord/Telegram/…）由 `~/.openclaw/openclaw.json` 里的 `approvals.exec` 决定。

推荐写法（示例）：

```json
{
  "approvals": {
    "exec": {
      "enabled": true,
      "mode": "both",
      "targets": [
        { "channel": "telegram", "to": "123456789" },
        { "channel": "discord", "to": "user:123456789012345678" }
      ]
    }
  }
}
```

说明：
- `mode`：`session`（跟随当前会话）/ `targets`（固定投递到指定账号/频道）/ `both`（两者都发）
- Telegram：`to` 填你的 Telegram 用户 ID（可用 `@userinfobot` 查询）
- Discord：建议显式写 `user:<id>`（DM）或 `channel:<id>`（频道），避免歧义
- 前提：对应渠道必须已接入并可发消息，否则审批提示无法投递

自动获取建议（尽量少手填）：
- 如果你希望固定投递到某个 Telegram/Discord/飞书窗口，最稳的方式是：**在那个窗口里与机器人对话**，然后再启用 `mode=targets/both`。
- 只要当前会话元数据里已经有目标字段，Skill 就可以自动落盘：
  - Telegram：优先 `chat_id`，回退 `sender_id`
  - Discord：优先 `channelId`（频道），DM 则需要用户 ID（拿不到时用 Discord Developer Mode 复制）
  - Feishu：优先 `chat_id`（群），回退 `open_id`（单聊）
