# easy-openclaw：OpenClaw 小白友好配置向导

如果你刚开始用 OpenClaw，觉得配置麻烦、功能太多看不懂，这个项目就是给你用的。

它把复杂配置变成“对话选择题”：你只要回答开/关，OpenClaw 就能帮你把常用优化一次性配好。

## 这个项目能帮你解决什么

- 不想手改配置文件，也不想记命令。
- 不想每个功能单独折腾，想一次搞定。
- 想开安全审批，但怕配错。
- 想接 Discord / Telegram / Feishu，但不知道从哪一步开始。
- 想装一些实用 Skills，但不想自己到处找教程。

## 一共 4 层能力（按顺序）

1. 基础体验优化
- 流式输出（回答边生成边显示）
- 记忆功能（记忆增强；可选每天归档）
- 消息回执（已读反馈）
- 联网搜索（`defuddle` 优先，`r.jina.ai` 备用）
- 权限模式（默认 coding，也可切 full / minimal）

2. 渠道专项优化
- Exec 高危操作审批（第 2 轮统一收集，`coding/full` 有效，默认建议关；开启后优先放行常见命令，尽量只拦真正高敏操作）
- Discord：频道免 @ 响应、审批按钮（可选，需先开启审批）
- Feishu：探测 24h 缓存优化（降低额度消耗）
- Telegram：本层可直接配置审批提示投递

3. Skills 推荐与安装
- 默认直接展示推荐清单
- 你点名要装哪个，AI 立刻执行安装
- 包含：OpenClaw Backup、Agent Reach、安全防御矩阵、Find Skills、Youtube Clipper、OpenClaw Medical Skills
- 同时直接展示两个生态仓库入口：`awesome-openclaw-usecases`、`awesome-openclaw-skills`

4. 新渠道接入引导
- 可以继续新增 Discord / Telegram / Feishu
- 按步骤引导你去平台后台拿必要信息
- 回传后自动写入并验证

## 你怎么用（3 步）

1. 对你的 OpenClaw 说：

> 从 `https://github.com/daheiai/easy-openclaw` 安装并启用这个 skill。

2. 跟着它回答问题（开/关，或按格式回复）。

3. 最后确认写入并重启 Gateway，配置就生效了。

## 新手建议

- 第一次先在测试环境跑一轮。
- 如果你要接 Discord / Telegram，确保：
  - OpenClaw 所在服务器可以访问对应平台
  - 你当前网络环境也可以访问对应平台

## 项目文件说明（想深入再看）

- `SKILL.md`：主流程规则（AI 按这个执行）
- `references/`：各层详细说明

## 项目参考

- OpenClaw：`https://github.com/openclaw-ai/openclaw`
- OpenClaw Backup：`https://clawhub.ai/alex3alex/openclaw-backup`
- Agent Reach：`https://github.com/Panniantong/Agent-Reach`
- 安全防御矩阵：`https://github.com/slowmist/openclaw-security-practice-guide`
- Find Skills：`https://clawhub.ai/JimLiuxinghai/find-skills`
- Youtube Clipper Skill：`https://github.com/op7418/Youtube-clipper-skill`
- OpenClaw Medical Skills：`https://github.com/FreedomIntelligence/OpenClaw-Medical-Skills`
- Awesome OpenClaw Usecases：`https://github.com/hesamsheikh/awesome-openclaw-usecases`
- Awesome OpenClaw Skills：`https://github.com/VoltAgent/awesome-openclaw-skills`

## 常见问题（FAQ）

1) **如果出现重大问题怎么办？**（比如程序直接崩溃、启动不起来、发消息不回等）

答：优先尝试运行 `openclaw doctor --fix` 看能否自动修复。实在不行就用备份恢复：本 Skill 在流程开始时会建议你做一次全盘备份，并明确告诉你备份文件的保存路径；解压覆盖后即可回到改动前的状态。

2) **如果我开了审批，结果 `cd` / `ls` 这样的命令都需要审批怎么办？**

答：这是典型的“审批策略写得过严/放行列表没写全”导致的现象（常见原因是 agent 没按 Skill 约定写入宽松 allowlist，或者把策略理解错了）。虽然文档里有明确规定，但确实仍有人反馈会遇到。

解决方案：
- 方案 A：先按提示审批扛过这一波，等流程跑完后，让它修改配置把审批关闭。
- 方案 B：按问题 1 的方式用备份重置回滚。

3) **这个 Skill 支持最新版吗？**

答：我目前测试通过了 `2.13` 和 `2.21` 以及 `3.8` 和 `3.12` 版；其中我自己测试了 20+ 遍，使用的模型为 `MiniMax M2.5`。因为这个 Skill 主要修改的是 OpenClaw 的配置文件，一般情况下配置格式不会频繁大改，我认为整体是通用的；如果未来出现兼容问题请联系我，我会继续维护。

4) **我应该给 OpenClaw 用什么模型？以及从哪里能找到便宜的 API 中转？**

答：我会优先推荐各家官方的 **Coding 套餐**（价格/额度可能随时调整，请以官方页面为准）。按我自己的使用体验：

- **MiniMax**：目前综合下来我最推荐。便宜档约 29/月（我基本 5 小时内能用满）；中档约 49/月，我反而没用满过。它家没有周限，量大便宜总体够用；缺点是“智商一般”，且不支持原生多模态。
- **GLM-5**：更聪明，但速度偏慢、价格也更贵，同样不支持多模态。我更建议在 Claude Code 这类编码场景用，而不是放在 OpenClaw 里当“全能通用助手”。
- **Kimi K2.5**：支持原生多模态，但价格贵且给的额度偏少。
- **Qwen 3.5 Plus**：支持原生多模态，首月 7.9、后续 20/40 价格也还可以。

如果你需要国外顶流模型：可以看看我整理的 180+ 中转站导航页 `https://api.daheiai.com/` 任君挑选。它们与我没有利益关系，我只是搜集；建议大家 **少充点、多次充**，没人能保证这个站现在好用、下个月也一定好用。

5) **如何获取各种最新的 AI 资讯？**

答：我做了一个每隔 4 小时更新的 AI 新闻监控站：`https://news.daheiai.com/`，同时开放了 RSS（你可以订阅，也可以让你的小龙虾直接去抓取）。

另外我还做了一个“各类 Agent 工具到底在更新啥”的监控站（Claude Code / Codex / OpenClaw / Opencode / Gemini CLI 等）：`https://news.daheiai.com/changelog.php`。有这方面需求欢迎查看。

6) **有群或者交流方式吗？**

答：有的。你可以点击 `https://daheiai.com` 查看我的更多项目，也可以在里面找到我们的交流群入口。
