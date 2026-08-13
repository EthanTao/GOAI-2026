# 双系统 Ubuntu 挂载 Windows D 盘指南

> 适用场景：Ubuntu + Windows 双系统，Ubuntu 分区空间不足，需要借用 Windows D 盘存放大文件（数据集、模型等）。

---

## 背景

- 磁盘总容量：1 TB SSD
- Ubuntu `/` 分区：100 GB（ext4，系统+代码）
- Windows D 盘：653 GB（NTFS，剩余约 137 GB）
- 目标：把 GOAI 项目的大文件（Assets、数据集、checkpoint、日志）存入 D 盘，代码和 conda 环境留在 ext4

**为什么系统必须装 ext4？** NTFS 不支持 Linux 文件权限和软链接，无法安装 Ubuntu 操作系统。详见附录。

---

## 1. 准备工作

### 1.1 确认分区信息

```bash
lsblk
```

找到 Windows D 盘对应的分区，例如 `/dev/nvme0n1p4`。

### 1.2 确认文件系统

```bash
sudo blkid /dev/nvme0n1p4
```

输出示例：

```
/dev/nvme0n1p4: UUID="5838FF2F38FF0B30" TYPE="ntfs" ...
```

记下 `UUID`，后面配置开机自动挂载需要用到。

### 1.3 安装 ntfs-3g（通常已预装）

```bash
sudo apt install ntfs-3g
```

---

## 2. 挂载 D 盘

### 2.1 创建挂载点

挂载点是 Linux 访问 NTFS 分区的一个"入口目录"：

```bash
sudo mkdir -p /mnt/data
```

`/mnt/data` 可以换成你喜欢的路径，比如 `/media/D`。

### 2.2 临时挂载（重启后失效）

```bash
sudo mount -t ntfs-3g /dev/nvme0n1p4 /mnt/data
```

验证：

```bash
ls /mnt/data
df -h /mnt/data
```

应该能看到 D 盘原有的文件，且容量正确。

### 2.3 设置开机自动挂载

**第一步：备份 fstab**（重要！改坏会导致系统无法启动）

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

**第二步：追加挂载配置**

```bash
echo "UUID=5838FF2F38FF0B30  /mnt/data  ntfs-3g  defaults,uid=1000,gid=1000,dmask=022,fmask=133  0  0" | sudo tee -a /etc/fstab
```

参数说明：

| 参数 | 含义 |
|------|------|
| `UUID=...` | 唯一标识 D 盘，比 `/dev/nvme0n1p4` 更稳定 |
| `/mnt/data` | 挂载点路径 |
| `ntfs-3g` | 文件系统类型（NTFS 的 Linux 驱动） |
| `defaults` | 使用默认挂载选项 |
| `uid=1000,gid=1000` | 文件所有者设为当前用户（避免 root 才能写入） |
| `dmask=022,fmask=133` | 文件夹权限 755，文件权限 644 |
| `0 0` | 不需要 dump 备份，不需要开机 fsck 检查 |

**第三步：测试配置是否正确**

```bash
sudo mount -a
```

如果没有报错，说明 fstab 配置正确。可以重启验证：

```bash
sudo reboot
```

---

## 3. 在项目中使用

### 3.1 创建数据目录

```bash
mkdir -p /mnt/data/GOAI/{Assets,data,checkpoints,logs,results,docs}
```

### 3.2 创建软链接

```bash
cd /home/ethantao/Project/GOAI

ln -s /mnt/data/GOAI/Assets      Assets
ln -s /mnt/data/GOAI/data        data
ln -s /mnt/data/GOAI/checkpoints checkpoints
ln -s /mnt/data/GOAI/logs        logs
ln -s /mnt/data/GOAI/results     results
ln -s /mnt/data/GOAI/docs        docs
```

### 3.3 验证

```bash
ls -la
```

软链接会显示 `l` 标志和 `→` 指向目标路径：

```
lrwxrwxrwx 1 ethantao ethantao  21  Assets -> /mnt/data/GOAI/Assets
```

### 3.4 最终目录结构

```text
/home/ethantao/Project/GOAI/                  (ext4, 代码+环境)
├── GOAI 2026 通用双臂协同操作挑战赛.md
├── Assets/      →  /mnt/data/GOAI/Assets/    (D 盘, 大文件)
├── data/        →  /mnt/data/GOAI/data/      (D 盘, 训练数据)
├── checkpoints/ →  /mnt/data/GOAI/checkpoints/(D 盘, 模型)
├── logs/        →  /mnt/data/GOAI/logs/      (D 盘, 日志)
├── results/     →  /mnt/data/GOAI/results/   (D 盘, 结果)
├── docs/        →  /mnt/data/GOAI/docs/      (D 盘, 文档)
├── RoboDojo/              (待克隆, 放 ext4)
└── XPolicyLab/            (待克隆, 放 ext4)
```

**原则**：代码、conda 环境、Python 包必须放 ext4；数据集、模型、日志放 D 盘。

---

## 4. 如何删除挂载（完全可逆）

删除挂载**不会影响 Windows D 盘的任何原有文件**，只删除软链接和挂载配置。

### 4.1 删除项目中的软链接

```bash
cd /home/ethantao/Project/GOAI
rm Assets checkpoints data docs logs results
```

> ⚠️ `rm 软链接名` 只删除链接本身，不会删除目标目录里的文件。**绝对不要**写成 `rm -rf Assets/`（末尾带斜杠会删掉目标目录内容！）。

### 4.2 卸载分区

```bash
sudo umount /mnt/data
```

如果提示 `target is busy`，说明有程序在使用该目录。关掉相应程序（或退出该目录下的终端），再执行一次。

### 4.3 删除 fstab 中的挂载配置

```bash
sudo nano /etc/fstab
```

找到 D 盘相关的那一行（搜索 `mnt/data`），整行删掉，保存退出（`Ctrl+O` → `Enter` → `Ctrl+X`）。

### 4.4 删除空挂载点

```bash
sudo rmdir /mnt/data
```

### 4.5 完成

此时 D 盘完全恢复原状，Windows 端不会感知到任何变化。

---

## 5. 常见问题

### Q1：挂载时报错 "unclean file system"

```
The disk contains an unclean file system (0, 0).
```

**原因**：Windows 开启了"快速启动"，关机时没有正常卸载 NTFS 分区。

**解决**：

1. 先只读挂载应急：
   ```bash
   sudo mount -t ntfs-3g -o ro /dev/nvme0n1p4 /mnt/data
   ```

2. 回 Windows，关掉快速启动：
   - 控制面板 → 电源选项 → 选择电源按钮的功能 → 取消"启用快速启动"

3. 重启到 Ubuntu，正常挂载即可。

### Q2：挂载后没有写入权限

**检查**：`/etc/fstab` 中是否正确设置了 `uid=1000,gid=1000`。

**验证**：

```bash
id -u    # 应该输出 1000
id -g    # 应该输出 1000
```

### Q3：重启后挂载丢失

`mount` 命令是临时的，重启后失效。必须配置 `/etc/fstab` 才能开机自动挂载。参见第 2.3 节。

### Q4：NTFS 性能差怎么办？

NTFS 对小文件（pip/conda 安装）性能确实差，但存放大文件（数据集、模型）时完全够用。这就是为什么只把数据放 D 盘，代码和 Python 环境仍放在 ext4。

### Q5：这样会损坏 Windows 文件吗？

不会。只要：
1. 关掉 Windows 快速启动
2. 只在 `/mnt/data/GOAI/` 自己的目录中操作
3. 不随意 `rm -rf` Windows 系统目录

---

## 附录：为什么 Ubuntu 系统必须装在 ext4 上？

| 特性 | ext4 | NTFS |
|------|------|------|
| Linux 权限 (rwx) | ✅ 原生支持 | ❌ 不支持 |
| 软链接 (symlink) | ✅ 完美支持 | ❌ 支持很差 |
| 小文件性能 | ✅ 极快 | ❌ 慢 |
| 执行可执行文件 | ✅ 原生 | ❌ 几乎不能 |
| 包管理器 (apt/pip) | ✅ 优化过 | ❌ 可能出错 |

简言之：NTFS 能"存东西"但不能"跑系统"，就像仓库不能当卧室住人。
