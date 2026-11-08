# 2026-11-08｜K 线库日更接管 + 开机自启等效验证（交付记录）

> 本仓为派活桥仓库；实际运行物在部署机本地（`C:\Users\hezheng\share_deploy\`）。本文件记录本次运维改动点与验证证据，便于交接与回滚。

## 0. 结论

- **任务一 · K 线库日更：✅ 已接管**。新增日更脚本 + Windows 计划任务 `\KlinesDailyUpdate`（周一~周五 15:50，交易日收盘后更新 `klines.db` 并在飞书群回报一行）；今日（2026-11-08 周日，非交易日）生产库零写入；**首次真实日更为 2026-11-09 15:50**。
- **任务二 · 开机自启：✅ 等效验证 PASS**。进程全清后，按开机启动项原样拉起 `guard / bridge / 8900(+watchdog)` 全部恢复；claim 自测回群 `claim #28`；桥重启后真人消息亦正常 claim（`claim #29`）。
- **未做真实冷启动重启**（用户显式允许「真实重启或等效」二选一）：真实重启会中断承载本次上报链路的 DSH 桌面会话，故采用等效方法，并如实记录其覆盖范围与残留风险（见 §3.4）。

## 1. 边界

- 只动 `klines.db` 与部署目录；未碰持仓快照、成交记录、飞书多维表格；**未删除任何文件**；未改本仓库代码。
- 任务一按边界**只写 `klines.db.kline` 单表**（不写 `stock_data.db.daily_quote`；历史采集器是双写，本次未沿用该双写）。

## 2. 任务一：K 线库日更

### 2.1 交付物

| 项 | 内容 |
|---|---|
| 脚本 | `C:\Users\hezheng\share_deploy\klines_daily_update.py`（纯 stdlib，pythonw 可运行；SHA256 `F18311C57AFBAC71F1B8EEC7C5C61BA5AA8C9AB44CFE3DBB486E84437DA85D02`） |
| 计划任务 | `\KlinesDailyUpdate`；Weekly 周一~周五 `15:50`；仅登录时运行（InteractiveToken）；StartWhenAvailable；Hidden；MultipleInstances=IgnoreNew；执行时限 2h |
| 下次运行 | **2026-11-09 15:50:00**（Status=Ready，Last Result=0） |
| 日志 | `C:\Users\hezheng\share_deploy\klines_daily_update.log`；证据目录 `share_deploy\kline_daily\` |

### 2.2 行为

- 交易日判定：`trading_days_2026.json` 日历；交易日 15:30 后目标日=当天，否则最近交易日；非交易日静默（0 抓取 / 0 写库 / 0 消息）。
- 数据源/口径：新浪原始日 K（不复权，与库同源）；池 = `core_watchlist.txt`(1716) ∪ `universe.json`(1489) = **2664**，与库内 distinct code 2664 **零差集**；每次抓最近 20 根（顺带自愈近端缺口），0.30s 全局节流 / 4 并发，3 次重试，连续限流 10 或连续失败 30 即熔断。
- 群回报（每交易日恰好一行）：
  - 正常：`2026-11-09 ｜ 新增 2,6xx 只 ｜ 库内最新 2026-11-09 ｜ 总行 5,929,xxx`
  - 失败：`2026-11-09 ｜ 未获取 ｜ 原因：<带数字原因>`
  - 发送失败落 `pending_reports.jsonl` 下轮补发；`reported_days.json` 防同日重复回报。
- 口径标定（实测）：库内成交量口径不统一——科创板 688/689 为「股」（同新浪 raw），其余板块（含北交所 920）为「手」= `round(股/100)`；脚本按板块换算，避免 100 倍偏差。`code_p` 仅对新写/更新行按权威 helper 填充，UPSERT 不把既有非空值置 NULL。

### 2.3 验证（2026-11-08，周日非交易日）

| 验证 | 结果 |
|---|---|
| 离线自检 `--selftest` | **11/11 通过**（目标日判定、空库首更、覆盖达标静默、code_p 保全、成交量口径、全失败→未获取、连续 456 熔断、非交易日静默、报表格式等） |
| 真实联网端到端（临时库，5 只跨板块） | ok=5/fail=0，写 100 行；与生产库逐行对拍 **99/99 行 OHLCV 完全一致（0 偏差）**；幂等重跑静默、行数不变 |
| 计划任务 + 群通道 | 临时任务动作 → `schtasks /run` 实测发出并读回一条**明确标注的自检行**（`message_id=om_x100b6030eefe5ca4dfcd3fb6b7acd27`，逐字一致）；随后还原正式动作并复核 XML |
| 非交易日门禁（真实 Task Scheduler/pythonw） | 两次实跑均 `[静默] 不抓取、不写库、不回报`，Last Result=0 |
| 生产库今日变更 | **无**：`COUNT(*)=5,926,277`；`MAX(date)=2026-11-06`；最新日 2,655 只；文件大小 1,155,448,832 B / mtime 2026-11-07 11:53:05 未变 |

### 2.4 限制

1. 真实交易日全链路首次运行在 **2026-11-09 15:50**（今天无法伪造）。
2. InteractiveToken：15:50 未登录/休眠不跑；StartWhenAvailable 登录后补跑；整天未开机由下一交易日运行自愈（每次抓 20 根可补近端缺口）。
3. 若源 15:50 尚未发布当日日 K → 如实报「未获取」，当天不自动重试（任务一日一次），次日自愈并留痕。
4. 与其它写库方并发时依赖 SQLite `busy_timeout=120s`；今日核查无其它 kline/collect 相关计划任务。

## 3. 任务二：开机自启等效验证

### 3.1 方法（用户允许「真实重启 或 等效」；本次取等效，不重启机器）

1. 记录基线（启动项、自动登录、PID/端口/健康/日志/rowid）。
2. **全清**：guard → bridge → 8900 supervisor(watchdog) → serve_data；静置 42s 确认无任何进程自愈、8900 无监听。
3. **按开机启动项原样拉起**（等价于登录时 Explorer 执行启动文件夹）：
   - `wscript.exe "...\Startup\FeishuBridgeGuard.vbs"` → guard → 自动拉起 bridge
   - ShellExecute `"...\Startup\InvestDataSvc.lnk"` → start_invest.ps1 → serve_data + watchdog
4. 轮询恢复并做 claim 测试。

### 3.2 恢复结果

| 服务 | 基线 PID | 全清 | 拉起后 PID（来源） | 结果 |
|---|---|---|---|---|
| guard | 13788 | killed 17:56:33 | **36128**（wscript→vbs） | ✅ |
| bridge | 28812 | killed 17:57:08 | **26376**（guard 拉起） | ✅ 轮询 2s |
| serve_data (8900) | 23132 | killed 17:57:09 | **15404**（InvestDataSvc.lnk） | ✅ 0.0.0.0:8900 LISTEN，`/index.json` HTTP 200 |
| watchdog.ps1 | 18776 | killed 17:57:09 | **22400**（InvestDataSvc.lnk） | ✅ |

- 拉起耗时：`.vbs` 路径全绿 **5.8s**；`.lnk` 路径 8900 监听 ~5.5s / HTTP 200 ~8.5s。
- 全清期间 DSH 3082、Tailscale、Syncthing 未受影响；启动文件夹 8 项内容/mtime 与基线一致。

### 3.3 claim 测试

- 桥自带等价事件自测（`feishu_bridge.py --inject`，走同一 claim/dispatch 代码路径）：`CLAIM om_selftest_... -> #28`，GitHub issue **#28** 建单成功，群里收到 **`claim #28`**；`bridge_state.last_rowid` 与 `orchestrator.sqlite` max(rowid) 对齐，轮询活性正常。
- 桥重启后真人消息亦正常：用户 18:08 群消息被桥 `CLAIM ... -> #29`（真人入口链路在重启后可用）。
- 说明：bot 自己发的群消息按设计 `human=false` 不接单，故不能直接用 bot 消息做 claim 测试。

### 3.4 限制与残留风险（如实）

1. **等效 ≠ 真实冷启动/登录**：未覆盖 Windows 登录外壳、自动登录、profile 加载、启动项并发/慢盘时序；若自动登录失败，启动项不会执行。
2. 自动登录密码在 LSA 机密中，无法只读验证（未重启，故无法执行真实登录验证）。
3. `--inject` 为等价事件，不等同真人 `@DSH主路由 接单:` 入口全链路（已由 18:08 真人消息 claim #29 补证）。
4. 既有脚本风险（本次未修，建议后续加固）：
   - `feishu_bridge_guard.ps1` 单实例判断按**命令行文本匹配**，任何命令行含 `feishu_bridge_guard.ps1` 的 powershell/pwsh 都会让新 guard 静默 `exit 0`；
   - `start_invest.ps1` 会杀**所有**命令行含 `watchdog.ps1` 的 powershell/pwsh，可能误伤同名脚本；
   - `watchdog.log` 每 30s 有 `INBOX ERR: ...LiteralPath 空值` 噪音（既有，与 8900 健康巡检无关）。

## 4. 改动点清单 / 回滚

- **新增（部署机）**：`share_deploy\klines_daily_update.py`；`share_deploy\kline_daily\`（日志/证据）；`share_deploy\boot_verify\`（证据）。
- **注册（部署机）**：Windows 计划任务 `\KlinesDailyUpdate`。
- **未改**：本仓库代码；`klines.db` 以外的生产数据；未删除任何文件。
- **回滚**：`schtasks /change /tn KlinesDailyUpdate /disable`（禁用即可，不删）；其余为新增文件，确认无引用后可手动移除（本次未删除任何文件）。

## 5. 证据位置（部署机）

- `C:\Users\hezheng\share_deploy\kline_daily\evidence.md`（任务一：探查/自检/联网对拍/任务 XML/群消息读回/生产库前后态）
- `C:\Users\hezheng\share_deploy\boot_verify\evidence.md`（任务二：基线/全清/拉起时间线/claim/最终态/限制）

## 6. 验收回执

- Issue #27 JSON 回执：`status=done`（`request_id=feishu-om_x100b603003dcc4a8de9509267396a1a`）。

---
**Note**: This document was produced by an automated agent (Niaaaa717 account) on 2026-11-08.
