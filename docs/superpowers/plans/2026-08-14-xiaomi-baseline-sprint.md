# GOAI 2026 Xiaomi-Robotics-1 冲刺实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用 RoboDojo 官方 Xiaomi-Robotics-1 微调 ckpt（只推理）跑通 24 配置 smoke、部署公网 Policy Server，并完成一次有效成绩提交。

**Architecture:** 当前执行机是 **RTX 4090 云主机**（只有 `/dev/vda1`，约 82G 剩余）。本机同时跑 Generalization Isaac smoke 与 Xiaomi-1 Policy Server。权重是官方 `ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0/`。不训练、不改 benchmark、不用 ACT / Xiaomi-0 / 未微调 5B、不下 240G hdf5。

**Tech Stack:** RoboDojo（Isaac Sim / IsaacLab，torch 2.7.0 cu128）、XPolicyLab/`Xiaomi_Robotics_1`（conda `mibot`，torch 2.8.0 cu128）、bash、git、WebSocket / TLS。

## Global Constraints

- 禁止修改官方 benchmark（task 定义 / reward / randomization / 评分 / configuration）。
- 禁止自己训练 / 微调任何模型。
- 禁止下载 240G 训练数据（`data/hdf5`）。
- 禁止用 Xiaomi-0 Pretrain、禁止用未微调的 `Xiaomi-Robotics-1-5B` 当提交权重。
- ACT 不再作为提交备用；应停止 ACT ckpt 下载。
- 评估环境 `RoboDojo`（torch 2.7.0 cu128）与策略环境 `mibot`（torch 2.8.0 cu128）必须 conda 隔离。
- `action_type` 只能是 `ee`。提交与本地一致；正式评测中途不换模型 / action / 协议。
- 每步用 git 记录可回滚的代码/文档改动；大文件不进 git。
- 任何大文件操作前 `df -h /`。没有 `/mnt/data`。加盘若只有 10G，只够缓存，不够放 Assets（~35G）；Assets 需要 **50G+** 云盘。
- smoke 验收：每个任务 `eval_result/*/_result.json` 的 `eval_time >= 1`（仅退出码 0 不算）。
- 执行前必读：`docs/接手文档.md`、`docs/xiaomi_ops.md`、`docs/superpowers/specs/2026-08-14-xiaomi-baseline-sprint-design.md`。

## 文件结构

- 创建：`results/smoke_test.csv`（smoke 结果表）
- 创建：`results/baseline.csv`（可选本地满测成功率）
- 创建：`docs/policy_server_deploy.md`（部署时再写，本计划 Task 6）
- 修改：`docs/接手文档.md`（本轮踩坑与状态）
- 参考（只读）：`XPolicyLab/policy/Xiaomi_Robotics_1/README.md`、`INSTALLATION.md`、`deploy.yml`、`scripts/robodojo.sh`

---

## Task 0: 磁盘评估（本机只有 vda1）

**Files:**
- 无必须新建（只读检查；必要时补 `.gitignore`）

**验证标准:** `df -h /` 剩余清楚；安装顺序定为 代码 → mibot → ckpt → Isaac → Assets；若 Assets 装不下，加 **50G+** 盘而不是 10G。

- [ ] **Step 1: 看磁盘**

```bash
df -h /
lsblk
```

预期：只有 `vda1` 挂在 `/`。2026-08-14 实测约 97G 总 / 82G 剩余。没有 `/mnt/data`。

- [ ] **Step 2: 确认不拉 ACT / 全量 hdf5**

不要跑 `download_ckpt.sh huggingface ACT`，不要下 `data/hdf5`。

- [ ] **Step 3: 空间预算（装仿真，仅 Generalization）**

| 顺序 | 项 | 大约 | 备注 |
| :-- | :-- | :-- | :-- |
| 1 | 代码 clone | <2G | 先装 |
| 2 | `mibot` + `RoboDojo` conda | 15–30G | 必须在 ext4 `/` |
| 3 | Xiaomi-1 ckpt + VLM 缓存 | 15–20G | 可迁到大云盘 |
| 4 | Isaac / IsaacLab / CuRobo | 20–40G | |
| 5 | GOAI Assets | ~35G | **最后装**；82G 很可能不够 |
| — | 加 10G 云盘 | 10G | 只够 pip/HF 缓存，**不够 Assets** |

每装完一步 `df -h /`。剩余 < 20G 时停，不要开始下 Assets。

- [ ] **Step 4: 确认 git 不会跟踪大文件**

```bash
git status
```

预期：无意外未跟踪的 `.pt` / `.ckpt` / Assets。缺规则则补 `.gitignore` 再继续。

---

## Task 1: 建 `mibot` 策略环境

**Files:**
- 无（conda 环境）

**验证标准:** conda 环境 `mibot`（或 `MIBOT_CONDA_ENV` 指定名）存在；`torch==2.8.0`；CUDA 可用。**不要**改 `RoboDojo` 环境。

- [ ] **Step 1: 确认适配目录**

```bash
ls XPolicyLab/policy/Xiaomi_Robotics_1/
head -80 XPolicyLab/policy/Xiaomi_Robotics_1/README.md
```

预期：有 `install.sh`、`eval.sh`、`deploy.yml`、`INSTALLATION.md`。

- [ ] **Step 2: 安装**

```bash
cd XPolicyLab/policy/Xiaomi_Robotics_1
bash install.sh
```

预期：创建 `mibot`（可用 `MIBOT_CONDA_ENV` 改名）。失败则按 `INSTALLATION.md` 手工装，**最小改动**，记到接手文档。不要把包装进 `RoboDojo`。4090 上 flash-attn 通常比 5070 Ti 好装。

- [ ] **Step 3: 验环境**

```bash
conda run -n mibot python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
conda run -n mibot python -c "import mibot; print('mibot ok')"
```

预期：`2.8.0`（或 README 指定的 2.8.x）、`True`、`mibot ok`。若 torch 落到 2.7.x，说明装错环境，停下来查。

---

## Task 2: 下载官方 Xiaomi-1 RoboDojo ckpt

**Files:**
- 权重放项目下 `checkpoints/`（本机没有 `/mnt/data`；若已挂 50G+ 云盘则放到云盘再软链）

**验证标准:** 存在 `RoboDojo-sim-arx_x5-ee-0/`，内含 `config.py`（或适配器能搜到的嵌套 `config.py`）以及 `last.ckpt/`；体积约 11G。

- [ ] **Step 1: 目标目录**

```bash
df -h /
mkdir -p /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1
```

若已挂大云盘（50G+），改到该挂载点；**10G 加盘不要用来放 ckpt+Assets。**

- [ ] **Step 2: 只下 XR-1 目录**

```bash
hf download RoboDojo-Benchmark/RoboDojo \
  --repo-type dataset \
  --include "ckpt/RoboDojo/Xiaomi_Robotics_1/*" \
  --local-dir /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 \
  --local-dir-use-symlinks False
```

**优先魔搭**：`bash scripts/RoboDojo/download_ckpt.sh modelscope Xiaomi_Robotics_1`，或 wget `modelscope.cn/datasets/RoboDojo-Benchmark/RoboDojo/resolve/master/ckpt/RoboDojo/Xiaomi_Robotics_1/...`。HF 镜像会跳海外 CDN，当备用。不要用 `download_ckpt.sh huggingface ACT`。

- [ ] **Step 3: 让 `ckpt_name=Xiaomi_Robotics_1` 能解析**

```bash
find /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 -name 'config.py' | head
find /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 -name 'mp_rank_00_model_states.pt' | head
du -sh /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1

mkdir -p XPolicyLab/policy/Xiaomi_Robotics_1/checkpoints
ln -sfn /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 \
  XPolicyLab/policy/Xiaomi_Robotics_1/checkpoints/Xiaomi_Robotics_1
```

若下载后多了一层 `ckpt/RoboDojo/Xiaomi_Robotics_1/`，把软链指到**含 `RoboDojo-sim-arx_x5-ee-0` 的那一层**，或在 `deploy.yml` 设 `model_dir` 为该目录的绝对路径。结论记接手文档。

- [ ] **Step 4: 不要转换 5B**

不要跑 `weight_convert.py`、不要 `hf download XiaomiRobotics/Xiaomi-Robotics-1-5B`，除非以后改决策去自己微调。

---

## Task 3: debug eval（不启动 Isaac）

**Files:**
- 无（只跑）

**验证标准:** `EVAL_ENV_TYPE=debug` 完成一次接线检查，策略能加载官方 ckpt 并返回动作形状。

- [ ] **Step 1: 读 deploy.yml**

确认 `action_type: ee`，`policy_name: Xiaomi_Robotics_1`。不要改成 `joint`。

- [ ] **Step 2: debug**

```bash
cd XPolicyLab/policy/Xiaomi_Robotics_1
export EVAL_ENV_TYPE=debug
bash eval.sh RoboDojo stack_bowls Xiaomi_Robotics_1 arx_x5 ee 0 0 0 mibot RoboDojo
```

参数顺序以该目录 README 为准：`<bench> <task> <ckpt_name> <env_cfg> <action_type> <seed> <policy_gpu> <env_gpu> <policy_env> <eval_env>`。

预期：不启 Isaac；能加载 ckpt；无 `joint` 报错、无缺 `config.py`。失败则先修路径（Task 2 Step 3），再查环境（Task 1）。

---

## Task 4: 显存门控

**Files:**
- 修改：`docs/接手文档.md`（记录本机 / 云 GPU 结论）

**验证标准:** 4090 上 bf16 可加载官方 ckpt 并出动作；与 Isaac 同卡时只跑单环境。

- [ ] **Step 1: 加载时看显存**

```bash
nvidia-smi
```

预期：无 OOM。5B 推理常见 14–18G，4090 24G 应有余量。与 Isaac 同卡则分进程、一次一个环境。

- [ ] **Step 2: 记录**

本机就是提交用的 Policy Server，不必再迁到另一台 GPU。把 `nvidia-smi` 峰值写进接手文档。

---

## Task 5: doctor + smoke（24 配置）

**Files:**
- 创建：`results/smoke_test.csv`

**验证标准:** 24 个配置写出 `_result.json` 且 `eval_time >= 1`。时间不够则记录降级范围。

- [ ] **Step 1: doctor**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh doctor
```

策略项若因 `mibot` 未进 doctor 白名单而 FAIL，可 `--skip-policy` 先过环境/资产，策略问题留给 smoke 暴露。不要为了 doctor 去改任务或评分代码。

- [ ] **Step 2: 单任务真跑测耗时（显存门控通过后）**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh eval \
  --policy-dir XPolicyLab/policy/Xiaomi_Robotics_1 \
  --task stack_bowls \
  --ckpt Xiaomi_Robotics_1 \
  --policy-env mibot \
  --eval-env RoboDojo \
  --action-type ee \
  --eval-num 1
```

预期：能跑、无 OOM。记下单集耗时。

- [ ] **Step 3: smoke**

```bash
mkdir -p logs/smoke
nohup bash scripts/robodojo.sh smoke \
  --dimension generalization \
  --policy-dir XPolicyLab/policy/Xiaomi_Robotics_1 \
  --ckpt Xiaomi_Robotics_1 \
  --policy-env mibot \
  --eval-env RoboDojo \
  --action-type ee \
  --eval-num 1 \
  > logs/smoke/smoke_$(date +%Y%m%d_%H%M%S).log 2>&1 &
```

`--ckpt` 对 XR-1 是**同一份**多任务权重，不再需要 ACT 的 per-task 软链。若 `robodojo.sh` 对 `--ckpt` 的拼接与 policy 目录约定不一致，改用 README 的 `eval.sh` 按任务循环，结论记接手文档。

- [ ] **Step 4: 验收 eval_time**

```bash
find eval_result -name "_result.json" | wc -l
for f in $(find eval_result -name "_result.json"); do
  python -c "import json; d=json.load(open('$f')); print(d.get('eval_time',0), '$f')"
done | sort -n
```

预期：24 个文件，值均 `>= 1`。退出码 0 不够。FAIL 的任务看日志区分环境 vs 策略；**不**为此改模型。

- [ ] **Step 5: 汇总 csv 并 commit 结果表（不含权重）**

```bash
bash scripts/robodojo.sh summarize
# 再整理到 results/smoke_test.csv
git add results/smoke_test.csv
git commit -m "test: Xiaomi-1 smoke 24 配置结果"
```

---

## Task 6: 公网 Policy Server

**Files:**
- 创建：`docs/policy_server_deploy.md`

**验证标准:** 评测端可连 `wss://host:port`；加载的仍是官方 XR-1 ckpt；`ee`。

- [ ] **Step 1: 就在这台 4090 上部署**

公网 IP + 端口 + TLS。不通再 Cloudflare Tunnel。

- [ ] **Step 2: 启动策略服务**

用 `XPolicyLab/policy/Xiaomi_Robotics_1` 的 `eval.sh` / `setup_eval_policy_server.sh`（以该目录 README「Deployment Flow」为准）。`deploy.yml`：`protocol: ws`，`action_type: ee`。

- [ ] **Step 3: TLS 与可达性**

官方要求评测端可访问的公网 WebSocket。申请表填 host 与 port（门户会拼成 `ws://` / `wss://`）。内网 IP 不行。穿透时确认 WebSocket 升级不被代理剥掉。

- [ ] **Step 4: 冻结**

审核和正式评测期间：同一进程、同一 ckpt、同一 `ee`、同一协议。不换模型。

- [ ] **Step 5: 写 `docs/policy_server_deploy.md` 并 commit**（不要把密钥写进去）

---

## Task 7: 提交

**Files:**
- 回执可放 `docs/`（无密钥）

**验证标准:** xsparkai 申请成功；评测结果邮件已保存；goaihz 作品提交含代码仓库链接与该邮件。每队 3 次，取最高；不要用正式提交调试。

- [ ] **Step 1: 本地自测完成后再申请**

对照评测站：完成本地准备与自测 → 本站评测申请 → 保存结果邮件 → goaihz「提交作品」。

- [ ] **Step 2: 填表**

队伍名、联系人、手机、邮箱、Policy Server host/port（1–8 个端点）、策略名（仅字母数字下划线，例如 `Xiaomi_Robotics_1`）、**Action Type = ee**。

- [ ] **Step 3: goaihz 交作品**

必须包含代码仓库与评测结果邮件。截止以官网为准（具身未来初赛曾公布 8 月 20 日 23:59）。

- [ ] **Step 4: 更新接手文档状态**

---

## Task 8（可选）: 本地满测降级

时间允许再跑满 episode，写入 `results/baseline.csv`。不够则保持 smoke + 提交，满测提交后补。代表任务建议：`stack_bowls`、`push_T`、`fold_clothes`。
