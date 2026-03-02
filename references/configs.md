# OpenClaw 优化配置参考总览

本目录已按“总览 + 六个子文件”拆分，减少单文件过长带来的维护和检索成本。

## 目录结构

- 第 0 层（测试观测模式，可选）：`references/layer0-testing.md`
- 第一层（基础推荐层）：`references/layer1-base.md`
- 第二层（渠道增强层）：`references/layer2-channels.md`
- 第三层（Skills 推荐层）：`references/layer3-skills.md`
- 第四层（新增渠道接入）：`references/layer4-onboarding.md`
- 故障与验证（重启/排障）：`references/troubleshooting.md`

## 配置文件路径

- 主配置：`~/.openclaw/openclaw.json`
- 工作区：`~/.openclaw/workspace/`
- Agent 目录：`~/.openclaw/agents/`
- 飞书插件探测文件：`~/.openclaw/extensions/feishu/src/probe.ts`

## 分层与接入总览

- 基础推荐层（跨渠道通用）：流式消息、记忆功能、消息回执、联网搜索、安全收紧
- 第 0 层测试观测（可选）：用于测试阶段输出详细观测信息与验收证据
- 渠道增强层（按渠道启用）：
  - Discord：频道免 @ 响应、审批转发联动
  - Feishu：探测 24h 缓存
  - Telegram：审批联动（在 Telegram 内 `/approve`）
- Skills 推荐层：默认直接展示推荐清单（可一键跳过；点名即执行安装）
- 新增渠道接入层（第 4 轮）：平台侧步骤 + 本地配置落盘 + 配对码放行

## 交互节奏（当前版本）

- 第 0 层（可选）：测试观测模式
- Step -1：环境预检（轻量，只读）
- Step 0：备份/恢复
- 第 1 轮：当前渠道识别（单选）+ 基础推荐层
  - 当前渠道可选：`discord` / `feishu` / `telegram` / `tui`
  - 若 `tui`：跳过第 2/3 轮，直接进入第 4 轮
- 第 2 轮：渠道增强层（只问当前渠道）
- 第 3 轮：Skills 推荐层（直接展示清单，可一键跳过，点名即安装）
- 第 4 轮：新增渠道接入引导（可跳过）

| 功能 | 适用渠道 | 默认建议 |
|---|---|---|
| 流式消息 | 全部 | 开 |
| 记忆功能 | 全部 | 开 |
| 消息回执 | 全部 | 开 |
| 联网搜索 | 全部 | 开（SearXNG 本地优先 + openclaw 隔离浏览器兜底） |
| 安全收紧 | 全部 | 可选（更安全但交互更繁琐） |
| 频道免 @ 响应 | Discord | 按需 |
| Discord 审批转发联动 | Discord | 开（开启安全收紧时建议同步开） |
| 探测 24h 缓存 | Feishu | 开（在用 Feishu 时） |
| Telegram 审批联动 | Telegram | 按需（开启第 5 项时建议同步开） |
| Skills 推荐层 | 全部 | 展示清单（可一键跳过，点名即安装） |
| 新增渠道接入引导（第 4 轮） | 全部 | 按需 |

## 建议读取顺序

1. 若用户开启测试版输出，先看 `references/layer0-testing.md`
2. 先看 `references/layer1-base.md`（基础能力）
3. 再看 `references/layer2-channels.md`（按渠道处理）
4. 第 3 轮默认读取 `references/layer3-skills.md`
5. 如用户开启第 4 轮，再看 `references/layer4-onboarding.md`
6. 执行和验证阶段统一看 `references/troubleshooting.md`
