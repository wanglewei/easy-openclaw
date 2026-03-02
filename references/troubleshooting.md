# 故障排查与生效验证

## 重启 Gateway（可验证流程）

```bash
BEFORE_PID="$(openclaw status --all 2>/dev/null | rg -o 'running \\(pid [0-9]+\\)' | rg -o '[0-9]+' | head -n1)"
openclaw gateway restart
sleep 2
AFTER_PID="$(openclaw status --all 2>/dev/null | rg -o 'running \\(pid [0-9]+\\)' | rg -o '[0-9]+' | head -n1)"
echo "before=${BEFORE_PID:-none} after=${AFTER_PID:-none}"
```

若执行工具返回 `Command still running (session ...)`：
- 继续轮询该 session，直到拿到命令结束和 exit code
- 未结束前不要宣告“重启成功”

判定建议：
- `before` 与 `after` 都存在且不同：重启成功
- PID 未变化或取不到：检查日志关键字

```bash
tail -n 200 ~/.openclaw/logs/gateway.log ~/.openclaw/logs/gateway.err.log | rg 'received SIGUSR1; restarting|restart mode: full process restart|all operations and replies completed; restarting gateway now|gateway/channels] restarting discord channel'
```

说明：
- 命中 `received SIGUSR1; restarting` / `restart mode: full process restart` / `all operations and replies completed; restarting gateway now`：可视为 Gateway 已重启
- 仅命中 `gateway/channels] restarting discord channel`：仅 Discord 频道连接重启，不等同于 Gateway 全量重启

自动重启失败时：
```bash
# macOS
launchctl stop ai.openclaw.gateway && sleep 2 && launchctl start ai.openclaw.gateway

# Linux
systemctl restart openclaw-gateway
```

## 安全收紧生效验证

```bash
openclaw security audit --deep
openclaw approvals get --json
openclaw sandbox explain
```

关键检查：
- `tools.elevated: disabled`
- approvals defaults 中：`ask=always`、`askFallback=deny`、`security=allowlist`
- 若日志出现 `spawn docker ENOENT`，将 `agents.defaults.sandbox.mode` 改回 `"off"` 并重启

## Discord 无响应排查（免 @ 场景）

先看渠道探针：
```bash
openclaw channels status --probe
```

排查顺序：
1. 确认 `channels.discord.groupPolicy=allowlist`
2. 确认 `channels.discord.allowFrom` 包含目标 `guildId`
3. 确认 `channels.discord.guilds.<guildId>.requireMention=false`
4. 若存在 `channels.discord.guilds.<guildId>.channels`，先移除再重启复测
5. 观察 `--probe` 是否还有 `unresolved`

## Discord guildId 自动获取排查（固定流程）

当“免 @ 响应”拿不到 `guildId` 时，必须按以下顺序排查：

1. 提取 `channelId`（优先会话元数据中的 `channel id:<id>`）
2. 读取 token（禁止手填）：
```bash
TOKEN="$(jq -r '.channels.discord.accounts.default.token // .channels.discord.token // empty' ~/.openclaw/openclaw.json)"
```
3. 正查 `guild_id`：
```bash
curl -s "https://discord.com/api/v10/channels/<channelId>" -H "Authorization: Bot $TOKEN" | jq '{id,guild_id,name,type,message,code}'
```
4. 失败分流（仅允许）：
- `401`：重读 token 后重试一次
- `403`：补 `View Channels` 等权限
- `404`：确认 `channelId` 与机器人可见性
5. 反查校验：
```bash
curl -s "https://discord.com/api/v10/guilds/<guildId>/channels" -H "Authorization: Bot $TOKEN" | jq '.[] | select(.id=="<channelId>") | {id,name,type}'
```
6. 校验通过后再写入 `allowlist/allowFrom/requireMention=false`

禁止项：
- 禁止把 `channelId` 当 `guildId`
- 禁止先让用户手工复制服务器 ID
- 禁止尝试 `/debug/*`、`/api/*`、`/guilds/<channelId>`

## 记忆功能验证（本地模型）

```bash
openclaw memory status --deep
```

期望：
- `Provider: local`
- `Model: hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf`

## 联网搜索验证（SearXNG + openclaw browser）

先确认浏览器默认 profile：

```bash
openclaw config get browser.defaultProfile
```

期望：
- 返回 `openclaw`

再做 SearXNG 连通性检查（本地优先）：

```bash
curl -s "http://127.0.0.1:8080/search?q=openclaw&format=json" | jq '.results | length'
```

若本地未部署或暂时不可用：
- 从 `https://searx.space/` 选择可用实例
- 将上面的地址替换为实例地址后重试（保留 `search?...&format=json`）

同时检查 `~/.openclaw/workspace/TOOLS.md`：
- 是否写明“先 SearXNG，再 browser(profile=openclaw) snapshot”的顺序

## 常见坑

- 写 JSON 时带注释导致解析失败
- 未做深度合并导致用户原配置丢失
- `streaming` 误写为 `true`（应为 `"partial"`）
- `agentId` 大小写不一致导致路由错乱
- 修改后未重启 Gateway 导致看起来“配置不生效”
- 误把 Discord 频道重连当成 Gateway 全量重启
