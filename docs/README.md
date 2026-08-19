# GOAI 2026 项目文档索引

> 项目：GOAI 2026 通用双臂协同操作挑战赛
> 更新：2026-08-14

## 快速入口

**接手项目 / 看现状 → 先读 [接手文档.md](接手文档.md)**。  
**本轮怎么跑 → [xiaomi_ops.md](xiaomi_ops.md)**（官方 Xiaomi-1 RoboDojo ckpt，只推理）。

## 权威层级

1. **赛规**：https://xsparkai.com/goai-2026/ 、https://www.goaihz.com/tracks?track=embodied 、[具身未来参赛手册](https://oss.goaihz.com/prod/20260808/67104827-5a82-4080-bfab-b338eff9987d.pdf)
2. **本轮执行**：接手文档 + [xiaomi_ops.md](xiaomi_ops.md) + 下方「进行中」的 design / plan
3. **根目录** [`GOAI 2026 通用双臂协同操作挑战赛.md`](../GOAI%202026%20通用双臂协同操作挑战赛.md) 是 **Agent 执行任务书**，不是组委会文件。其中「第一阶段只用 ACT」对本轮作废。

官方允许自有 / 第三方策略（XPolicyLab 30+ baseline，可接自有模型）。禁止改的是 RoboDojo 任务、评分与 randomization，不是「只能用 ACT」。

## 文档地图（三层）

### 一、状态（当前进行到哪）

| 文档 | 说明 |
| :-- | :-- |
| [接手文档.md](接手文档.md) | 项目状态总入口：环境、进度、坑、命令、待办 |
| [xiaomi_ops.md](xiaomi_ops.md) | Xiaomi-1 官方 ckpt：下载、环境、debug、smoke、提交 |

### 二、计划（接下来做什么，执行中）

| 文档 | 说明 |
| :-- | :-- |
| [设计文档（Xiaomi-1）](superpowers/specs/2026-08-14-xiaomi-baseline-sprint-design.md) | 当前方案：官方 XR-1 RoboDojo ckpt、为何不用 ACT / 0 号 / 5B 零样本、VRAM 门控 |
| [实施计划（Xiaomi-1）](superpowers/plans/2026-08-14-xiaomi-baseline-sprint.md) | 当前执行：Task 0–7 |

### 三、参考（规则 / 环境 / 已取代方案）

| 文档 | 说明 |
| :-- | :-- |
| [Agent 任务书](../GOAI%202026%20通用双臂协同操作挑战赛.md) | 历史 Agent 手册；赛规与本轮策略以本节「权威层级」为准 |
| [environment.md](environment.md) | 环境快照（硬件 / 磁盘 / conda） |
| [dual_boot_mount_guide.md](dual_boot_mount_guide.md) | 双系统挂载 D 盘（已完成；重装/换机时参考） |
| [旧 ACT 设计](superpowers/specs/2026-08-13-goai-baseline-sprint-design.md) | **已被 2026-08-14 Xiaomi-1 方案取代** |
| [旧 ACT 计划](superpowers/plans/2026-08-13-goai-baseline-sprint.md) | **已被 2026-08-14 Xiaomi-1 方案取代** |

## 说明

- `RoboDojo/` 是官方 benchmark 仓库（vendored，自带 `.git`），其中 README 不属于本项目文档，**勿改任务 / 评分**。
- 当前执行机只有 `/dev/vda1`（约 97G / 82G 剩余），没有 `/mnt/data`。大文件默认 `/home/ubuntu/GOAI-2026/`。
- 磁盘敏感：大文件操作前 `df -h /`。本轮不下 240G 训练数据，不下 ACT ckpt，只下约 11G 的 Xiaomi-1 官方 ckpt。Assets ~35G 最后装；10G 加盘不够，缺空间加 50G+。
