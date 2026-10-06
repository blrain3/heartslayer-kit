# 事故#2：CM 静默死锁（F59R3VP6YLV4，2026-10-05 19:15:46）
- 现场：act3/f44 商店（三钥匙全收后）；bot 发 `choose 10`（fear potion）→ mod 无回复
- 事实：无异常日志；mod 无 Sending；bot 同步阻塞；game 主循环正常；jstack 无死锁
- 与事故#1（26Y37ID1YW3W）同指纹：净化流后首次购买 → 静默
- 判定：状态发送被静默抑制（非异常类；v1 异常兜底不触发）
- 处置：CM v2 看门狗（15s 补发/强制 resume + WARN）+ bot client 超时探测自愈（25s→state×3）+ stop-batch 游戏进程终止
- 证据：本目录 jstack-frozen.txt / SlayTheSpire-frozen.log / 本 run 日志；回执 cm-update-20261005/CM-HARDENING2-receipt.md
- 残局：弃（证据留档）；重启后即弃
