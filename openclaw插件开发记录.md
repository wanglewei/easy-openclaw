# OpenClaw 插件开发记录

## 项目目的

本项目 `easy-openclaw` 的目标是提供一个对话式配置向导，让用户在尽量少的交互轮次内完成 OpenClaw 常用优化，并保证：
- 先收集选择，再统一执行
- 深度合并配置，不覆盖用户已有有效设置
- 最后只重启一次 Gateway

---

## 当前版本（最终思路）

当前采用“第 0 层测试观测（可选）+ 三层优化 + 第四轮接入”：

0. 测试观测层（可选）
- 测试版可开启详细观测输出
- 每步输出“动作/结果/下一步”
- 写入前后值、重启前后 PID、验收命令结果均可见

1. 基础推荐层（通用）
- 流式消息
- 记忆功能
- 消息回执
- 联网搜索
- 安全收紧（全局执行策略，可选：更安全但审批交互会变多）

2. 渠道增强层（按用户所选渠道动态出现）
- Discord：频道免 @ 响应、审批转发联动
- Feishu：探测 24h 缓存优化
- Telegram：审批联动（在 Telegram 内接收审批并 `/approve`）

3. Skills 推荐层（可执行）
- 第 3 轮默认直接展示推荐清单，不再使用“11 开”子菜单
- 支持“跳过第三层”
- 用户点名 Skill 后立即执行安装链路（README 安装段落 -> 执行 -> 最小验证）
- awesome 合集默认先抓取并输出精选摘要，再按用户意图继续展开/安装

交互节奏：
- Step 0：备份/恢复
- 第 1 轮：当前渠道识别（单选：`discord`/`feishu`/`telegram`/`tui`）+ 基础推荐层
- 第 2 轮：渠道增强层（按当前单一渠道提问）
- 第 3 轮：Skills 推荐层（可一键跳过）
- 第 4 轮：新增渠道接入引导（可一键跳过）
- `tui` 分支：若用户当前未接入渠道，直接跳到第 4 轮
- 条件补充：只问已开启项
- 最终确认后统一写入、统一重启

---

## 关键配置原则

### 1) 安全收紧（全局）
必须同时满足：
- `openclaw.json` 执行策略（`tools.exec`、`tools.elevated`、`approvals.exec`）
- `exec-approvals.json` defaults（`ask`、`askFallback`、`security`）

### 2) Discord 审批转发联动（渠道层）
- 放在 Discord 分支，不与全局安全收紧混在同一层
- 用于在 Discord 中接收审批并执行 `/approve`

### 3) Discord 免 @ 响应
- `channels.discord.groupPolicy = "allowlist"`
- `channels.discord.allowFrom` 包含目标服务器 ID（追加去重）
- `channels.discord.guilds.<serverId>.requireMention = false`
- 默认不写 `channels.discord.guilds.<serverId>.channels`
- 仅在用户明确要求“限制到某个频道”时，才写 `channels.<channelId>.allow = true`
- 避免使用 `channels."".allow=true` 作为默认写法（部分版本会导致频道消息静默丢弃）

### 4) Feishu 限额优化
- 修改 `~/.openclaw/extensions/feishu/src/probe.ts`
- 探测结果按 `appId + domain` 做 24h 缓存

### 5) 记忆功能（本地模型）
- 启用记忆时默认配置本地 embedding 模型：
  `hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf`
- 默认 `fallback=none`，保持纯本地模式
- 首次使用会自动下载约 329MB 模型文件
### 6) Docker 兼容规则
- 若环境无 Docker，不要使用 `sandbox.mode = "all"`
- 遇到 `spawn docker ENOENT`，应回退为 `sandbox.mode = "off"`

---

## 近期补充（已验证）

### Discord 频道“在线但不回复”的真实坑点
- 现象：Bot 在线、探针显示可用，但频道消息被静默丢弃。
- 触发条件：`groupPolicy=allowlist` 下写入了不稳定/歧义的 `guilds.<id>.channels` 映射（尤其是默认全放行写法）。
- 稳定修复：
  - 保留最小稳态三项：`groupPolicy=allowlist`、`allowFrom` 包含 `guildId`、`requireMention=false`
  - 移除不必要的 `guilds.<id>.channels` 配置
  - 执行 `openclaw channels status --probe`，确认无 `unresolved`
  - 重启 Gateway 后复测 DM 与频道

---

## 当前仓库关键文件

- `SKILL.md`：主流程与对话行为规则（三层优化 + 第四轮接入）
- `references/configs.md`：总览入口（已拆分）
- `references/layer0-testing.md`：第 0 层测试观测模式（可选）
- `references/layer1-base.md`：第一层基础推荐
- `references/layer2-channels.md`：第二层渠道增强
- `references/layer3-skills.md`：第三层 Skills 推荐（可执行版）
- `references/layer4-onboarding.md`：第四层新增渠道接入（双阶段）
- `references/troubleshooting.md`：重启验证与故障排查
- `openclaw插件开发记录.md`：本总结文档

---

## 今日归档（2026-02-28）

### 已完成
- 将 `references/configs.md` 重构为“1 个总览 + 5 个子文件”，降低单文件复杂度。
- 第三层 Skills 推荐已改为可执行：支持点名即安装，且默认聚合抓取 awesome 双仓并输出精选摘要。
- 调整 Discord 渠道增强提问顺序：`6 免 @`、`7 审批转发`，并移除“多 Agent”项（避免与第 4 轮新增渠道语义重叠）。
- 优化 Discord `guildId` 获取逻辑：
  - 默认路径改为 `channelId -> GET /channels/<channelId> -> guild_id`；
  - 成功后直接写入，不再先要求用户手工提供服务器 ID；
  - 仅在 DM/取不到 `channelId`/API 失败时回退询问。
- 补充 Discord `guildId` 获取“自动化指引 + 反查模板”：
  - 将 guildId 获取流程升级为“固定 6 步 + 禁止项（禁止偏航命令）”，要求低参数模型按顺序执行，不允许自由尝试。
  - 增加 `channelId` 提取优先级、Token 来源检查、API 正查/反查命令与 `401/403/404` 分流；
  - 目标是降低低参数模型在“当前频道可见但无法拿 serverId”场景下的失败率。
- 交互流程升级为“前 3 轮优化 + 第 4 轮新增渠道接入”：
  - 第 1 轮渠道识别改为单选；
  - 支持 `tui`（无渠道）分支直达第 4 轮接入引导；
  - Discord 渠道增强收敛为“免 @ → 审批转发”两项。
- 第 4 轮新增渠道接入补全为“双阶段”：
  - 阶段 A：用户先去各平台完成外部配置，按模板回传必要信息；
  - 阶段 B：AI 再执行本地 `openclaw.json` 对接与探针验证；
  - 三渠道键位统一为：Discord(`token/accounts`)、Telegram(`botToken`)、Feishu(`appId/appSecret`)。
- Feishu 新增接入细化为“两段 12 步”：
  - 先让用户完成前 1-7 步并回传 `App ID/App Secret`；
  - 再由 AI 执行 Feishu 插件安装/启用、本地配置和一次前置连接验证；
  - 连接成功后再引导用户完成后 8-12 步并回传配对码。
  - Feishu 插件安装方式统一为：`openclaw plugins install @openclaw/feishu`（npm）。
- 三渠道新增接入补齐“配对码闭环”：
  - 用户回传配对码后，优先执行 `openclaw pairing approve <channel> <code>`（仅多账号冲突时再补 `--account`）。
- 强化“重启成功”判定：
  - 要求 `PID before/after` 对比；
  - 日志关键字区分“Gateway 全量重启”与“仅 Discord 频道重连”；
  - 若出现 `Command still running`，必须继续轮询完成后再宣告成功。
- 按 `backup-unoptimized-state-20260227-204752.zip` 完成一次未优化基线回滚，并验证哈希一致、Gateway 可用。
- 复盘本轮测试日志，确认本次确实发生了 Gateway 重启；并确认 `guild_id` 来源为 Discord channels API 查询结果。

### 当前状态
- 三层流程可用，第二层 Discord 分支交互已降压（仅保留高频两项）。
- 第三层默认直接展示推荐清单，并保留“跳过第三层”一键跳过选项；用户点名则立即安装执行。
- 运行策略为“先实测体感，再做最小必要校验”，暂不强推额外自动化回归脚本。
- 第 5 项“安全收紧”前置认知已确认；Discord / Telegram 审批联动已写入执行规则，待补跨渠道实测记录。
- 第 4 项“联网搜索”已收敛为：SearXNG 本地优先（官方安装文档引导）+ `searx.space` 公共实例备用 + `browser.defaultProfile="openclaw"` 隔离浏览器兜底。

### 下次可直接续做
- 补齐 Telegram 审批联动端到端实测记录（含 `/approve` 成功与失败分流）。
- 持续维护第三层 Skills 推荐清单（当前 6 个条目，含 2 个 awesome 聚合数据源）。
- 细化联网搜索可写配置键和 Feishu 自动验收项。

---

## 使用建议（新一轮测试）

1. 先从未优化基线开始跑一轮 skill。
2. 验证第 1/2/3 轮是否符合三层优化流程（第三层可跳过）；若开启第 4 轮，再验证新增渠道接入分支。
3. 重点验证：
- 敏感命令是否触发 `/approve`
- Discord 的 DM/频道审批是否都能触发
- Discord 免 @ 在目标服务器是否生效
- Feishu 缓存优化是否按条件执行
4. 每轮测试后保存快照 zip（建议带状态标签）。

---

## TODO（待完成）

1. Telegram 审批联动：补齐端到端实测结论（`pairing -> 审批消息 -> /approve`）。
2. Skills 推荐层：维护当前 6 个条目（含 2 个 awesome 聚合数据源），持续优化“先摘要后展开”的推荐质量。
3. Skills 推荐层实测：补齐 Youtube Clipper / TTS / RedBookSkills / 大黑AI日报 的端到端安装与验收记录。
4. 大黑AI日报实测：验证固定时刻调度（`00:05` 起每 4 小时）在目标环境持续稳定触发。
5. 联网搜索配置：补一条“本地 SearXNG 连通性自动验收”的快捷检查模板（便于每轮测试复用）。
6. Feishu 24h 缓存优化：补一条“已改造成功”的自动验收检查项（便于测试时快速确认）。
7. 渠道审批联动补齐：按需为 Discord / Feishu / Telegram 配置 `execApprovals`，并完成 `/approve` 实测闭环。
8. （可选）回归测试脚本：如后续测试规模扩大，再补自动化检查。

---

## 简要里程碑（过程摘要）

项目早期经历过提问轮次与分层方式的多次调整，也经历过审批“可转发但不拦截”的排查。当前方案已收敛为“三层优化 + 第四轮接入扩展”，核心链路包括安全收紧、Discord 渠道增强、新增渠道接入与回滚恢复流程，均已有实战验证与文档化。
