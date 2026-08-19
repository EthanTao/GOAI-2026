# Xiaomi-Robotics-1 操作手册

> 更新：2026-08-14
> 本轮提交权重：**RoboDojo 官方 Xiaomi-1 微调 ckpt**（只推理）
> 当前执行机：**RTX 4090 云主机**，只有 `/dev/vda1`（约 82G 剩余），本机装 Generalization 仿真 + Policy Server
> 设计 / 分步计划：`docs/superpowers/specs/2026-08-14-xiaomi-baseline-sprint-design.md`、`docs/superpowers/plans/2026-08-14-xiaomi-baseline-sprint.md`

XPolicyLab 适配目录：`XPolicyLab/policy/Xiaomi_Robotics_1/`。上游说明以该目录 `README.md`、`INSTALLATION.md`、`deploy.yml` 为准。

**磁盘：** 没有 `/mnt/data`。大文件默认 `/home/ubuntu/GOAI-2026/checkpoints/`。加 **10G** 云盘不够放 Assets（~35G），只够缓存；Assets 请加 **50G+**。安装顺序：代码 → conda → ckpt → Isaac → **最后** Assets。每步 `df -h /`。

---

## 1. 用哪份权重

**用这个：**

```text
ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0/
```

HuggingFace 数据集 `RoboDojo-Benchmark/RoboDojo`，约 11GB。从 `Xiaomi-Robotics-1-5B` 出发、在 RoboDojo HDF5 上微调后的评测权重。公开 Generalization：score 23.55 / SR 17.00%。

**不要用：**

| 不要 | 原因 |
| :-- | :-- |
| `XiaomiRobotics/Xiaomi-Robotics-0-Pretrain` | 另一代模型；预训练不含 RoboDojo |
| `XiaomiRobotics/Xiaomi-Robotics-1-5B` | 底座 post-train，未在 RoboDojo 上微调 |
| `ckpt/RoboDojo/ACT/` | 已冻结，不是本轮提交 |

自己不训练：不要跑 `process_data.sh` / `train.sh` / `weight_convert.py`（那些是自己微调才需要）。

动作空间：**只能 `ee`**。适配器 60 维动作没有关节槽，`joint` 启动即报错。

---

## 2. 环境

| 角色 | conda | torch | 用途 |
| :-- | :-- | :-- | :-- |
| 评估 | `RoboDojo` | 2.7.0 cu128 | Isaac Sim / 评测客户端 |
| 策略 | `mibot`（`MIBOT_CONDA_ENV` 可改） | 2.8.0 cu128 | Xiaomi-1 Policy Server |

两套环境不可混装。

```bash
cd XPolicyLab/policy/Xiaomi_Robotics_1
bash install.sh
conda activate mibot   # 或 install.sh 打印的环境名

conda run -n mibot python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
conda run -n mibot python -c "import mibot; print('mibot ok')"
```

flash-attn 失败则按 `INSTALLATION.md` 手工装，改动记到接手文档，不要动 `RoboDojo` 环境。本机是 4090，一般比笔记本 5070 Ti 好装。

---

## 3. 下载 ckpt

先 `df -h /`。默认放到项目目录（本机无数据盘）。**优先魔搭**，HF 镜像会跳到海外 CDN，慢很多。

```bash
mkdir -p /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1/ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0/last.ckpt/checkpoint
DEST=/home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1/ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0
MS=https://www.modelscope.cn/datasets/RoboDojo-Benchmark/RoboDojo/resolve/master/ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0

wget -c -O "$DEST/config.py" "$MS/config.py"
wget -c -O "$DEST/config.yaml" "$MS/config.yaml"
wget -c -O "$DEST/last.ckpt/checkpoint/mp_rank_00_model_states.pt" \
  "$MS/last.ckpt/checkpoint/mp_rank_00_model_states.pt"
```

官方脚本等价（同样只拉这一份 policy，不要下 ACT）：

```bash
cd /home/ubuntu/GOAI-2026/RoboDojo
bash scripts/RoboDojo/download_ckpt.sh modelscope Xiaomi_Robotics_1
```

HF 仅作备用（易 429 / 走海外 CDN）：

```bash
export HF_ENDPOINT=https://hf-mirror.com
hf download RoboDojo-Benchmark/RoboDojo --repo-type dataset \
  --include "ckpt/RoboDojo/Xiaomi_Robotics_1/*" \
  --local-dir /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1
```

接到 policy 目录（`ckpt_name=Xiaomi_Robotics_1` 时找 `checkpoints/Xiaomi_Robotics_1/`）：

```bash
mkdir -p XPolicyLab/policy/Xiaomi_Robotics_1/checkpoints
ln -sfn /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 \
  XPolicyLab/policy/Xiaomi_Robotics_1/checkpoints/Xiaomi_Robotics_1
```

若 `hf download` 多包了一层 `ckpt/RoboDojo/Xiaomi_Robotics_1/`，把软链或 `deploy.yml` 的 `model_dir` 指到**含 `RoboDojo-sim-arx_x5-ee-0` 的目录**。适配器会在嵌套路径里搜 `config.py`。

验收：

```bash
find /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 -name 'config.py' | head
find /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1 -name 'mp_rank_00_model_states.pt' | head
du -sh /home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1
```

---

## 4. debug（不启 Isaac）

CWD 必须是 policy 目录。参数顺序以 README 为准。

```bash
cd XPolicyLab/policy/Xiaomi_Robotics_1
export EVAL_ENV_TYPE=debug
bash eval.sh RoboDojo stack_bowls Xiaomi_Robotics_1 arx_x5 ee 0 0 0 mibot RoboDojo
```

含义：`bench task ckpt_name env_cfg action_type seed policy_gpu env_gpu policy_conda eval_conda`。

成功：加载 `config.py` + `last.ckpt/`，完成 obs→action 接线。失败先查路径和 `ee`，再查 `mibot`。

---

## 5. 显存（4090 24GB）

ckpt 约 11GB，推理常见 14–18G。**4090 够用。** 与 Isaac 同卡时分进程、一次一个环境，看 `nvidia-smi`。量化不当默认提交路径。

---

## 6. doctor / 单任务 / smoke

在**仓库里跑 `scripts/robodojo.sh` 的根目录**（通常是 `RoboDojo/` 或项目根，以实际脚本位置为准）：

```bash
conda activate RoboDojo

bash scripts/robodojo.sh doctor
# 策略项卡住时可：
# bash scripts/robodojo.sh doctor --skip-isaac --skip-conda --skip-policy

bash scripts/robodojo.sh eval \
  --policy-dir XPolicyLab/policy/Xiaomi_Robotics_1 \
  --task stack_bowls \
  --ckpt Xiaomi_Robotics_1 \
  --policy-env mibot \
  --eval-env RoboDojo \
  --action-type ee \
  --eval-num 1

bash scripts/robodojo.sh smoke \
  --dimension generalization \
  --policy-dir XPolicyLab/policy/Xiaomi_Robotics_1 \
  --ckpt Xiaomi_Robotics_1 \
  --policy-env mibot \
  --eval-env RoboDojo \
  --action-type ee \
  --eval-num 1
```

`--ckpt Xiaomi_Robotics_1` 是**一份**多任务权重，不必再为 12 个任务建 ACT 那种 `policy_last.ckpt` 软链。

验收：每个任务 `eval_result/**/_result.json` 里 **`eval_time >= 1`**。退出码 0 不够。

```bash
find eval_result -name "_result.json" | wc -l
bash scripts/robodojo.sh summarize
```

也可用 policy 目录的 `eval.sh` 做单任务（CWD = `Xiaomi_Robotics_1`，不要设 `EVAL_ENV_TYPE=debug`）：

```bash
cd XPolicyLab/policy/Xiaomi_Robotics_1
bash eval.sh RoboDojo stack_bowls Xiaomi_Robotics_1 arx_x5 ee 0 0 0 mibot RoboDojo
```

分机部署（策略一卡/一机、Isaac 另一机）见 XPolicyLab README 的 Deployment Flow：`setup_eval_policy_server.sh` / `setup_eval_env_client.sh`。

---

## 7. Policy Server 与提交

`deploy.yml` 关键项：`protocol: ws`，`action_type: ee`，`port`（默认 6000），`model_dir` 空则用 `checkpoints/<ckpt_name>/`。

评测端需要公网 WebSocket。申请表填 host 与 port。审核和正式评测期间：

- 保持同一服务在线
- 不更换模型、动作类型、协议行为

xsparkai 申请字段：队伍名、联系人、手机、邮箱、Policy Server host/port（1–8 端点）、策略名（仅字母数字下划线）、**Action Type = ee**。

流程：本地自测 → https://xsparkai.com/goai-2026/apply → 保存评测结果邮件 → https://www.goaihz.com/ 提交作品（代码仓库 + 该邮件）。每队最多 3 次有效成绩，取最高。不要用正式提交调试。

---

## 8. 命令速查

```bash
df -h /

# 停误下的 ACT（确认 PID 后再杀）
ps aux | grep -E "download_ckpt|git-lfs" | grep -v grep

# 策略环境
cd XPolicyLab/policy/Xiaomi_Robotics_1 && bash install.sh

# 权重：优先魔搭（约 11G）
DEST=/home/ubuntu/GOAI-2026/checkpoints/Xiaomi_Robotics_1/ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0
MS=https://www.modelscope.cn/datasets/RoboDojo-Benchmark/RoboDojo/resolve/master/ckpt/RoboDojo/Xiaomi_Robotics_1/RoboDojo-sim-arx_x5-ee-0
mkdir -p "$DEST/last.ckpt/checkpoint"
wget -c -O "$DEST/config.py" "$MS/config.py"
wget -c -O "$DEST/config.yaml" "$MS/config.yaml"
wget -c -O "$DEST/last.ckpt/checkpoint/mp_rank_00_model_states.pt" \
  "$MS/last.ckpt/checkpoint/mp_rank_00_model_states.pt"

# debug
cd XPolicyLab/policy/Xiaomi_Robotics_1
EVAL_ENV_TYPE=debug bash eval.sh RoboDojo stack_bowls Xiaomi_Robotics_1 arx_x5 ee 0 0 0 mibot RoboDojo
```
