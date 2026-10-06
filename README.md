# heartslayer-kit

bottled_ai「观者碎心」批次的运营与验证工具包。

让 bottled_ai 机器人在**观者（OBSERVANT_HEARTSLAYER）**模式下走完三钥匙 → Act 4 → 击杀碎心（Corrupt Heart）；
本仓库收录负责**守卫式启动、对局守望、卡死自愈、结果验证**的脚本与档案。

> 2026-10-05 里程碑：首次击杀碎心（seed `5N2LLKFRVCE5`，floor 56 / 1180 分 / 8 回合 / 零空闲回合）。

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
| `verify_file_version.py` | 历史用例（XDMVJN end#3 文件版对照 + 合成用例） |

> tools 均 `import rs.*`：请从 bottled_ai 仓库根运行（如 `cd ~/projects/bottled_ai && python3 /path/to/tools/verify_live_run.py logs/runs/<log>`）。

## 运行环境

- CachyOS/Arch 笔记本 + Hyprland（本套按此调试；其它发行版需自行调整显示/OCR链路）
- Steam 已登录（AutoLogin；`connection_log` 出现 `Logged On`）
- ModTheSpire 经 `systemd-run --user` 拉起（`mts-ui` 单元）
- 依赖：`python3`、`bash`、`grim`、`tesseract`(+`tesseract-data-eng`)、`xdotool`

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

## 设计要点与已知坑

- **五重门控**：防 Steam 未登录时 MTS 拉到 0 个模组、防 CommunicationMod jar 被 Steam 校验回滚（指纹比对）、防残留实例双开。
- **卡死防线（三重）**：
  1. `watchdog-stuck.sh` 兜底（命中即止损+重开，整批挂起降级为损失一局）；
  2. CommunicationMod 加固补丁（命令执行兜底 catch + 状态发送看门狗，见主仓 `ops/cm-fork/`）；
  3. bot 端 `client.py` 回复超时 + `state` 探测自愈。
- **OCR 点击**要求 MTS 窗口在最前；被其他全屏窗口盖住会 `GUARD-FAIL`（先收起其它全屏程序）。
- 游戏更新可能覆盖 workshop 内的 CM jar —— 门控会拦截并提示重装（加固版指纹见 `EXPECT_CM_SHA`）。

## docs/ 档案

- `KILL-SUMMARY.md` — 首杀战报（2026-10-05）
- `BATCH1-SUMMARY.md` / `HEART-FIGHT-ANALYSIS.md` / `RECORDS-SUMMARY.md` — 长批数据与心脏战分析
- `INCIDENT-shop-hang.md` / `INCIDENT-wedge2-F59R3VP6YLV4.md` — 两次 CM 静默死锁事故报告

## 相关

- 主项目：[blrain3/bottled_ai](https://github.com/blrain3/bottled_ai)（fork；碎心移植 + 心脏战评估修复 + CM 加固补丁）
- 里程碑：首杀 `5N2LLKFRVCE5`（见 docs/）
