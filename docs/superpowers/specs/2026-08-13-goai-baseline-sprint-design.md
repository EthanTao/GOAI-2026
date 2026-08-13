# GOAI 2026 Baseline 冲刺方案设计（方案 A）

> 日期：2026-08-13
> 状态：已确认（brainstorming 收敛）
> 关联文档：根目录 `GOAI 2026 通用双臂协同操作挑战赛.md`、`docs/接手文档.md`

---

## 1. 背景与硬约束

- **参赛**：GOAI 2026 通用双臂协同操作挑战赛（初赛评测 Generalization 维度，12 任务 × standard/random = 24 配置）。
- **当前进度**：评估环境已装好（RoboDojo / Isaac Sim / CuRobo），后台正在下载 Assets（~35G）与官方 ACT ckpt（~34G），**baseline 尚未跑通**。
- **时间**：距初赛提交约几天 ~ 一周（紧）。
- **硬件**：RTX 5070 Laptop 12GB 显存、30GB 内存、数据盘 `/mnt/data` 剩余 ~137G。
- **规则硬约束**：第一阶段禁止设计新模型；禁止修改官方 benchmark（task 定义 / reward / randomization / 评分）。

---

## 2. 目标与非目标

### 目标

在截止前用**官方 ACT checkpoint** 跑通 24 配置本地评测，部署 Policy Server，完成**一次有效成绩提交**，兜住"有成绩"这条底线。

### 非目标（这几天明确不做）

- ❌ 不训练 / 微调任何模型（ACT、小米 VLA 均不）。
- ❌ 不下载 240G 训练数据。
- ❌ 不修改官方 benchmark 代码。
- ❌ 小米 VLA 完全冻结，等出分后有富余时间再启动。

---

## 3. 关键决策记录

### 决策 1：小米 VLA 延后

原始诉求"把小米机器人 VLA 融进去"。经 brainstorming 收敛后明确：

- 目标模型为 **[Xiaomi-Robotics-0](https://xiaomi-robotics-0.github.io)**（[代码](https://github.com/XiaomiRobotics/Xiaomi-Robotics-0)、[权重](https://huggingface.co/XiaomiRobotics)），47 亿参数，VLM（Qwen3-VL-4B）大脑 + 16 层 DiT 小脑，流匹配生成动作片段。
- 但"**拿成绩为主 + 时间紧 + 12GB 显存 + 磁盘 137G + 第一阶段禁新模型**"五重约束叠加，训练/微调 VLA 在几天内不可行。
- **结论**：VLA 延后为"后续路线 A 加分项"，不进入本次冲刺。

### 决策 2：方案 A（官方 ckpt 直达提交）

在三条路径中选择 **A**：不碰训练，官方 ckpt 跑通即提交。

- 方案 B（多 seed 集成 / action 平滑等零训练优化）→ 提交后有余力才做的加分项。
- 方案 C（下载数据子集微调 ACT）→ 排除（磁盘不够 + 时间不够 + 12GB 显存训练吃紧）。

### 决策 3：完整评测可降级

第 ⑧ 步完整评测若时间/算力不允许，降级为"smoke 跑通 + 挑 2~3 个代表任务跑满 episode"，先保提交，完整 24 配置满测在**提交之后**再补跑。

---

## 4. 关键路径

①~⑧ 全部在本地跑（不需要云服务器），⑨~⑩ 为提交环节。

| 步 | 做什么 | 目的 | 产物 |
| :-- | :-- | :-- | :-- |
| ① | 确认 Assets/ckpt 下载完成 + `df -h /mnt/data` 查磁盘 | 底料齐了没 | 两个下载完成 + 磁盘余量清楚 |
| ② | 建 ACT 策略环境（torch 2.4.1，与评估环境 RoboDojo 隔离） | 让策略代码能跑 | 独立 conda 环境 |
| ③ | 修复 ckpt 命名（`policy_epoch_6000_seed_0.ckpt` → `policy_last.ckpt` 符号链接） | 让策略找到权重 | 每任务目录有 `policy_last.ckpt` |
| ④ | 搞清 per-task ckpt 映射（读 `scripts/robodojo.sh` + `run_policy_eval.sh`） | 12 任务各用各的权重 | 明确 smoke 如何按任务选 ckpt |
| ⑤ | `doctor` 体检 | 跑之前先检查 | doctor 通过 |
| ⑥ | smoke 测试（`--dimension generalization`，24 配置各 1 集） | 验证整条链路通 | 24 配置运行结果 |
| ⑦ | 验收 `eval_result/*/_result.json` 的 `eval_time >= 1` | 确认真跑起来了 | 24 配置全部通过 |
| ⑧ | 本地完整评测（跑满 episode 拿真实成功率；可降级） | 拿到真实成绩 | `results/` 成功率表 |
| ⑨ | 部署 Policy Server（公网 `wss://` + TLS） | 让官方能连进来 | 公网可达的服务器 |
| ⑩ | 提交申请（队伍名/host/port/策略名/action type） | 完成一次有效成绩 | 官方回执 |

### 各步要点

- **② 环境隔离**：ACT 依赖 torch 2.4.1，评估环境是 torch 2.7.0 cu128，两者不可混装（CUDA 冲突），必须 conda 分开。
- **③ 符号链接**：官方 ckpt 名为 `policy_epoch_6000_seed_X.ckpt`，但 `XPolicyLab/policy/ACT/detr/act_policy.py` 只认 `policy_last.ckpt`，需为每个任务建软链 `policy_last.ckpt -> policy_epoch_6000_seed_0.ckpt`。
- **④ 映射**：官方 ckpt 按任务分目录，但 `smoke --ckpt <名>` 对所有任务用同一个名字，需读脚本确认是否有按 task 拼 ckpt 的逻辑；若无，最可能是"逐任务 `eval --task <task> --ckpt act-RoboDojo-<task>`"或"各任务软链成统一名"。
- **⑦ 验收**：退出码 0 不够，必须 `eval_time >= 1`（见 RoboDojo CLAUDE.md）。

---

## 5. 风险点与对策

| # | 风险 | 对策 |
| :-- | :-- | :-- |
| 1 | **磁盘**：137G 已压 Assets+ckpt 约 69G，评测中间产物可能爆盘 | 每步前 `df -h`；清理 ckpt 中用不到的 seed/task |
| 2 | **per-task ckpt 映射机制未知**（接手文档坑 5） | 先读脚本，最快路径是"逐任务 eval"或"各任务软链" |
| 3 | **Policy Server 公网**：要求公网 IP + TLS + WebSocket，内网 IP 不可用 | 优先云服务器；无则用 Cloudflare Tunnel / Ngrok 内网穿透（见第 7 节） |
| 4 | **Isaac Sim 跑 24 配置耗时**：5070 Laptop 可能很慢 | 先跑 1 配置测单集耗时，再估算总量；必要时降级提交（决策 3） |
| 5 | **硬件吃紧**：12GB 显存 + 30G 内存对 Isaac Sim 偏小 | 单配置先跑通，避免多配置并行 |

---

## 6. 验收标准

- [ ] 24 配置 smoke 全部 `eval_time >= 1`（非仅退出码 0）。
- [ ] 本地评测跑出每个任务成功率（或降级：2~3 代表任务满测）。
- [ ] Policy Server 公网 `wss://` 可访问。
- [ ] 提交一次，拿到官方回执。

---

## 7. Policy Server 公网方案预案（第 ⑨ 步）

- **首选**：云服务器部署（公网 IP + 端口 + TLS）。
- **无云服务器时的替代**（内网穿透，让公网通过隧道访问本地）：
  1. **Cloudflare Tunnel**（推荐）：免费、稳定、自带 TLS、支持 WebSocket，需免费 Cloudflare 账号。
  2. **Ngrok**：一条命令最省事，但免费版 URL 每次变、有连接数/带宽限制，正式评测不稳。
  3. **Tailscale Funnel**：免费，但并发限制、略门槛。
- **穿透方案风险**：额外延迟/抖动对实时评测不利；本地机器需 24h 开机 + 网络稳定；host 字段稳定性。
- 该问题到第 ⑨ 步再定即可（搭穿透几十分钟可完成，不提前准备）。
