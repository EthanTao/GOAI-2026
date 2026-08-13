# GOAI 2026 Baseline 冲刺实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在初赛截止前用官方 ACT checkpoint 跑通 24 配置本地评测、部署 Policy Server 并完成一次有效成绩提交。

**Architecture:** 全程本地执行（RoboDojo 仿真 + XPolicyLab/ACT 策略），不训练、不改 benchmark、不碰小米 VLA。先解决 ckpt 命名与映射两个已知坑，再 doctor → smoke → 验收 → 本地评测 → 部署公网 Policy Server → 提交。

**Tech Stack:** RoboDojo（Isaac Sim/IsaacLab，torch 2.7.0 cu128）、XPolicyLab/ACT（torch 2.4.1，独立 conda 环境）、bash、git、WebSocket/TLS。

## Global Constraints

- 禁止修改官方 benchmark（task 定义 / reward / randomization / 评分 / configuration）。
- 禁止训练 / 微调任何模型（ACT、小米 VLA 均不）。
- 禁止下载 240G 训练数据（`data/hdf5`）。
- 禁止碰小米 VLA（完全冻结至出分后）。
- 评估环境 `RoboDojo`（torch 2.7.0 cu128）与策略环境（torch 2.4.1）必须 conda 隔离，不可混装。
- 每步用 git 记录，保证可回滚。
- 磁盘 `/mnt/data` 剩余约 137G，任何大文件操作前先 `df -h /mnt/data`。
- smoke 验收标准：每个任务 `eval_result/*/_result.json` 的 `eval_time >= 1`（仅退出码 0 不算通过）。
- 参考文档（执行前必读）：`docs/接手文档.md`（关键坑）、根目录 `GOAI 2026 通用双臂协同操作挑战赛.md`、`docs/superpowers/specs/2026-08-13-goai-baseline-sprint-design.md`。

## 文件结构

- 创建：`scripts/link_ckpt.sh`（为 12 个任务建 `policy_last.ckpt` 软链的脚本）
- 创建：`results/smoke_test.csv`（smoke 结果表）
- 创建：`results/baseline.csv`（本地评测成功率表）
- 创建：`docs/xpolicylab_dataflow.md`（ckpt 映射结论记录）
- 修改：`docs/接手文档.md`（更新 docs 符号链接描述 + 补充本轮踩坑）
- 参考（只读）：`scripts/robodojo.sh`、`scripts/internal/run_policy_eval.sh`、`XPolicyLab/policy/ACT/detr/act_policy.py`

---

## Task 1: 确认下载完成与磁盘评估

**Files:**
- 无新建/修改（只读检查）

**验证标准:** 后台下载进程全部结束；Assets 约 35G、官方 ckpt 约 34G 均到位；`/mnt/data` 剩余空间清楚。

- [ ] **Step 1: 检查下载进程是否结束**

```bash
ps aux | grep -E "modelscope|git-lfs|huggingface" | grep -v grep
```

预期：无输出（进程已结束）。若有进程仍在跑，记下 PID 并跳至 Task 1 Step 4 等待。

- [ ] **Step 2: 检查 Assets 与 ckpt 体积**

```bash
du -sh /mnt/data/GOAI/Assets 2>/dev/null
du -sh /mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo 2>/dev/null
```

预期：Assets 约 35G；ckpt 缓存约 34G。若明显偏小（如 Assets 只有几百 MB），说明下载未完成或失败，回到 `docs/接手文档.md` 第二节的下载命令重下。

- [ ] **Step 3: 评估磁盘剩余空间**

```bash
df -h /mnt/data
```

预期：剩余 > 50G 才够评测中间产物。若 < 30G，记录到风险清单，后续 Task 6 前清理 ckpt 中未用 seed/task。

- [ ] **Step 4: 确认官方 ckpt 目录结构**

```bash
find /mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo -name "*.ckpt" | head -20
```

预期：能看到 `.../act-RoboDojo-<task>/arx_x5-100-joint/policy_epoch_6000_seed_0.ckpt` 这类路径。记下实际的绝对根路径（后续软链要用）。

- [ ] **Step 5: 记录环境快照到 git（若有变化）**

```bash
git status
```

预期：无意外未跟踪大文件。若 `.gitignore` 未覆盖某个大文件被 git 识别，先补 `.gitignore` 再继续。

---

## Task 2: 建 ACT 策略环境（torch 2.4.1，隔离）

**Files:**
- 无文件修改（conda 环境安装）

**验证标准:** 新 conda 环境存在，`torch.__version__` 为 2.4.1，可 `import` ACT 策略依赖。

- [ ] **Step 1: 确认 XPolicyLab/ACT 的 install 脚本位置**

```bash
ls XPolicyLab/policy/ACT/
cat XPolicyLab/policy/ACT/README.md | head -60
```

预期：存在 `install.sh`（或等价安装脚本）；README 说明环境名与 torch 版本要求。

- [ ] **Step 2: 运行安装脚本建环境**

```bash
bash XPolicyLab/policy/ACT/install.sh
```

预期：脚本创建 conda 环境并安装依赖。**不要**改动 `RoboDojo` 环境。

- [ ] **Step 3: 确认新环境名与 torch 版本**

```bash
conda env list
conda run -n <策略环境名> python -c "import torch; print(torch.__version__)"
```

预期：torch 打印 `2.4.1`（或 README 指定的版本）。若为 2.7.x，说明装错环境，回到 Step 2 检查。

- [ ] **Step 4: 验证能 import ACT 策略**

```bash
conda run -n <策略环境名> python -c "import sys; sys.path.insert(0, 'XPolicyLab/policy/ACT'); import detr.act_policy; print('ok')"
```

预期：打印 `ok`，无 ImportError。

> 注：策略环境名（如 `act_policy`）确定后，后面 Task 6 的 `--policy-env` 参数统一用它。

---

## Task 3: 修复 ckpt 命名（符号链接）

**Files:**
- 创建：`scripts/link_ckpt.sh`

**验证标准:** 12 个 generalization 任务的 ckpt 目录里都有 `policy_last.ckpt -> policy_epoch_6000_seed_0.ckpt` 软链。

- [ ] **Step 1: 确认 ACT 推理代码认的 ckpt 名**

```bash
sed -n '130,160p' XPolicyLab/policy/ACT/detr/act_policy.py
```

预期：看到 `ckpt_path = os.path.join(ckpt_dir, "policy_last.ckpt")` 且缺文件会 `FileNotFoundError`。这确认了软链目标名必须是 `policy_last.ckpt`。

- [ ] **Step 2: 确认 ckpt 根路径与 12 个任务目录名**

```bash
ls /mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo/ckpt/RoboDojo/ACT/ 2>/dev/null
```

预期：看到 `act-RoboDojo-stack_bowls`、`act-RoboDojo-push_T`、… 等 12 个目录（任务名见设计文档第 4 节）。记下完整根路径记为 `$CKPT_ROOT`。

- [ ] **Step 3: 写建软链脚本**

```bash
mkdir -p scripts
cat > scripts/link_ckpt.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
# 为 12 个 generalization 任务的官方 ckpt 建 policy_last.ckpt 软链
# 用法: CKPT_ROOT=<官方ckpt根路径> bash scripts/link_ckpt.sh
CKPT_ROOT="${CKPT_ROOT:?请设置 CKPT_ROOT 环境变量}"

TASKS=(stack_bowls push_T pack_objects_into_box fold_clothes hang_mugs
       sweep_blocks pour_liquid_into_cup make_toast arrange_largest_number
       sort_nesting_dolls_by_size store_laptop_and_headphones stack_blocks)

for t in "${TASKS[@]}"; do
  d="${CKPT_ROOT}/act-RoboDojo-${t}/arx_x5-100-joint"
  if [ ! -d "$d" ]; then
    echo "⚠️  缺少目录: $d" >&2
    continue
  fi
  # 默认用 seed_0；若官方指定其他 seed 再改
  src="${d}/policy_epoch_6000_seed_0.ckpt"
  if [ ! -f "$src" ]; then
    echo "⚠️  缺少源文件: $src" >&2
    continue
  fi
  ln -sfn "policy_epoch_6000_seed_0.ckpt" "${d}/policy_last.ckpt"
  echo "✅ ${d}/policy_last.ckpt -> policy_epoch_6000_seed_0.ckpt"
done
EOF
chmod +x scripts/link_ckpt.sh
```

- [ ] **Step 4: 运行脚本建软链**

```bash
CKPT_ROOT=/mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo/ckpt/RoboDojo/ACT \
  bash scripts/link_ckpt.sh
```

预期：12 行 `✅ ...policy_last.ckpt`，无 `⚠️`。若有 `⚠️ 缺少目录`，说明该任务未下载，回到下载步骤补齐。

- [ ] **Step 5: 抽查软链有效性**

```bash
ls -l /mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo/ckpt/RoboDojo/ACT/act-RoboDojo-stack_bowls/arx_x5-100-joint/
```

预期：`policy_last.ckpt -> policy_epoch_6000_seed_0.ckpt` 且源文件存在。

- [ ] **Step 6: Commit**

```bash
git add scripts/link_ckpt.sh
git commit -m "feat: 添加官方 ckpt 命名修复软链脚本"
```

---

## Task 4: 解决 per-task ckpt 映射（探索型）

**Files:**
- 修改（可能）：`scripts/robodojo.sh` 或运行时传参方式（**不改 benchmark 逻辑**）
- 创建：`docs/xpolicylab_dataflow.md`（记录结论）

**验证标准:** 明确 smoke 如何为不同任务选不同 ckpt；单任务 dry-run 能正确指向对应 ckpt。

- [ ] **Step 1: 读 smoke 入口，找 --ckpt 与 --action-type 如何被使用**

```bash
grep -nE "ckpt|action[-_]type|ACTION_TYPE" scripts/robodojo.sh | head -60
```

预期：定位 smoke 命令里 `--ckpt` 传给哪个内部脚本；同时确认 `--action-type` 的合法取值（`ee` / `joint`）。

- [ ] **Step 2: 读内部评测脚本的 ckpt 拼接逻辑**

```bash
grep -rn "ckpt" scripts/internal/run_policy_eval.sh 2>/dev/null | head -40
# 若文件不在，用:
ls scripts/internal/ 2>/dev/null
```

预期：找到 ckpt 路径是如何由 `--ckpt` + 任务名（或统一名）拼出来的。**重点判断**：脚本是否有按 `task` 名拼 `act-RoboDojo-<task>` 的逻辑。

- [ ] **Step 3: 根据结论选一条路径执行**

- 情况 A：脚本**有**按 task 拼 ckpt 的逻辑 → 记录结论到 `docs/xpolicylab_dataflow.md`，Task 4 完成。
- 情况 B：脚本**无**按 task 拼逻辑 → 用单任务 `eval` 逐个任务传 `--ckpt act-RoboDojo-<task>`（见 Task 6 备选命令）。
- 情况 C：无法确定 → 用单任务 `--dry-run` 验证（Step 4）。

- [ ] **Step 3b: 确认 action type（`ee` 还是 `joint`）**

```bash
grep -rniE "action.type|action_type" XPolicyLab/policy/ACT/ --include="*.py" --include="*.sh" --include="*.md" | head -20
ls /mnt/data/GOAI/.cache/robodojo_ckpt_huggingface_repo/ckpt/RoboDojo/ACT/act-RoboDojo-stack_bowls/
```

预期：从 ACT 配置/文档，或 ckpt 目录名 `arx_x5-100-joint`（`-joint` 暗示 joint 动作空间）确定官方 ckpt 用的 action type。以**官方 ckpt 实际使用的类型**为准，记录为 `$ACTION_TYPE`，Task 6/8 统一使用。若 `ee` 与 `joint` 存疑，以单任务真跑（Task 6 Step 1）能否正确执行来最终判定。

- [ ] **Step 4: 单任务 dry-run 验证 ckpt 能对上（不跑 Isaac/策略）**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh eval --policy-dir XPolicyLab/policy/ACT \
  --task stack_bowls --ckpt act-RoboDojo-stack_bowls \
  --policy-env <策略环境名> --dry-run
```

预期：流程跑通、无 `FileNotFoundError: policy_last.ckpt`。若报缺 ckpt，说明 `--ckpt` 拼接方式与假设不符，回到 Step 2 继续读。

- [ ] **Step 5: 记录结论并 commit**

把结论（含实际 ckpt 拼接规则、采用的路径）写入 `docs/xpolicylab_dataflow.md`，然后：

```bash
git add docs/xpolicylab_dataflow.md
git commit -m "docs: 记录 per-task ckpt 映射结论"
```

---

## Task 5: doctor 体检

**Files:**
- 无（只读检查）

**验证标准:** doctor 通过（或每个报错项都已定位）。

- [ ] **Step 1: 先跑全量 doctor**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh doctor
```

预期：逐项 PASS/FAIL。记录所有 FAIL 项。

- [ ] **Step 2: 逐个 FAIL 项定位，不盲目改依赖**

参考 `docs/接手文档.md` 第五节。每修一个，重跑对应 doctor 子项验证。涉及 CUDA/conda/torch 版本冲突的，先查官方 issue，再最小改动。

- [ ] **Step 3: 若医生项与策略环境耦合导致卡住，先跳过策略项**

```bash
bash scripts/robodojo.sh doctor --skip-policy
```

预期：策略项跳过不影响环境/资产检查，策略环境问题留给 Task 6 smoke 时暴露。

- [ ] **Step 4: 确认 doctor 无阻塞项后 commit（若有配置改动）**

```bash
git status && git add -A && git commit -m "chore: doctor 检查通过" 
```

> 若 doctor 仅提示下载未完成等已知问题，不产生文件改动，可跳过 commit。

---

## Task 6: smoke 测试（24 配置）

**Files:**
- 创建：`results/smoke_test.csv`

**验证标准:** 24 个配置全部跑完退出码 0（是否 `eval_time>=1` 由 Task 7 单独验收）。

- [ ] **Step 1: 先单任务真跑测单集耗时**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh eval --policy-dir XPolicyLab/policy/ACT \
  --task stack_bowls --ckpt act-RoboDojo-stack_bowls \
  --policy-env <策略环境名> --eval-num 1
```

预期：跑完且无异常。记录单集耗时 → 估算 24 配置总时长。

- [ ] **Step 2: 跑全量 smoke（后台 + 日志）**

```bash
mkdir -p logs/smoke
nohup bash scripts/robodojo.sh smoke \
  --dimension generalization \
  --policy-dir XPolicyLab/policy/ACT \
  --ckpt <统一CKPT或按Task4结论> \
  --policy-env <策略环境名> \
  --eval-env RoboDojo \
  --action-type "$ACTION_TYPE" \
  --eval-num 1 \
  > logs/smoke/smoke_$(date +%Y%m%d_%H%M%S).log 2>&1 &
```

预期：后台进程启动，`tail -f` 能看到逐任务输出。若 Task 4 结论是"逐任务 eval"，则改为循环脚本对 12 任务 × 2 变体各跑 `--eval-num 1`。

- [ ] **Step 3: 监控进度并确认 24 配置都跑到**

```bash
tail -f logs/smoke/smoke_*.log
find eval_result -name "_result.json" | wc -l
```

预期：`_result.json` 数量最终为 24。

- [ ] **Step 4: 生成 smoke_test.csv**

按设计文档字段（task/variant/success/episode/error/runtime/notes）汇总 `eval_result/` 下各 `_result.json`。可先用 `bash scripts/robodojo.sh summarize` 生成 markdown 再转 csv。

- [ ] **Step 5: Commit 结果**

```bash
git add results/smoke_test.csv
git commit -m "test: smoke 测试 24 配置结果"
```

---

## Task 7: 验收 eval_time >= 1

**Files:**
- 修改：`results/smoke_test.csv`（补验收列）

**验证标准:** 24 个 `_result.json` 的 `eval_time` 字段均 `>= 1`。

- [ ] **Step 1: 批量检查 eval_time**

```bash
for f in $(find eval_result -name "_result.json"); do
  t=$(python -c "import json;print(json.load(open('$f')).get('eval_time',0))" 2>/dev/null)
  echo "$t  $f"
done | sort -n
```

预期：所有值 `>= 1`。任何 `0` 或缺失的，记为 FAIL。

- [ ] **Step 2: 处理 FAIL 项**

对 `eval_time < 1` 的任务：看对应日志区分是"环境问题"还是"策略问题"（接手文档第 15 节字段要求）。环境问题→修环境重跑该任务；策略问题→记录但**不**改模型（本阶段禁训）。

- [ ] **Step 3: 更新 smoke_test.csv 并 commit**

```bash
git add results/smoke_test.csv
git commit -m "test: 验收 24 配置 eval_time>=1"
```

---

## Task 8: 本地完整评测（可降级）

**Files:**
- 创建：`results/baseline.csv`

**验证标准:** 拿到真实成功率（全量 24 配置，或按设计文档决策 3 降级为 2~3 代表任务满测）。

- [ ] **Step 1: 根据 Task 6 Step 1 的单集耗时 + 剩余时间，决定全量 or 降级**

- 时间够 → 全量：12 任务 × 2 变体 × 满 episode 数。
- 时间紧 → 降级：挑 3 个代表性任务（`stack_bowls`、`push_T`、`fold_clothes`）跑满 episode，其余留 smoke 结果。

- [ ] **Step 2: 跑满 episode 数**

```bash
conda activate RoboDojo
bash scripts/robodojo.sh eval --policy-dir XPolicyLab/policy/ACT \
  --task <task> --ckpt act-RoboDojo-<task> \
  --policy-env <策略环境名> --eval-num <满episode数>
```

预期：逐任务跑出成功率。

- [ ] **Step 3: 汇总到 results/baseline.csv 并 commit**

```bash
git add results/baseline.csv
git commit -m "test: 本地评测成功率基线"
```

---

## Task 9: 部署 Policy Server（公网 wss://）

**Files:**
- 创建：部署说明（`docs/policy_server_deploy.md`）

**验证标准:** 公网 `wss://host:port` 可访问，官方评测服务器能连入调用策略。

- [ ] **Step 1: 确认是否有云服务器/公网 IP**

有 → 走 Step 2A；无 → 走 Step 2B（内网穿透）。

- [ ] **Step 2A: 云服务器部署**

1. 在云服务器上装策略环境 + 拷模型权重（ckpt）。
2. 开放端口 + 安全组 + 防火墙。
3. 配 TLS 证书（Let's Encrypt）。
4. 启动 Policy Server，监听 `wss://<公网IP或域名>:<port>`。

- [ ] **Step 2B: 内网穿透（Cloudflare Tunnel 优先）**

```bash
# 安装 cloudflared 后，把本地 Policy Server 端口映射到公网
cloudflared tunnel --url http://localhost:<本地Policy端口>
```

预期：得到 `https://xxxx.trycloudflare.com`，即 `wss://xxxx.trycloudflare.com`。

- [ ] **Step 3: 验证公网可访问**

用公网侧（或手机流量）测试 `wss://` 握手成功、能返回一次策略动作。

- [ ] **Step 4: 记录 host/port/部署方式到 docs/policy_server_deploy.md 并 commit**

```bash
git add docs/policy_server_deploy.md
git commit -m "infra: Policy Server 部署说明"
```

---

## Task 10: 提交申请

**Files:**
- 无（填写线上表单）

**验证标准:** 提交成功，拿到官方回执，且提交期间策略名称/action type 与本地一致。

- [ ] **Step 1: 核对提交前纪律（设计文档/主任务文档第 22 节）**

确认：模型固定、action type 固定、Policy Server 不重启不改、本地=提交=服务器三者一致。

- [ ] **Step 2: 填申请表**

字段：队伍名称、联系人、手机号、邮箱、Policy Server Host、Policy Server Port、策略名称（只允许字母数字下划线，如 `ACT_baseline_v1`）、Action Type。

- [ ] **Step 3: 提交并保存回执**

保存官方回执邮件/截图到 `docs/`，并 commit。

```bash
git add docs/
git commit -m "docs: 保存初赛提交回执"
```

---

## Task 11: 收尾（更新接手文档 + 归档）

**Files:**
- 修改：`docs/接手文档.md`

**验证标准:** 接手文档反映最新状态（docs 已非 symlink、ckpt 映射结论、提交结果、本轮踩坑）。

- [ ] **Step 1: 更新接手文档的符号链接列表**

把第四节"符号链接"里 `docs -> /mnt/data/GOAI/docs` 改为"`docs/` 已是真实目录（2026-08-13 为版本控制文档而改）"。

- [ ] **Step 2: 补充本轮结论**

把 ckpt 映射结论、策略环境名、action type、提交结果、新增踩坑写入接手文档对应章节。

- [ ] **Step 3: 清理数据盘旧 docs（可选，需确认）**

```bash
# 确认数据盘旧 docs 已无引用后，可删（危险操作，先确认再执行）
ls -la /mnt/data/GOAI/docs/
```

**不要**贸然 `rm`；确认两份内容一致且无进程引用后再删，或保留作备份。

- [ ] **Step 4: 最终 commit**

```bash
git add -A
git commit -m "docs: 更新接手文档至冲刺后状态"
```
