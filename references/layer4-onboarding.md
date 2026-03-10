# 第四层：新增渠道接入（第 4 轮）

## 第 4 轮：新增渠道接入（双阶段）

目标：把“平台侧准备”和“OpenClaw 本地对接”分开，先由用户完成外部准备，再由 AI 落盘配置。

### 阶段 A：用户主导的平台侧准备

按用户选择的新增渠道发送步骤与回传模板：

- 输出约束（强制）：
  - 当用户选择 `12 开，新增 <channel>` 时，必须逐条完整发送该渠道的阶段 A 步骤，不得压缩成“只回传 token/appSecret”。
  - Telegram 场景不得省略 `@BotFather`、`/newbot`、用户名规则、配对码回传。
  - 若首轮回复有遗漏，必须在下一条消息补齐完整步骤与模板，再进入阶段 B。
  - 不要向普通用户展示大段“回传模板”代码块；优先用自然语言说明“把 token 发给我”“把配对码发给我”。
  - 当用户回传 token 或 pairingCode 后，必须立即给出状态反馈和下一步动作，不能静默停住。
  - Discord 配对成功后，必须在同一条回复里直接继续主流程或进入统一执行确认，不要停在“已成功”状态等待用户追问。
  - Feishu 场景不得退化成“去开放平台创建应用并把 App ID/App Secret 发给我”这类泛化说明；必须按 1-7 与 8-12 两段完整输出。

- 海外渠道连通性前置提醒：
  - 若要接入 `discord` / `telegram` 等海外渠道，需同时确认「OpenClaw 所在服务器」与「你当前使用的网络环境」都可访问对应平台及其 API。
  - 任一侧不可达都可能导致登录、配对、收发消息或探针验证失败。

- Discord（用户侧）
  - 打开 `https://discord.com/developers/applications`，点击 `New Application` 新建应用并输入名称
  - 可按需设置头像，点击左侧 `Bot`，在页面中部点击 `Reset Token` 生成新秘钥并保存
  - 仍在 `Bot` 页面向下，5 个滑块中仅第 2 个关闭，其余 1/3/4/5 打开
  - 点击左侧 `OAuth2`，勾选 `applications.commands` 和 `bot`
  - 在 `bot permissions` 勾选：`View Channels`、`Send Messages`、`Read Message History`、`Embed Links`、`Attach Files`、`Add Reactions`
  - 页面底部生成邀请链接并邀请机器人进目标服务器；完成后把刚刚 `Reset Token` 生成的新 Bot Token 发给我
  - 我写入配置后，你去 Discord 里 `@机器人` 发一条消息测试；收到配对码后把配对码发给我，我来允许配对
  - 若你已经完成了 Discord 侧测试，也可以在发回配对码时顺手补一句“继续”，这样我会直接往后执行主流程

- Telegram（用户侧）
  - 在 Telegram 搜索 `@BotFather`（带 `@`，区分大小写，选择用户最多的官方机器人）
  - 对话输入 `/newbot`
  - 设置机器人名称：需唯一，且用户名必须以 `bot` 结尾
  - 成功后会收到 `Done!`，其中包含 Bot Token；将该 Token 回传
  - 在 Telegram 内给机器人发一条消息（或 `/start`）只会触发“配对码下发”，不等于配对完成
  - 收到配对码后必须回传，我来执行 `openclaw pairing approve telegram <pairingCode>`；执行成功后才算配对完成

- Feishu（用户侧）
  - 第一段（先做 1-7，做完先回传 `App ID/App Secret`）：
  - 1) 打开 `https://open.feishu.cn` 进入开放平台，进入开发者后台并创建企业自建应用
  - 2) 填名称/描述/头像（自定义即可）
  - 3) 在“凭证与基础信息”记录 `App ID`、`App Secret`
  - 4) 在“添加应用能力”添加机器人
  - 5) 在“权限管理-开通权限”搜索 `im:`，即时通信相关权限全部打开
  - 6) 在页面顶部创建版本并发布
  - 7) 回传 `App ID`、`App Secret`（此时由 AI 先安装插件并本地配置）
  - 第二段（AI确认中间执行成功后，再让用户做 8-12）：
  - 8) 在“事件与回调-事件配置”中将配置方式设为“长连接接收事件”
  - 9) 添加事件并搜索“接受消息”，勾选添加
  - 10) 回到“权限管理-开通权限”，搜索“通讯录基本信息”并打开相关权限
  - 11) 再次创建版本并发布
  - 12) 在飞书内搜索机器人并发起对话，收到配对码后回传

内部字段约定（不给用户展示大段模板）：
- Discord：`token`、`accountId=default`、`pairingCode`
- Telegram：`botToken`、`accountId=default`、`pairingCode`
- Feishu：`appId`、`appSecret`、`domain=feishu`、`connectionMode=websocket`、`pairingCode`

### 阶段 B：AI 执行本地对接（写配置）

仅在用户回传必要字段后执行。

- Discord：
  - 深度合并 `channels.discord.enabled=true`
  - 优先写 `channels.discord.accounts.<accountId>.token`
  - 若当前是单账号结构可写 `channels.discord.token`
  - 写入后必须立刻反馈：`Discord 配置已写入 -> 现在请去 Discord @机器人发一条消息测试 -> 收到配对码后发给我`

- Telegram：
  - 深度合并 `channels.telegram.enabled=true`
  - 写 `channels.telegram.botToken`
  - 建议补齐：`dmPolicy=pairing`、`groupPolicy=open`
  - 写入完成后必须先反馈“Telegram 配置已写入”，再提示用户触发配对码（发送 `/start` 或任意消息）
  - 写入后优先执行 `openclaw pairing list telegram` 检查是否已存在 pending pairing；若存在，必须明确告知“当前已写入待配对，请把刚收到的配对码发给我，我来继续 approve”，不能直接宣告成功
  - 若用户已在同一轮回传配对码，则立刻执行 `openclaw pairing approve telegram <pairingCode>`
  - 严禁宣告“Telegram 接入成功”直到配对码已回传且 `pairing approve` 执行成功

- Feishu：
  - 先检查/安装插件：
    - `openclaw plugins info feishu --json`
    - 若不可用，执行 `openclaw plugins install @openclaw/feishu`
    - `openclaw plugins enable feishu`
  - 深度合并 `channels.feishu.enabled=true`
  - 写 `channels.feishu.appId`、`channels.feishu.appSecret`
  - 建议补齐：`domain=feishu`、`connectionMode=websocket`、`dmPolicy=pairing`
  - 执行一次前置连接验证：`openclaw gateway restart` + `openclaw channels status --probe`
  - 验证通过后再让用户执行 Feishu 后续 8-12 步

配对码回传后放行：
- 统一优先命令：`openclaw pairing approve <channel> <pairingCode>`
- 示例：
  - Discord：`openclaw pairing approve discord <pairingCode>`
  - Telegram：`openclaw pairing approve telegram <pairingCode>`
  - Feishu：`openclaw pairing approve feishu <pairingCode>`
- 仅在多账号冲突时再补：`--account <accountId>`

配对后续流转（强制）：
- Discord pairing approve 成功后，必须在同一条回复里明确给出：
  - `Discord 配对成功`
  - `本轮新增渠道已完成`
  - 若前面还有统一执行确认未完成，则直接继续输出“确认后我将统一执行以下变更”
- Telegram pairing approve 成功后，也必须在同一条回复里明确给出：
  - `Telegram 配对成功`
  - `本轮新增渠道已完成`
  - 若前面还有统一执行确认未完成，则直接继续输出“确认后我将统一执行以下变更”
- 禁止在配对成功后停住不说下一步。

配对完成判定（Telegram）：
- 已执行 `openclaw pairing approve telegram <pairingCode>` 且命令成功
- `openclaw channels status --probe` 显示 Telegram `works`
- 未满足以上条件时，状态必须标记为“已写入待配对”，不能标记为“接入成功”

执行后验证：

```bash
openclaw channels status --probe
```

判定：
- 仅对“本轮新增渠道”检查 `enabled/configured/running/works`
- 失败时输出“失败项 + 缺失字段 + 下一步修复建议”

安全要求：
- 日志或总结中对 token/appSecret 脱敏显示
- 不在频道公开敏感凭据（优先引导用户私聊或分段提供）
- 若仅 Feishu 执行了前置重启且后续无新增写入，可跳过最终重复重启
