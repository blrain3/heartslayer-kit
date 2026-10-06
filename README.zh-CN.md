# heartslayer-kit

bottled_ai「观者碎心」批次的运营与验证工具包 — 驱动观者机器人走完三钥匙 → Act 4 → 击杀碎心（Corrupt Heart）。

守卫式启动、对局守望、卡死自愈、离线决策验证；从达成首次碎心击杀的项目中抽取。

> **里程碑** — 首次击杀碎心：2026-10-05，seed `5N2LLKFRVCE5`，floor 56 / 1180 分，8 回合战斗零空闲回合。

[English](README.md) | **简体中文**

## 结构

```
scripts/   运营脚本（互相引用，需保持同目录）
tools/     分析/验证脚本（需在 bottled_ai 仓库根运行，import rs.*）
docs/      批次档案（首杀战报、分析、事故报告）
```

## scripts/ 运营脚本

| 脚本 | 用途 |
| --- | --- |
| `launch-batch.sh [局数] [--dry-run]` | 五重门控启动：CM 指纹 → 残留进程 → Steam 登录 → 模组数 → OCR 点击 Play；并拉起守望器 |
| `stop-batch.sh [--all]` | 安全停止：bot → ModTheSpire/游戏进程（含 Steam 接管的 jre）→ user units |
| `watchdog-stuck.sh [--once]` | 卡死看护：bot 存活但 run 日志连续 >180s 无写入（双采样）→ 止损并自动重开小批（上限 5 次） |
| `watch-milestones.sh` | 里程碑守望：常驻记录 emerald / sapphire / recall / 心脏 / 胜利 等关键事件 |
| `check-shops.py [logs...]` | 商店采购核对：每笔 `choose N` 的扣金 == 标价 |
| `status.sh` | 当前对局状态一览 |

## tools/ 分析与验证

| 脚本 | 用途 |
| --- | --- |
| `verify_live_run.py <log>` | 单局验证：逐回合余能/出牌/低效标记 + 新旧 comparator 决策对照 + 预算越界检查 |
| `replay_decision.py <log> <end_idx>` | 离线回放某次决策：对比空过线与继续出牌线的评估值 |
| `sim_batch_clean.py` | 干净版批量回放：BigFight(原) vs HeartFight(新)，跨历史心脏局统计决策翻转 |
| `sim_fix_test.py` | 候选修复模拟（预算 5/8 两档）在历史局上的逐点对照 |
| `sim_decode.py` | 解码指定决策的完整出牌序列 + 每步伤害/净损演化 |
| `verify_file_version.py` | 历史用例（配对轨迹 + 合成用例）验证 comparator 修复 |

> tools 均 `import rs.*`：请从 bottled_ai 仓库根运行（如 `cd ~/projects/bottled_ai && python3 /path/to/tools/verify_live_run.py logs/runs/<log>`）。

## 运行环境

- Linux 桌面 + Hyprland 会话（OCR 链路用 `grim`；点击用 `xdotool` 于 `DISPLAY=:0`）
- Steam 已登录（AutoLogin 亦可；`connection_log.txt` 出现 `Logged On`）
- [ModTheSpire](https://github.com/kiooeht/ModTheSpire) + [CommunicationMod](https://github.com/ForgottenArbiter/CommunicationMod) 与 [bottled_ai](https://github.com/xaved88/bottled_ai) 检出
- `bash`、`python3`（3.11+）、`grim`、`tesseract`(+`tesseract-data-eng`)、`xdotool`、systemd 用户会话

在 CachyOS/Arch 上调试通过；其它发行版原则上可用——按需调整显示/OCR 链路与环境变量。

**可移植性**（环境变量，默认值见各脚本头部）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `STEAM_DIR` | `~/.local/share/Steam` | Steam 根目录 |
| `STS_DIR` / `MTS_JAR` / `CM_JAR` | 标准 workshop 路径 | 游戏与模组 jar 路径 |
| `EXPECT_CM_SHA` | 加固版指纹 | CM jar 门控指纹（换版本后同步更新） |
| `BOTTLED_REPO` | `~/projects/bottled_ai` | bot 仓库（读 run 日志） |
| `BOTTLED_WATCH_OUT` | 脚本所在目录 | 守望日志输出目录 |
| `THRESH` / `INTERVAL` / `MAX_RECOVERIES` | 180 / 30 / 5 | 看护参数 |

## 快速上手

```bash
# 1. 确认 Steam 已登录（出现 Logged On）
# 2. 守卫式启动（2 局）
./scripts/launch-batch.sh 2
# 3. 观察（里程碑 + 看护日志写在 scripts/ 旁）
tail -f scripts/milestones.log
# 4. 收局
./scripts/stop-batch.sh --all
# 5. 验证某局（在 bottled_ai 仓库根运行）
python3 /path/to/tools/verify_live_run.py logs/runs/<log>
```

## 工作原理

**五重门控启动。** 每道门控都源自实际踩过的坑：Steam 未登录时 MTS 会「成功」加载 0 个模组；游戏更新会重新校验并回滚 workshop 里的 CommunicationMod jar；上一批半死进程会毒化下一批。门控在动任何状态前大声失败（`GUARD-FAIL: …`）。`--dry-run` 只演练全部检查项，不启动任何东西。

**卡死防线（三重）。** 项目遭遇过两类静默死锁（均有档案，见 `docs/`）：命令执行中的未捕获异常；以及 mod 活着但状态发送被静默抑制。防线刻意分层：

1. 加固版 CommunicationMod — 命令执行兜底 catch（记录日志并强制补发状态）+ 15s 状态流看门狗；
2. bot 端回复超时 — 25s 无回复则用 `state` 探测，最多 3 次；
3. 外部 `watchdog-stuck.sh` — 最后手段：止损并重开小批（上限 5 次恢复），把「整批挂起」降级为「损失一局」。

**OCR 点击。** `grim` → `tesseract tsv` → 找 `play` 词 → `xdotool` 点击。要求 MTS 窗口在当前工作区可见；被盖住则 `GUARD-FAIL: 未找到 Play 按钮`（先把其它全屏窗口收起）。

## 疑难排查

- 游戏更新可能覆盖 workshop 内的 CommunicationMod jar —— 指纹门控会拦截启动，直到重装加固版并更新 `EXPECT_CM_SHA`。
- MTS 启动前 Steam 必须**完全登录**，否则模组列表为空。
- 看护的恢复动作是「重开一批新局」，**不会**续跑卡住的残局。
- `tools/` 需要带 `OBSERVANT_HEARTSLAYER` 策略的 bottled_ai 检出（fork 分支 `verify-localization-fix` 承载移植、comparator 修复与加固配方）。

## docs/ 档案（中文）

- `KILL-SUMMARY.md` — 首杀战报（2026-10-05）
- `BATCH1-SUMMARY.md` / `HEART-FIGHT-ANALYSIS.md` / `RECORDS-SUMMARY.md` — 长批数据与心脏战分析
- `INCIDENT-shop-hang.md` / `INCIDENT-wedge2-F59R3VP6YLV4.md` — 两次 CM 静默死锁事故报告

## 致谢

- [bottled_ai](https://github.com/xaved88/bottled_ai) — 机器人本体；本工具包为运行与验证它而生。
- [CommunicationMod](https://github.com/ForgottenArbiter/CommunicationMod) / [spirecomm](https://github.com/ForgottenArbiter/spirecomm) — 游戏↔进程协议及其生态。
- Slay the Spire © MegaCrit Games。本项目与官方无关；自动化对象为本地正版游戏。

## 许可

[MIT](LICENSE)
