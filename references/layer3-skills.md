# 第三层：Skills 推荐层（第 3 轮，可执行版）

本层用于在完成基础配置后，直接扩展可用能力。

## 执行规则（固定）

- 第 3 轮默认直接展示推荐清单，不再使用“11 开”子菜单。
- 清单末尾只保留一个选项：`跳过第三层`。
- 用户点名要安装某个 Skill 时，立即执行该 Skill 的安装流程，不再回到“仅推荐”。
- 每次真正发送第 3 轮清单前，必须重新读取本文件；不得凭记忆、不得沿用上一轮缓存。
- 展示推荐清单前，必须先读取本文件；默认推荐项只能来自本文件，禁止改用系统 `<available_skills>`、`openclaw skills list`、`find-skills` 或 `~/.npm-global/lib/node_modules/openclaw/skills/` 目录枚举结果。
- 第 3 轮发送前必须自检：若输出中出现 `blogwatcher`、`github`、`clawhub`、`weather` 等本文件未列出的系统 Skill，立即丢弃并按本文件重生一版，不得发送给用户。
- 发送时必须严格按本文件的编号顺序展示，不得替换、增删、改名、概括成“系统现有技能推荐”。
- 若用户回复编号列表（如 `1 2` / `1,2`），只能处理这些被明确选中的编号；禁止顺带安装未被选择的条目。
- 多个已选条目必须逐项回报状态，不能把未选择项混进“安装完成”清单。
- 用户一旦点名安装某个条目，即视为同意该条目的最小必要依赖补齐；不要再把 `pip/ffmpeg/yt-dlp` 这种基础依赖抛回给用户手动处理。
- 安装流程不得一刀切；必须按条目类型执行：
  - 仓库型 Skill：读取仓库 README 安装段落 -> 原样执行安装命令 -> 执行最小可用验证 -> 汇报结果。
- 若安装过程中缺少基础依赖（如 `pip`）：
  - 先按同一仓库 README / install 文档里的依赖步骤自动补齐
  - 文档未给出时，再尝试标准补齐（如 `python3 -m ensurepip --upgrade`，必要时 `get-pip.py`）
  - 禁止向用户索要“服务器密码 / 终端密码”
  - 只有在环境权限不足或补齐失败时，才把该项标记为“待人工处理”
- 全部已选条目处理完后，必须明确回到第 4 轮；若用户选择“跳过第四层”，应立即进入收尾确认，不能无响应停住。
- 对“仓库入口型条目”，默认只展示仓库链接与用途；用户明确要求“展开总结”再抓取并分类整理。

## 当前推荐清单（可执行）

### 1) OpenClaw Backup（可安装，用于后续备份管理）

- ClawHub：`https://clawhub.ai/alex3alex/openclaw-backup`
- 核心能力：备份与恢复 `~/.openclaw` 数据，适用于手动备份、自动备份调度、备份轮转、从备份恢复。
- 适用场景：用户已经完成初始配置，后续希望把“备份/恢复/轮转”长期交给 Skill 管理，而不是每次手动敲命令。
- 与第 1 层关系（强制）：
  - 第 1 层 Step 0 的一次性全量备份仍然保留，优先用于当前改配置前的即时回滚保护
  - `OpenClaw Backup` 属于后续增强能力，不替代 Step 0
- 执行动作（固定）：
  - 按 ClawHub 标准安装链路安装 `openclaw-backup`
  - 安装后做一次最小验证：查询它支持的备份/恢复动作，或执行一次“只创建单个测试备份、不覆盖现有配置”的安全测试
- 使用边界（强制）：
  - 默认优先做“创建新备份”验证，禁止一上来执行恢复覆盖
  - 若用户要启用自动备份，必须先明确备份输出目录与保留策略
  - 若备份文件落在 `~/.openclaw/` 内，必须确认其排除了旧备份目录，避免重复打包导致“滚雪球”

### 2) Agent Reach（可安装，需 exec 权限）

- 仓库：`https://github.com/Panniantong/Agent-Reach`
- 安装文档：`https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md`
- 核心能力：给 Agent 补齐互联网访问与平台读取能力，覆盖网页、YouTube、RSS、GitHub、Twitter/X、小红书、微博、微信公众号等，并内置 `agent-reach doctor` 诊断。
- 适用场景：用户希望让 OpenClaw 后续具备更强的“读网页/读视频/读社媒/搜全网”能力，而不想自己逐个折腾依赖和平台配置。
- 前置检查（强制）：
  - OpenClaw 场景下，先确认 `tools.profile` 不是 `minimal`，且具备 exec 执行能力
  - 至少确认 `python3` / `pip` 可用；若缺失，再进入上游安装文档指定的依赖补齐流程
  - 若用户要走安全模式，必须明确使用上游文档里的安全安装参数，而不是擅自放宽权限
- 安装顺序（固定）：
  - 第一步：确认 OpenClaw 已允许执行命令；若当前是 `minimal`，先提示切到 `coding`
  - 第二步：直接按上游安装文档执行：
    - `帮我安装 Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md`
  - 第三步：若用户要求安全模式，则改用上游文档里的 `--safe` 路线
  - 第四步：安装后执行一次最小验证：运行 `agent-reach doctor`，或验证其中一个免配置能力（如网页 / YouTube / RSS）可用
- 验收规则（固定）：
  - 只有当安装流程完成，且 `agent-reach doctor` 或最小能力测试可用时，才可宣告安装完成
  - 若失败，要明确区分是“exec 权限不足”“Python/pip 缺失”“上游依赖安装失败”还是“某个具体渠道未配置”
  - 不得把“某个平台还没配 Cookie / 代理”误报成“整个 Agent Reach 安装失败”
  - 若通过自动补齐已解决 `pip` 缺失，应直接继续安装；不要停在“请用户自己先装 pip”

### 3) Find Skills（可安装，用于后续扩展）

- ClawHub：`https://clawhub.ai/JimLiuxinghai/find-skills`
- 核心能力：当用户后续明确提出“帮我找某类 Skill / 有没有能做 X 的 Skill / 我还想扩展能力”时，帮助发现并安装合适的 Skill。
- 适用场景：用户已经完成当前基础配置，后续又想继续扩展功能，但不想自己手动翻仓库。
- 执行动作（固定）：
  - 按 ClawHub 标准安装链路安装 `find-skills`
  - 安装后做一次最小验证：让它回答“针对某个明确需求，推荐什么类型的 Skill”
- 使用边界（强制）：
  - 它只能用于“用户明确要求继续找更多 Skill”之后的扩展探索
  - 不得把它当作本层默认推荐清单的生成器
  - 不得用它覆盖本文件里的固定推荐项

### 4) Youtube Clipper Skill（可直接安装）

- 仓库：`https://github.com/op7418/Youtube-clipper-skill`
- 推荐场景：YouTube 内容快速剪辑、提取、再利用。
- 执行动作（固定）：按仓库 README 安装步骤执行，安装后做一次最小链路验证。

### 5) OpenClaw Medical Skills（仓库入口，医学方向）

- 仓库：`https://github.com/FreedomIntelligence/OpenClaw-Medical-Skills`
- 核心能力：提供大规模医学与生物科研技能库，覆盖临床、基因组学、药物发现、生物信息学、医疗器械等方向。
- 适用场景：用户本身就是医学、生命科学、临床研究相关使用者，想把 OpenClaw 扩展成领域型研究助手。
- 安装方式（固定）：
  - 优先按上游 README 的 OpenClaw 路线安装
  - 推荐方式是 clone 仓库后，把 `skills/` 下所需技能复制到 `<workspace>/skills/`，或通过 `skills.load.extraDirs` 挂载整个目录
  - 若只是试用，优先建议“精选子集安装”，不要默认把整库全量复制进当前 workspace
- 验收规则（固定）：
  - 安装后让 agent 回答“你现在有哪些医学/临床相关 skills”
  - 若用户只安装了部分子集，汇报时必须明确“已安装的是子集，不是整库”

### 6) Awesome OpenClaw Usecases（仓库入口）

- 仓库：`https://github.com/hesamsheikh/awesome-openclaw-usecases`
- 用途：查看别人怎么实际使用 OpenClaw、有哪些可复用场景与工作流。
- 默认动作：仅展示仓库链接和用途。
- 若用户明确要求“展开总结”，再抓取并分类整理典型用例。

### 7) Awesome OpenClaw Skills（仓库入口）

- 仓库：`https://github.com/VoltAgent/awesome-openclaw-skills`
- 用途：查看更广泛的 Skills 清单与生态入口，适合继续挑选其他可安装技能。
- 默认动作：仅展示仓库链接和用途。
- 若用户明确要求“展开总结”，再抓取并分类整理候选技能。
