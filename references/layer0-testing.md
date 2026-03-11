# 第 0 层：测试观测模式（可选）

用于测试阶段输出更完整的“可观测信息”，方便定位问题与回归对比。

## 启用方式

- 用户明确提到：`测试版` / `测试模式` / `调试模式` / `输出观测信息` / `详细日志`。
- 启用后仅增强输出，不改变功能逻辑。

## 输出要求（固定）

每个执行阶段都输出以下 4 行：
- `阶段`：当前步骤名称
- `动作`：执行了什么命令/写入了哪些键
- `结果`：成功/失败 + 关键返回
- `下一步`：继续执行或修复建议

## 配置写入观测

每次写入 `~/.openclaw/openclaw.json` 后，至少输出：
- 写入键清单（点路径）
- 写入前值（仅相关键）
- 写入后值（仅相关键）

注意：
- `token`、`appSecret`、`apiKey` 等敏感值必须脱敏显示。
- 若写入失败，必须明确“未写入成功”，禁止宣告完成。

## 重启观测（兼容无 rg 环境）

优先使用以下命令输出前后 PID：

```bash
BEFORE_PID="$(openclaw status --all 2>/dev/null | grep -oE 'pid [0-9]+' | head -n1 | awk '{print $2}')"
openclaw gateway restart
sleep 2
AFTER_PID="$(openclaw status --all 2>/dev/null | grep -oE 'pid [0-9]+' | head -n1 | awk '{print $2}')"
echo "before=${BEFORE_PID:-none} after=${AFTER_PID:-none}"
```

判定：
- `before` 和 `after` 均存在且不同：重启成功。
- 否则继续查日志或 systemd 启动时间再判定。

若工具返回 `Command still running (session ...)`：
- 必须继续轮询直到拿到 exit code，再给结论。

## 功能验收观测

至少输出本轮已开启项的验收结论：
- 权限模式：`openclaw security audit --deep` + `openclaw approvals get --json`（若第 2 轮已开启审批）
- 渠道健康：`openclaw channels status --probe`
- 记忆功能：检查 `memoryFlush.enabled=true` 与 `softThresholdTokens=40000`（若已开启）
- 联网搜索：正文提取链路检查（`defuddle -> r.jina.ai -> browser`，若已开启）
- 审批联动：触发一次审批并回传 `/approve` 结果（若已开启）

## 收尾输出（测试版）

结束时固定输出：
- 本轮变更键总览
- 未完成项清单
- 失败命令与错误摘要
- 建议复测入口（从哪一步重试）
