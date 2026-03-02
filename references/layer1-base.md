# 第一层：基础推荐层（第 1 轮）

## 备份

备份目标：
- `~/.openclaw/openclaw.json`
- `~/.openclaw/workspace/`
- `~/.openclaw/agents/`

命名规则：`backup-YYYYMMDD-HHMMSS.zip`

执行命令：
```bash
cd ~/.openclaw && zip -r backup-$(date +%Y%m%d-%H%M%S).zip openclaw.json workspace/ agents/
```

## 1) 流式消息

推荐写法：
```json
"channels": {
  "discord": {
    "streaming": "partial"
  }
}
```

合法值：
- `"off"`
- `"partial"`（推荐）
- `"block"`
- `"progress"`

可配合：
```json
"agents": {
  "defaults": {
    "blockStreamingDefault": "on",
    "blockStreamingBreak": "text_end"
  }
}
```

## 2) 记忆功能（本地 embedding）

```json
"agents": {
  "defaults": {
    "compaction": {
      "memoryFlush": {
        "enabled": true,
        "softThresholdTokens": 40000
      }
    },
    "memorySearch": {
      "enabled": true,
      "provider": "local",
      "local": {
        "modelPath": "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf"
      },
      "fallback": "none"
    }
  }
}
```

注意：
- `compaction` 必须深度合并，保留原有字段。
- 首次执行 memory index/search 时会自动下载约 329MB 本地模型文件。

## 3) 消息回执

```json
"messages": {
  "ackReactionScope": "all"
}
```

## 4) 联网搜索

目标：不给用户强绑 Brave API，优先使用本地可控搜索，再用隔离浏览器兜底抓取页面内容。

推荐策略（按优先级）：
1) 本地 SearXNG（推荐，开箱后免费可用）
2) `searx.space` 公共实例（本地安装失败/暂不想部署时）
3) 浏览器隔离抓取兜底（`browser.defaultProfile="openclaw"`）

官方安装文档：
- `https://docs.searxng.org/admin/installation-searxng.html`

公共实例入口（备用）：
- `https://searx.space/`
- 说明：页面由 JS 渲染实例列表，优先选择在线、延迟低、稳定的 HTTPS 实例

`openclaw.json` 建议写入（浏览器兜底）：

```json
"browser": {
  "enabled": true,
  "defaultProfile": "openclaw"
}
```

说明：
- `defaultProfile: "chrome"` 会走扩展 relay 和用户日常浏览器上下文，不适合作为默认自动化通道。
- `defaultProfile: "openclaw"` 使用 OpenClaw 托管隔离 profile，更安全也更符合开箱路径。

写入 `~/.openclaw/workspace/TOOLS.md`（告知 agent 的操作偏好）：

```markdown
## 搜索服务（默认）
- 优先使用 SearXNG，不用 Brave API 或其他付费接口。
- 默认地址：`http://127.0.0.1:8081/search?q=关键词&format=json`
- 若本地实例不可用，改用 `https://searx.space/` 中可用实例，替换基础地址后再检索。
- 工作流：先检索（SearXNG JSON）-> 再整理答案；必要时再用 `browser`（profile=`openclaw`）打开网页并 snapshot。
```

本地部署注意（强制检查）：
- 推荐端口映射：`-p 8081:8080`（避免宿主机 `8080` 已被其他服务占用）
- 若 `format=json` 返回 403，而不带 `format` 的页面返回 200：通常是 `search.formats` 未包含 `json`
- 需要确保 SearXNG 的 `settings.yml` 中 `search.formats` 至少包含 `html` 和 `json`

SearXNG 连通性检查：

```bash
curl -s "http://127.0.0.1:8081/search?q=openclaw&format=json" | jq '.results | length'
```

若使用公共实例，将地址替换为对应实例 URL（保留 `search?...&format=json` 结构）。

## 5) 安全收紧（可选）

已实测可生效的“收紧版”至少需要两层：`openclaw.json` 执行策略 + `exec-approvals.json` 默认策略。  
若希望在聊天渠道内接收审批并 `/approve`，需在第二层开启对应渠道联动（Discord 第 7 项 / Telegram 第 9 项）。

`openclaw.json`（核心约束）：

```json
"approvals": {
  "exec": {
    "enabled": true,
    "mode": "session",
    "sessionFilter": ["discord", "telegram", "feishu"]
  }
},
"tools": {
  "elevated": {
    "enabled": false
  },
  "exec": {
    "host": "gateway",
    "security": "allowlist",
    "ask": "always"
  }
},
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "off"
    }
  }
}
```

`exec-approvals.json` 默认策略：

```json
{
  "version": 1,
  "socket": {},
  "defaults": {
    "security": "allowlist",
    "ask": "always",
    "askFallback": "deny"
  },
  "agents": {}
}
```

推荐命令（避免手写 JSON 出错）：

```bash
printf '%s' '{"version":1,"socket":{},"defaults":{"security":"allowlist","ask":"always","askFallback":"deny"},"agents":{}}' | openclaw approvals set --stdin --json
```

审批命令：
- `/approve <id> allow-once`
- `/approve <id> allow-always`
- `/approve <id> deny`

注意：
- OpenClaw `2026.2.24` 下，`tools.exec.askFallback` 不是合法键，`askFallback` 要放在 `exec-approvals.json`。
- 若设置 `agents.defaults.sandbox.mode: "all"` 但系统未安装 Docker，会报 `spawn docker ENOENT`；此时改回 `"off"` 并重启 Gateway。
