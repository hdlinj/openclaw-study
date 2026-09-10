# 2026-09-08 OpenClaw `/upgrade` 失败 + service 22 小时无人拉起复盘

> **TL;DR**：098 端用户在飞书发 `/upgrade` → web_search + docs fetch 双失败 + feishu streaming timeout → 实际升级未进行但导致 gateway SIGTERM → drop-in `Restart=on-failure` 不触发 → service dead 22h（9-08 20:27 → 9-09 15:42）→ 思科瑞 9-09 救活 + 真实升级 9.3 成功

## 时间线（journal 实锤）

### 9-08 升级触发阶段（10:09 ~ 10:23）

| 时间            | 事件                                                                                                                  | 类型  |
| ------------- | ------------------------------------------------------------------------------------------------------------------- | --- |
| 09:11 ~ 10:02 | minimax provider rate_limit 持续（升级**前**已有压力）                                                                         | 背景  |
| 10:09:59      | feishu streaming card fetch-timeout 30000ms（`open.feishu.cn/open-apis/cardkit/...`）                                 | ❌   |
| 10:15:49      | gateway `cron: job added`                                                                                           | 中性  |
| 10:17:39      | gateway `cron: job updated`                                                                                         | 中性  |
| 10:19:26      | `web_search failed: web_search is disabled or no provider is available` ×2（搜 9.2 release notes / 7.1→9.2 migration） | ❌❌  |
| 10:19:48      | 用户发 `/upgrade` chat 命令                                                                                              | 触发  |
| 10:19:48      | `web_fetch docs.openclaw.ai/upgrade` 被外层 `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="52f4d42989de6c70">>>` 截断            | ❌   |
| 10:23:33      | feishu Update TimeoutError 再次                                                                                       | ❌   |

### 9-08 多次 SIGTERM + restart-recovery 失败阶段（15:16 ~ 20:27）

| 时间          | 事件                                                                                                                     |
| ----------- | ---------------------------------------------------------------------------------------------------------------------- |
| 15:16:00    | gateway `signal SIGTERM received`                                                                                      |
| 15:16:01    | `Stopped openclaw-gateway.service` (v2026.7.1-2)                                                                       |
| 20:12:40    | `Started openclaw-gateway.service`（持续 5h 无人拉，5h 后被拉起）                                                                  |
| 20:12:51    | `marked 1 startup-orphaned main session`                                                                               |
| 20:12:58    | **`marked interrupted main session failed: agent:main:feishu:direct:ou_7dc85b... (transcript tail is not resumable)`** |
| 20:15:33    | 又一次 SIGTERM                                                                                                            |
| 20:22:59    | 又一次 Started                                                                                                            |
| 20:23:06-12 | 又一次 restart-recovery failed                                                                                            |
| 20:27:08    | 又一次 SIGTERM                                                                                                            |
| 20:27:09    | **`Stopped openclaw-gateway.service` — service dead 进入无人拉起状态**                                                         |

### 9-09 救活 + 真升级阶段

- **15:42** 思科瑞098 `systemctl start openclaw-gateway` → service active
- **16:54** 我第二次拉起（drop-in 修复后）+ 真升级包替换完成（version 9.3）
- **17:09-17:18** dispatch 修复（`bindings` 加好）

---

## 多层根因（4 层叠加）

### ❶ Layer 1: `web_search` 工具未配置

**journal 实锤**：

```
[tools] web_search failed: web_search is disabled or no provider is available.
raw_params={"query":"OpenClaw 2026.9.2 release notes upgrade guide breaking changes","count":8}
[tools] web_search failed: web_search is disabled or no provider is available.
raw_params={"query":"OpenClaw 7.1 to 9.2 migration feishu channel plugin","count":8}
```

**配置层面**：

- `098 openclaw.json` 没有 `web_search` / `search_provider` 配置块
- `plugins.allow` 里有 `tavily`，但 tavily plugin 没启用或 `TAVILY_API_KEY` 没注入
- OpenClaw 9.3 `web_search` 是**内置 tool**（不是 plugin 提供的），调用时若没有 provider 配置直接 fail

**影响**：升级前调研 release notes / migration guide **完全失败**——根本不知道 9.3 schema 改了 `bindings[]` 嵌套格式、9.3 的 cli/update.md 怎么用

### ❷ Layer 2: `docs.openclaw.ai/upgrade` fetch 被截断

**journal 实锤**：

```
[10:19:48]  /upgrade
[10:19:48] <<<END_EXTERNAL_UNTRUSTED_CONTENT id="52f4d42989de6c70">>> raw_params={"url":"https://docs.openclaw.ai/upgrade","maxChars":10000}
```

**机制**：OpenClaw 把外部 fetch 内容视为 **untrusted**（防止 prompt injection），用 sentinel 包裹，**完整内容不进入 LLM context**。

**影响**：就算 web_search 不 fail，agent 也看不到升级文档的**完整**内容（只能看到 sentinel 标记的元数据）

### ❸ Layer 3: feishu streaming card 30s timeout

**journal 实锤**：

```
[10:09:59] [feishu] feishu[default] Update failed: TimeoutError: request timed out
[fetch-timeout] fetch timeout after 30000ms (elapsed 30000ms) operation=fetchWithSsrFGuard 
url=https://open.feishu.cn/open-apis/cardkit/v1/cards/7682980560591965172/elements/content/content
[10:23:33] [feishu] feishu[default] Update failed: TimeoutError: request timed out
```

**机制**：

- agent 回复飞书消息时用 streaming card 持续更新进度
- feishu cardkit API 30s 超时（`fetch-timeout after 30000ms`）
- 升级过程长（>30s），card 更新失败
- 流式关闭后**不能再 update**（`streaming mode is closed; (code=300309)`）

**影响**：

- 用户以为升级没成功
- agent 不知道升级实际进度（看不到自己的回复状态）
- 反复重试加重失败

### 🚨 Layer 4 (最致命): drop-in `Restart=on-failure` 不触发

**9-08 现象**：

```bash
$ systemctl --user show openclaw-gateway --property Restart,ExecMainStatus
Restart=on-failure      ← drop-in 覆盖
ExecMainStatus=0         ← exit 0 干净退出
```

**journal 实锤**：20:27 SIGTERM → 20:27 Stopped → **之后 22 小时无人拉起**

**根因**：

- 098 端 drop-in `restart-policy.conf`：
  
  ```ini
  [Service]
  Restart=on-failure   ← 7-28 改的，平时不暴露
  RestartSec=5
  ```

- 升级过程（无论 `/upgrade` 成功还是失败）OpenClaw 主动 stop service，**exit code 0** = clean shutdown

- `Restart=on-failure` 只在 exit code ≠ 0 时拉起

- exit 0 → on-failure **不触发** → service dead

**升级文档（cli/update.md 第 N 段）原话**：

> "Only `activating` stops the managed service. ... In `verifying`, the updater checks that the managed service is running and owns its port"

—— 但 service 启动后能否被 systemd 接住，**依赖 drop-in Restart 策略**。这是 OpenClaw 文档**没强调的盲点**。

---

## 教训（5 条）

### 1. **/upgrade 是 chat 内置命令，不是 CLI**

- 触发方式：用户在飞书发 `/upgrade`，gateway 内部走 `chat-commands` 处理
- 行为：自动调用 `web_search` + `web_fetch docs.openclaw.ai/upgrade` 调研 + 自动发起 `openclaw update`
- **风险**：调研失败 = 升级"瞎跑"

### 2. **web_search / web_fetch 必须先 verify**

- 升级前必跑：`openclaw` 内置 web_search 是否能用
- 没有 provider 配置 = `web_search failed` = 调研失败 = 升级无依据
- 配置层面：`plugins.allow` 含 tavily ≠ tavily 可用，还需要 `TAVILY_API_KEY` + plugin enabled

### 3. **降级机制（drop-in）有"安静失败"模式**

- drop-in `Restart=on-failure` 平时（service crash 时）看起来正常
- 升级时 service **clean shutdown** (exit 0) = 降级机制**不触发**
- **预演测试**：手动 `systemctl stop openclaw-gateway` → 看是否自动拉起 → 如果不拉 = drop-in 坑

### 4. **feishu streaming card 30s timeout 是升级的死结**

- 升级过程长（>30s），card 更新必超时
- 升级消息发不出来 = 用户不知进度 = 反复发指令
- **改进方向**：升级前用非流式消息（"升级开始，预计 N 分钟"），升级中不更新 card

### 5. **service dead 不一定立即发现**

- 9-08 20:27 SIGTERM → 9-09 10:04 我察觉（中间 14h 沉默）
- 没有任何 cron / watchdog 检测"service active"状态
- **改进**：加 watchdog cron（`*/5 * * * * systemctl is-active` + 不 active 立即告警）

---

## 修复（9-09 实战）

| #   | 动作                                           | 结果             |
| --- | -------------------------------------------- | -------------- |
| 1   | `systemctl --user start openclaw-gateway`    | service active |
| 2   | drop-in 改 `Restart=always` + `daemon-reload` | ✅ 永不丢服务        |
| 3   | 187 端真升级到 9.3（包替换 + bindings schema 修复）      | ✅ 完成           |
| 4   | cron `*/30 * * * * status-report-098.sh`     | ✅ 状态同步         |
| 5   | 098 + 187 双端 A2A 协议实战                        | ✅ 实时业务交互       |

详细记录：`memory/2026-09-09.md` + `MEMORY.md` 9-09 段落

---

## 预防措施（已实施 / 待实施）

### ✅ 已实施

1. drop-in `Restart=always`（防下次升级再踩）
2. cron `*/30 * * * * status-report-098.sh`（098 状态同步到 187）
3. 098 + 187 双端 A2A 协议（实时业务交互首选）

### ⏳ 待实施（优先级排序）

| 优先级 | 措施                                                                                                     | 理由                          |
| --- | ------------------------------------------------------------------------------------------------------ | --------------------------- |
| P0  | 098 openclaw.json 加 web_search provider（如 tavily plugin enabled + API key）                             | 下次升级需要调研 release notes      |
| P0  | watchdog cron `*/5 * * * * systemctl is-active openclaw-gateway`                                       | service dead 立即告警，不再 14h 沉默 |
| P1  | 升级前 verify 脚本（`upgrade_pre_check.sh`）：检查 drop-in Restart / quota cap / 备份 / web_search provider        | 机制分层原则：只针对升级这一场景            |
| P1  | 升级失败自动 fallback：`openclaw.json.last-good` 备份 + 检测到 `AgentSelectionRequiredError` 等 schema 错自动 rollback | 减少人工救活时间                    |
| P2  | `/upgrade` 命令禁用或加确认（强制走 `openclaw update wizard` 交互式）                                                  | 防避免自动化升级"瞎跑"                |

---

## 排查命令集（升级前 + 升级失败后）

```bash
# ===== 升级前 verify =====
# 1. web_search provider 是否可用
grep -E "tavily|web_search|search_provider" /home/lin/.openclaw/openclaw.json

# 2. drop-in Restart 策略（必须 Restart=always 或 RestartPreventExitStatus 包含 0）
systemctl --user show openclaw-gateway --property Restart,RestartSec,RestartPreventExitStatus,SuccessExitStatus

# 3. quota cap 状态（不在 cap 期升级）
journalctl --user -u openclaw-gateway --since "5 hours ago" | grep -cE "2067|rate_limit_error"

# 4. 备份完整（必须含 openclaw.json 且 mtime < 24h）
tar -tzf /home/lin/.openclaw/workspace/data/backups/$(ls -t /home/lin/.openclaw/workspace/data/backups/*.tar.gz.gpg | head -1) | grep openclaw.json

# 5. 当前 OpenClaw 版本 + update channel
npm ls -g openclaw --depth=0
openclaw update status

# ===== 升级失败后 troubleshoot =====
# 6. 9-08 失败的 4 个症状日志
journalctl --user -u openclaw-gateway --since "1 day ago" \
  | grep -E "web_search failed|END_EXTERNAL_UNTRUSTED|fetch-timeout|Update failed: TimeoutError|SIGTERM" | head -20

# 7. restart-recovery 失败（session 不可恢复）
journalctl --user -u openclaw-gateway --since "1 day ago" | grep "restart-recovery.*failed"

# 8. 当前 service 真实状态
systemctl --user is-active openclaw-gateway
systemctl --user show openclaw-gateway --property ActiveEnterTimestamp,SubState,Result

# 9. drop-in 文件被改没
grep -rE "Restart=" /home/lin/.config/systemd/user/openclaw-gateway.service.d/

# 10. 拉起 gateway（手动）
systemctl --user reset-failed openclaw-gateway
systemctl --user start openclaw-gateway
```

---

## 跟其他 incident 的关系

- **8-03 飞书 5min cap**：同是 OpenClaw 升级相关，但不是同类型问题（5min cap 是飞书 sequential queue，不是升级机制）
- **8-11 memories 撒谎**：同是 patch 持久化问题（disk patch 没真生效）
- **8-24 协作新规**：同是 service 故障发现延迟（但8-24 是 LLM token unusable，不是 service dead）
- **9-09 升级完整实战**：本次 incident 的**修复版本**，闭环

---

**关键认知**：OpenClaw 升级是**分布式系统升级**——涉及 systemd service / drop-in 配置 / npm 包 / openclaw.json / feishu streaming / 飞书 schema / 文档调研 / web_search 工具 / cron 任务，任何一环没准备好都会失败。

**通用心法**：升级前 = 3 grep（drop-in Restart / quota cap / web_search provider），升级中 = journal 实时跟踪，升级后 = verify 5 项（service active / feishu ready / cron 在 / 套餐恢复 / dispatch OK）。