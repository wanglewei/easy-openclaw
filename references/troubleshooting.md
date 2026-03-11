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

## 权限模式生效验证

```bash
openclaw security audit --deep
openclaw approvals get --json
openclaw sandbox explain
openclaw config get tools.profile
```

关键检查：
- 维持现状：`tools.profile` 应保持 `coding`（或用户原本的可执行状态）
- 完全开放：`tools.profile=full` 且 `sandbox.mode=off`
- 最小安全：`tools.profile=minimal`；此模式接近聊天机器人，很多目录读取/文件操作会被限制
- 审批测试默认建议：保持 `tools.profile=coding`，但不要直接拿“删除文件是否弹窗”当唯一判据
- `minimal` 下 `exec` 往往不可用，因此审批机制也没有触发机会；若目标是“可执行 + 可审批”，保持 `coding` 或 `full` 再启用审批
- 若要测 `exec` 审批，启用 `exec-approvals.json` 的 `security=allowlist + ask=on-miss`，并执行一个**明确不在默认宽 allowlist 中**的命令来触发审批，例如 `uname -a`；不要再拿 `curl` / `cat` / `ls` / `pip3` 这类默认应放行的命令做判据
- 若会话仍是宿主机默认 full access，命令可能直接执行，不会进入审批
- 若日志出现 `spawn docker ENOENT` 或 `Sandbox mode requires Docker`，将 `agents.defaults.sandbox.mode` 改回 `"off"` 并重启

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

## 记忆功能验证（强制刷新）

```bash
openclaw memory status --deep
```

期望：
- `memoryFlush.enabled=true`
- `softThresholdTokens=40000`
- 不再要求额外校验 `Provider/Model`

## 联网搜索验证（defuddle + r.jina.ai + openclaw browser）

先确认浏览器默认 profile：

```bash
openclaw config get browser.defaultProfile
```

期望：
- 返回 `openclaw`

再做正文提取连通性检查：

```bash
curl -L -s "https://defuddle.md/https://example.com" | head
curl -L -s "https://r.jina.ai/http://example.com" | head
```

判定：
- `defuddle` 成功：优先正文提取链路可用
- `r.jina.ai` 成功：备用正文提取链路可用
- 若两者都失败，再尝试浏览器兜底

同时检查 `~/.openclaw/workspace/TOOLS.md`：
- 是否写明“先 defuddle，再 r.jina.ai，最后 browser(profile=openclaw) snapshot”的顺序
- 是否写明“无图形界面环境下浏览器兜底可能不可用”

## 常见坑

- 写 JSON 时带注释导致解析失败
- 未做深度合并导致用户原配置丢失
- `streaming` 误写为 `true`（应为 `"partial"`）
- 把用户的 `hi/nihao/test/现在如何了` 之类验收消息误当成“继续修配置/继续重启”的授权
- `agentId` 大小写不一致导致路由错乱
- 修改后未重启 Gateway 导致看起来“配置不生效”
- 误把 Discord 频道重连当成 Gateway 全量重启
- 目标站点存在登录墙、Cloudflare 或 403 防护，导致 `defuddle` / `r.jina.ai` 都失败
- Docker / VPS / 无桌面环境缺少图形界面，导致浏览器兜底不可用

## 更新后复测

- 更新 `easy-openclaw` skill 后，必须开启**新会话**再复测。
- 不要拿旧会话继续验证新规则；旧 session 可能还在沿用旧版 skill 快照。
