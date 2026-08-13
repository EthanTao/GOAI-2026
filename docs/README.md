# GOAI 2026 项目文档索引

> 项目：GOAI 2026 通用双臂协同操作挑战赛
> 更新：2026-08-13

## 快速入口

**接手项目 / 看现状 → 先读 [接手文档.md](接手文档.md)**（唯一的状态入口：环境、进度、踩坑、命令速查、下一步待办都在里面）。

## 文档地图（三层）

### 一、状态（当前进行到哪）

| 文档 | 说明 |
| :-- | :-- |
| [接手文档.md](接手文档.md) | 项目状态总入口：环境、进度、7 个坑、命令速查、待办清单 |

### 二、计划（接下来做什么，执行中）

| 文档 | 说明 |
| :-- | :-- |
| [设计文档](superpowers/specs/2026-08-13-goai-baseline-sprint-design.md) | baseline 冲刺设计：目标/非目标、关键决策、风险点 |
| [实施计划](superpowers/plans/2026-08-13-goai-baseline-sprint.md) | baseline 冲刺实施计划：Task 1~11 分步执行步骤 |

### 三、参考（权威规则 / 历史记录）

| 文档 | 说明 |
| :-- | :-- |
| [官方赛题](../GOAI%202026%20通用双臂协同操作挑战赛.md) | 官方赛题，**一切以此为准**（根目录） |
| [environment.md](environment.md) | 环境快照（赛题要求记录的文件，硬件/磁盘/conda 明细） |
| [dual_boot_mount_guide.md](dual_boot_mount_guide.md) | 双系统挂载 D 盘指南（一次性操作，已完成；重装/换机时参考） |

## 说明

- `RoboDojo/` 是官方 benchmark 仓库（vendored，自带 `.git`），里面的 README/文档是第三方自带的，**不属于本项目文档，勿动**。
- 数据目录 `Assets/` `data/` `checkpoints/` `logs/` `results/` 是指向 `/mnt/data/GOAI/` 的符号链接；`docs/` 已是真实目录（纳入版本控制）。
- 磁盘空间敏感：任何大文件操作前先 `df -h /mnt/data`。
