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
- 记忆功能（上下文更连贯）
- 消息回执（已读反馈）
- 联网搜索（`defuddle` 优先，`r.jina.ai` 备用）
- 权限模式（默认 coding，也可切 full / minimal）

2. 渠道专项优化
- Discord：频道免 @ 响应、审批联动
- Feishu：探测 24h 缓存优化（降低额度消耗）
- Telegram：审批联动（支持 `/approve`）

3. Skills 推荐与安装
- 默认直接展示推荐清单
- 你点名要装哪个，AI 立刻执行安装
- 包含：Youtube Clipper、TTS、RedBookSkills、大黑AI日报
- 额外提供“更多 Skills 扩展入口”（两个 awesome 仓库链接）

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
- 大黑AI日报安装后，先手动发一条测试消息确认可达，再启用定时推送。

## 项目文件说明（想深入再看）

- `SKILL.md`：主流程规则（AI 按这个执行）
- `references/`：各层详细说明
- `openclaw插件开发记录.md`：开发记录和待办

## 当前进度

主流程已可用，适合直接测试。

当前主要剩余项是实测闭环（Telegram 审批、TTS/RedBook 实装验收、Feishu 缓存验收、三渠道审批闭环）。
