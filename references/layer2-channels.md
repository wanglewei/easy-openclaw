# 第二层：渠道增强层（第 2 轮）

## 6) Discord 频道免 @ 响应

适用场景：在 Discord 服务器频道里，不希望每次都必须 @ 机器人才触发回复。

推荐写法（`<serverId>` 为目标服务器 ID，最小稳态）：

```json
"channels": {
  "discord": {
    "groupPolicy": "allowlist",
    "allowFrom": ["<serverId>"],
    "guilds": {
      "<serverId>": {
        "requireMention": false
      }
    }
  }
}
```

执行要点：
- `allowFrom` 做追加去重，不覆盖已有值
- 若当前在 Discord 频道会话中，先读取当前 `channelId`
- `channelId` 获取优先级：
  - 先从会话元数据提取（如 `conversation_label` 中的 `channel id:<id>`）
  - 再从当前消息上下文字段提取（若有）
  - 若仍缺失，再尝试 `openclaw directory groups list --channel discord` 反查
- 默认执行 `channelId -> Discord API GET /channels/<channelId> -> guild_id`，再把 `guild_id` 作为 `<serverId>`
- 解析成功后直接继续，不要求用户手工提供服务器 ID
- 请求前先确认 Token 来源（优先 `channels.discord.accounts.<accountId>.token`，否则 `channels.discord.token`），并使用 `Authorization: Bot <token>`
- 推荐查询模板（正查）：
  - `curl -s "https://discord.com/api/v10/channels/<channelId>" -H "Authorization: Bot <token>" | jq '{id,guild_id,name,type,message,code}'`
- 反查模板（校验 `<channelId>` 是否属于 `<guildId>`）：
  - `curl -s "https://discord.com/api/v10/guilds/<guildId>/channels" -H "Authorization: Bot <token>" | jq '.[] | select(.id=="<channelId>") | {id,name,type}'`
- 常见失败分流：
  - `401 Unauthorized`：Token 无效/过期/字段取错（重读配置 token 并重试）
  - `403 Forbidden`：机器人缺频道可见权限（补 `View Channels`）
  - `404 Not Found`：`channelId` 错误或机器人不可见该频道
- 若当前为 DM、拿不到 `channelId`、或 API 解析失败，才向用户询问服务器 ID
- `<serverId>` 必须是 `guildId`，不能填频道 ID（`channelId`）
- 建议配合 `groupPolicy: "allowlist"`，避免开放群组带来的风险
- 默认不要写 `guilds.<serverId>.channels`
- 禁止使用 `channels."".allow=true` 作为默认全频道放行写法（部分版本会导致频道消息静默丢弃）

固定执行链（覆盖上文，低参数模型必须使用）：
1. 先取 `channelId`：会话元数据 -> 消息上下文字段 -> `openclaw directory groups list --channel discord`
2. 固定取 token：`TOKEN="$(jq -r '.channels.discord.accounts.default.token // .channels.discord.token // empty' ~/.openclaw/openclaw.json)"`
3. 固定正查：`curl -s "https://discord.com/api/v10/channels/<channelId>" -H "Authorization: Bot $TOKEN" | jq '{id,guild_id,name,type,message,code}'`
4. 固定分流：
  - `401` 只允许重试一次
  - `403` 提示补权限
  - `404` 提示频道不可见或 ID 错误
5. 固定反查：`curl -s "https://discord.com/api/v10/guilds/<guildId>/channels" -H "Authorization: Bot $TOKEN" | jq '.[] | select(.id=="<channelId>") | {id,name,type}'`
6. 通过后再写免 @ 配置（`allowlist + allowFrom + requireMention=false`）

禁止项：
- 禁止把 `channelId` 当 `guildId`
- 禁止先让用户去手工复制服务器 ID
- 禁止偏航尝试 `/debug/*`、`/api/*`、`/guilds/<channelId>`

仅在用户明确要求“限制到某个频道”时，使用：

```json
"channels": {
  "discord": {
    "guilds": {
      "<serverId>": {
        "channels": {
          "<channelId>": {
            "allow": true
          }
        }
      }
    }
  }
}
```

## 7) Exec 高危操作审批（跨渠道，可选）

适用场景：当前权限模式不是 `minimal`，希望把“未命中 allowlist 的高危命令”改成先审批再执行。

定位说明：
- 这一项属于第 2 轮统一收集，不再放在第 1 轮“权限模式”里。
- 第 1 轮第 5 项只决定 `coding / full / minimal`。
- 只有在 `coding` / `full` 下，这一项才有意义；`minimal` 下 exec 往往不可用，审批也没有触发机会。
- 默认建议关闭；只有用户明确需要“先审批再执行”时再开启。因为开启后很多读取、检查、执行动作也会更繁琐。

可选值：
- `关`
- `session`：审批提示跟随当前会话
- `targets`：审批提示固定发到指定账号/频道
- `both`：当前会话和固定目标都发

执行规则：
- 审批策略统一使用 `exec-approvals.json` 的 `security=allowlist + ask=on-miss + askFallback=deny`
- 审批提示投递统一写到 `openclaw.json` 的 `approvals.exec`
- 若用户选择 `targets` / `both`，优先自动提取当前会话目标：
  - Telegram：优先 `chat_id`，回退 `sender_id`
  - Discord：频道会话优先 `channelId`，写成 `channel:<id>`；DM 优先用户 ID，写成 `user:<id>`
  - Feishu：优先 `chat_id`，回退 `open_id`
- 若当前会话拿不到目标 ID，才让用户手动提供
- 若第 5 项选择了 `最小安全`，则这一项必须强制关闭，不再继续追问投递目标

建议优先参考：`references/layer1-base.md` 中的“Exec 高危操作审批（机制说明；实际收集在第 2 轮）”。

## 8) Discord 审批按钮（可选）

适用场景：已在第 2 轮开启第 7 项审批后，希望在 Discord 内用按钮完成审批。

前提（强制）：
- 审批提示必须先能投递到 Discord：在第 2 轮第 7 项中，将 `approvals.exec.mode` 设为 `session`（并在 Discord 会话触发）或 `targets/both`（包含 `{"channel":"discord","to":"user:<id>"}` / `{"channel":"discord","to":"channel:<id>"}`）。
- 本段配置只影响“在 Discord 内以按钮形式审批”，不负责“审批提示投递到哪里”。

```json
"channels": {
  "discord": {
    "execApprovals": {
      "enabled": true,
      "approvers": ["你的Discord用户ID"],
      "target": "both",
      "agentFilter": ["main"],
      "sessionFilter": ["discord"]
    }
  }
}
```

## 9) 飞书探测缓存优化

适用场景：用户在用 Feishu 渠道，默认每分钟探测连接会消耗大量 API 配额。

优化目标：
- 对探测结果做 24 小时缓存
- 按 `appId + domain` 分桶缓存
- 减少不必要的重复探测请求

目标文件：`~/.openclaw/extensions/feishu/src/probe.ts`

将其更新为以下代码：

```ts
import type { FeishuConfig, FeishuProbeResult } from "./types.js";
import { createFeishuClient } from "./client.js";
import { resolveFeishuCredentials } from "./accounts.js";

// Cache probe results to avoid hitting API rate limits
// Cache for 24 hours (86400 seconds)
const PROBE_CACHE_TTL_MS = 24 * 60 * 60 * 1000;
const probeCache = new Map<string, { result: FeishuProbeResult; timestamp: number }>();

function getCacheKey(cfg?: FeishuConfig): string {
  if (!cfg?.appId) return "no-creds";
  return `${cfg.appId}:${cfg.domain ?? "feishu"}`;
}

export async function probeFeishu(cfg?: FeishuConfig): Promise<FeishuProbeResult> {
  const creds = resolveFeishuCredentials(cfg);
  if (!creds) {
    return {
      ok: false,
      error: "missing credentials (appId, appSecret)",
    };
  }

  // Check cache first
  const cacheKey = getCacheKey(cfg);
  const cached = probeCache.get(cacheKey);
  if (cached && Date.now() - cached.timestamp < PROBE_CACHE_TTL_MS) {
    return cached.result;
  }

  try {
    const client = createFeishuClient(cfg!);
    // Use im.chat.list as a simple connectivity test
    // The bot info API path varies by SDK version
    const response = await (client as any).request({
      method: "GET",
      url: "/open-apis/bot/v3/info",
      data: {},
    });

    if (response.code !== 0) {
      const result = {
        ok: false,
        appId: creds.appId,
        error: `API error: ${response.msg || `code ${response.code}`}`,
      };
      probeCache.set(cacheKey, { result, timestamp: Date.now() });
      return result;
    }

    const bot = response.bot || response.data?.bot;
    const result = {
      ok: true,
      appId: creds.appId,
      botName: bot?.bot_name,
      botOpenId: bot?.open_id,
    };
    probeCache.set(cacheKey, { result, timestamp: Date.now() });
    return result;
  } catch (err) {
    const result = {
      ok: false,
      appId: creds.appId,
      error: err instanceof Error ? err.message : String(err),
    };
    probeCache.set(cacheKey, { result, timestamp: Date.now() });
    return result;
  }
}

// Clear the probe cache (useful for testing or when credentials change)
export function clearProbeCache(): void {
  probeCache.clear();
}

// Export for testing
export { PROBE_CACHE_TTL_MS };
```

若文件不存在：提示用户插件路径缺失并跳过该项，不中断其他配置。

## 10) Telegram

Telegram 在第 2 轮没有单独的渠道增强项，但可以直接使用第 7 项“Exec 高危操作审批”。

建议：
- 若只想在当前 Telegram 对话里收审批，优先选 `session`
- 若要长期固定投递，选 `targets` 或 `both`，并优先自动提取当前会话的 `chat_id`
