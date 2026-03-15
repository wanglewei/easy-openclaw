# OpenClaw 优化配置参考总览

本目录已按“总览 + 八个子文件”拆分，减少单文件过长带来的维护和检索成本。

## 目录结构

- 第 0 层（测试观测模式，可选）：`references/layer0-testing.md`
- 第一层（基础推荐层）：`references/layer1-base.md`
- 第二层（渠道增强层）：`references/layer2-channels.md`
- 第三层（Skills 推荐层）：`references/layer3-skills.md`
- 第四层（新增渠道接入）：`references/layer4-onboarding.md`
- 条件补充提问：`references/collection.md`
- 统一执行与收口：`references/execution.md`
- 故障与验证（重启/排障）：`references/troubleshooting.md`

## 配置文件路径

- 主配置：`~/.openclaw/openclaw.json`
- 工作区：`~/.openclaw/workspace/`
- Agent 目录：`~/.openclaw/agents/`
- 飞书插件探测文件：`~/.openclaw/extensions/feishu/src/probe.ts`

注意（Windows / WSL / 多环境常见误区）：
- 本仓库默认假设 OpenClaw 的“数据目录”（配置/工作区/agents）在用户主目录下的 `~/.openclaw/`。
- 在 Windows 上，很多人是通过 **WSL Ubuntu** 安装并运行 OpenClaw：此时 `~/.openclaw/` 位于 Linux 的 home（如 `/home/<user>/.openclaw`），但其底层磁盘文件（WSL 的 `ext4.vhdx`）经常仍然落在 Windows 的 `C:`，所以会产生“我明明装在 D 盘，为何数据在 C 盘”的错觉。
- 真正会导致“需要手工合并配置”的常见原因是：**安装/启用 Skill 的终端环境**与**运行 Gateway 的环境**不一致（例如 Windows PowerShell 装、WSL 里跑；或两个 WSL distro/两个 Linux 用户混用），最终改动写进了另一份 `~/.openclaw/`。

## 分层与接入总览

- 基础推荐层（跨渠道通用）：流式消息、记忆功能、消息回执、联网搜索、权限模式
- 第 0 层测试观测（可选）：用于测试阶段输出详细观测信息与验收证据
- 渠道增强层（按渠道启用）：
  - 跨渠道：Exec 高危操作审批（仅 `coding/full` 有效，默认建议关；开启后采用宽 allowlist，避免常见命令频繁审批）
  - Discord：频道免 @ 响应、审批按钮（可选，需先开启审批）
  - Feishu：探测 24h 缓存
  - Telegram：可在本层直接配置审批提示投递
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
| 联网搜索 | 全部 | 开（defuddle 优先 + r.jina.ai 备用 + openclaw 隔离浏览器兜底） |
| 权限模式 | 全部 | 可选（强烈建议保持 coding；minimal=纯聊天机器人；full=完全放开） |
| Exec 高危操作审批 | 全部 | 按需（第 2 轮收集；默认建议关；仅 coding/full 有效：宽 allowlist + ask=on-miss，尽量只让少见/高敏命令弹审批；minimal 下 exec 往往不可用） |
| 审批提示投递 | Discord/Telegram 等 | 按需（`approvals.exec` 的 `mode=session/targets/both` + `targets` 一次性配置） |
| 频道免 @ 响应 | Discord | 按需 |
| Discord 审批按钮 | Discord | 按需（仅在第 2 轮已开启审批后才有意义） |
| 探测 24h 缓存 | Feishu | 开（在用 Feishu 时） |
| Skills 推荐层 | 全部 | 展示清单（可一键跳过，点名即安装） |
| 新增渠道接入引导（第 4 轮） | 全部 | 按需 |

## 建议读取顺序

1. 若用户开启测试版输出，先看 `references/layer0-testing.md`
2. 第 1 轮前看 `references/layer1-base.md`
3. 第 2 轮前看 `references/layer2-channels.md`
4. 第 3 轮前看 `references/layer3-skills.md`
5. 第 4 轮前看 `references/layer4-onboarding.md`
6. 四轮结束后、统一确认前看 `references/collection.md`
7. 用户确认执行后看 `references/execution.md`
8. 出现异常或需要核验时看 `references/troubleshooting.md`
