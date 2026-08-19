# GOAI 2026 Baseline 冲刺方案设计（Xiaomi-Robotics-1）

> 日期：2026-08-14
> 状态：已确认
> 取代：`docs/superpowers/specs/2026-08-13-goai-baseline-sprint-design.md`（官方 ACT 方案）
> 关联：`docs/接手文档.md`、`docs/xiaomi_ops.md`、`docs/superpowers/plans/2026-08-14-xiaomi-baseline-sprint.md`

---

## 1. 背景

- **参赛**：GOAI 2026 通用双臂协同操作挑战赛。初赛只评 RoboDojo **Generalization**（12 任务 × standard / `_random` = 24 配置）。
- **当前进度**：提交基线已定为官方 Xiaomi-1 RoboDojo ckpt。笔记本上曾装过 RoboDojo/Isaac；**当前执行机是 4090 云主机**（见下），需在本机重装 Generalization 仿真 + 策略服务。
- **时间**：具身未来赛道初赛作品提交截止 2026-08-20 23:59（以官网为准）。
- **当前执行机（2026-08-14 实测）**：云主机 **RTX 4090 24GB**；块设备只有 `vda` 100G，可用分区 **`/dev/vda1` → `/`，约 97G 总 / 15G 已用 / 82G 剩余**。没有 `/mnt/data`。笔记本（5070 Ti 12GB + NTFS 数据盘）仅作历史环境，不再当提交机。
- **本机仿真范围**：只装 **Generalization** 本地 smoke（24 配置），不装五维全量、不下 240G 训练数据。官方正式评测仍在官方客户端跑仿真；本机 Isaac 是为了交之前自己跑通。

### 权威来源（不要混）

1. **赛规**：https://xsparkai.com/goai-2026/ 、https://www.goaihz.com/tracks?track=embodied 、[具身未来参赛手册 PDF](https://oss.goaihz.com/prod/20260808/67104827-5a82-4080-bfab-b338eff9987d.pdf)
2. **本轮执行**：本文 + 实施计划 + `docs/xiaomi_ops.md` + `docs/接手文档.md`
3. **根目录** `GOAI 2026 通用双臂协同操作挑战赛.md` 是 **Agent 执行任务书**，不是组委会赛规。其中「第一阶段只用 ACT / 每次只改一个因素」**对本轮作废**。

官方允许自有模型：评测站写明 XPolicyLab 已集成 30 余个 baseline，可接入自有模型；参赛手册 8.1 要求围绕 VLA / WAM / 相关具身模型。正式评测期间不得更换模型、动作类型或协议行为。每队最多 **3 次**有效成绩，取最高。

仍禁止修改：RoboDojo 任务定义 / evaluation / reward / randomization / official configuration。

---

## 2. 目标与非目标

### 目标

用 **RoboDojo 官方发布的 Xiaomi-Robotics-1 微调 checkpoint**（只推理、自己不训）跑通 24 配置本地 smoke，部署公网 Policy Server，完成至少一次有效成绩提交。

权重路径（官方数据集内）：

```text
ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0/
```

来源：HuggingFace `RoboDojo-Benchmark/RoboDojo`。适配名：`XPolicyLab/policy/Xiaomi_Robotics_1`。动作空间：**仅 `ee`**。

公开对照（RoboDojo 仿真，Xiaomi-Robotics-1 论文 Tab. 5 / XPolicyLab 2026-08-04 快照）：Generalization **23.55 / 17.00%**，五维平均 **20.07 / 13.93%**。这是该 ckpt 在 **RoboDojo 数据上微调后** 的成绩，不是 5B 零样本。

### 非目标

- 不自己训练 / 微调 Xiaomi-1、Xiaomi-0 或 ACT。
- 不下载 240G `data/hdf5` 训练数据。
- 不用 Xiaomi-0 Pretrain，不用未在 RoboDojo 上微调的 `Xiaomi-Robotics-1-5B`。
- 不把 ACT 当提交备用。
- 不改官方 benchmark 代码。

---

## 3. 为何是这份 ckpt（决策记录）

GOAI 评的是 RoboDojo Generalization、ARX X5、Isaac Sim。要对齐这条线，权重必须已经在 **同一套仿真、同一台本体、同一套任务数据** 上训过。自己不训的前提下，只能选官方已经微调好的 ckpt。

| 候选 | 和 GOAI 同分布 | 公开 Gen 成绩 | 自己还要训吗 | 本轮 |
| :-- | :-- | :-- | :-- | :-- |
| 官方 ACT ckpt | 是 | 0.69 / 0.56% | 否 | 否（成绩过低，且已冻结） |
| Xiaomi-0 Pretrain 零样本 | 否（DROID / MolmoAct / 自采，无 RoboDojo） | 无 | 否 | 否 |
| `Xiaomi-Robotics-1-5B` | 否（UMI 100k h + 跨本体 10k h：Bridge / RT-1 / DROID / 自采双臂） | 无 | 否 | 否 |
| **官方 `Xiaomi_Robotics_1` RoboDojo ckpt** | **是**（5B + RoboDojo HDF5 微调） | **23.55 / 17.00%** | **否** | **是** |

XPolicyLab `policy/Xiaomi_Robotics_1/README.md` 写明：评测下载 `ckpt/RoboDojo/Xiaomi_Robotics_1/*`；若自己训练才从 `Xiaomi-Robotics-1-5B` 转换后 `train.sh`。本轮走评测路径，不走训练路径。

### 其它固定决策

- **action_type = ee**。适配器 packed 60 维动作没有关节槽，`joint` 会在启动时报错。提交字段必须与本地一致。
- **单份 ckpt 覆盖 12 个 generalization 任务**（目录名 `RoboDojo-sim-arx_x5-ee-0`），不再有 ACT 那种 per-task `act-RoboDojo-<task>` 映射问题。
- **完整 24 配置满测可降级**：先保 smoke + 提交；时间不够则 2～3 个代表任务满测，其余提交后再补。
- **本机同时装仿真**：4090 上装 Generalization 用 Isaac + GOAI Assets + `mibot` + 官方 ckpt。加盘若只加 **10G，不够扛 Assets（~35G）**；空间不够应加 **50G+** 云盘挂 Assets/ckpt，10G 只能放 pip/HF 缓存。

---

## 4. 关键路径

本地 smoke 与官方评测共用同一份 ckpt、同一个 Policy Server 协议（WebSocket，ee）。

```text
官方 RoboDojo Xiaomi-1 ckpt
        ↓
Xiaomi_Robotics_1 Policy Server（conda mibot, torch 2.8.0）
        ↓
   ┌────┴────┐
   ↓         ↓
本地 Isaac   官方评测客户端
（smoke）    （wss:// 提交）
```

| 步 | 做什么 | 产物 |
| :-- | :-- | :-- |
| 0 | 磁盘评估（本机只有 `/`）；确认不加 10G 盘扛不住 Assets | 空间清楚、安装顺序定好 |
| 1 | 建 `mibot`（`Xiaomi_Robotics_1/install.sh`，torch 2.8.0 cu128） | 独立策略环境 |
| 2 | 只下 `ckpt/RoboDojo/Xiaomi_Robotics_1/*` 到 `/` 上项目目录 | 评测能解析到 `config.py` + `last.ckpt/` |
| 3 | `EVAL_ENV_TYPE=debug` | obs→action 接线通过 |
| 4 | 4090 显存确认（预期够用）；Isaac 与 VLA 仍建议分进程、单环境 | 本机可推理 |
| 5 | smoke `--dimension generalization`，24 配置 | `_result.json` 且 `eval_time >= 1` |
| 6 | 公网 Policy Server `wss://` + TLS | 评测端可达 |
| 7 | xsparkai 申请 + 保存评测邮件 + goaihz 交作品 | 一次有效成绩 |

---

## 5. 磁盘与显存（当前 4090 云机）

### 磁盘（只有 vda1）

安装顺序：**代码 → `mibot` → ckpt → Isaac → Assets（最后）**。每步后 `df -h /`。

| 项 | 大约 | 放哪 |
| :-- | :-- | :-- |
| `mibot` + `RoboDojo` conda | 15–30G | `/`（必须 ext4） |
| Xiaomi-1 ckpt + VLM 缓存 | 15–20G | `/` 或大云盘 |
| Isaac Sim / IsaacLab / CuRobo | 20–40G | `/` |
| GOAI Assets（Generalization 包） | ~35G | **最可能爆盘**；10G 加盘不够 |
| smoke 产物 | 5–20G | `/` 或云盘 |
| 五维全量 / 240G hdf5 / ACT 34G | — | 不要 |

82G 剩余：**策略+ckpt+Isaac 有机会装下；再加 35G Assets 会顶满。** 加盘请加 **50G+** 专放 Assets/ckpt。加 10G 只适合 HF/pip 缓存，不能当数据盘。

### 显存（4090 24GB）

- ckpt 文件约 11GB，推理常见 14–18G。**4090 够跑 Policy Server。**
- 与 Isaac 同卡：单环境、分进程可以试；不要并行多环境。
- 笔记本 12GB 不再作为提交推理卡。

---

## 6. 风险与对策

| # | 风险 | 对策 |
| :-- | :-- | :-- |
| 1 | 4090 上 Isaac+VLA 同卡峰值 | 单环境、分进程；`nvidia-smi` 盯一次 smoke |
| 2 | flash-attn / torch 2.8 编译失败 | 按 `INSTALLATION.md` 最小改动；不混装进 `RoboDojo` 环境 |
| 3 | 只有 82G：Assets 35G 装不下 | Assets 放到 50G+ 云盘；不要指望 +10G |
| 4 | HuggingFace 429 | 镜像 `HF_ENDPOINT=https://hf-mirror.com` 或 ModelScope 等价路径 |
| 5 | Isaac 24 配置耗时长 | 先单配置测时；必要时降级满测 |
| 6 | 正式评测中途换模型 / action / 协议 | 提交后冻结；3 次机会不拿来调试 |
| 7 | 零样本预期错位 | 用的是已微调官方 ckpt，不要拿 5B / Xiaomi-0 去预期 |

---

## 7. 验收标准

- [ ] 官方 XR-1 RoboDojo ckpt 到位，`eval` 能解析 `config.py` + `last.ckpt/`。
- [ ] debug eval 能完成一次 obs→action。
- [ ] 4090 上模型可加载；磁盘方案已定（`/` 或 50G+ 云盘放 Assets）。
- [ ] 24 配置 smoke 全部 `eval_time >= 1`（非仅退出码 0）；时间不够则记录降级范围。
- [ ] Policy Server 公网 `wss://` 可访问；`action_type=ee`；策略名仅字母数字下划线。
- [ ] 完成一次 xsparkai 评测申请，并在 goaihz 提交作品（含评测结果邮件）。

---

## 8. Policy Server 公网预案

- **首选**：就在这台 4090 上跑 Policy Server（公网 IP + 端口 + TLS）。
- 无公网或端口不通时：Cloudflare Tunnel；Ngrok 免费版正式评测慎用。
- 审核和正式评测期间保持同一服务在线，不更换模型、动作类型或协议行为。
