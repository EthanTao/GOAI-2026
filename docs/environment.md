# Environment

> 环境快照，最后更新：2026-08-13
> 赛题要求记录的文件（见赛题文档「检查系统环境」章节）。磁盘数字会随下载/评测变化，以 `df -h` 实际为准。

## OS

- Ubuntu 24.04.4 LTS (noble)
- Linux 7.0.0-28-generic x86_64

## CPU

- x86_64，24 核

## RAM

- Total: 30 GiB
- Available: 约 20 GiB

## GPU

- NVIDIA GeForce RTX 5070 Ti Laptop GPU
- Driver Version: 580.173.02

## VRAM

- 12 GB (12227 MiB)

## CUDA

- Driver CUDA Version: 13.0
- 系统 `nvcc` 未装（torch 各环境自带 CUDA 运行时，无需单独装 Toolkit）

## Python

- 系统 Python：3.14.6
- 评估环境 `RoboDojo`：Python 3.11（torch 2.7.0 cu128）

## Conda

- Miniconda 已安装（`/home/ethantao/miniconda3`）
- 环境：
  - `base`
  - `RoboDojo`（评估环境，torch 2.7.0 cu128）
  - 待建：ACT 策略环境（torch 2.4.1，与评估环境隔离）

## Git

- Git: 2.43.0
- Git LFS: 3.4.1

## Disk

- 系统盘 (`/`)：99G 总量，31G 可用（68% 已用）
- 数据盘 (`/mnt/data`)：653G 总量，243G 可用（63% 已用）

## Notes

- GPU 驱动 580.173.02 + CUDA 13.0 可用
- 数据盘 `/mnt/data` 是 Windows D 盘（NTFS），大文件放这里，代码/conda 环境留在 ext4 系统盘（见 `docs/dual_boot_mount_guide.md`）
- 详细环境背景与踩坑见 `docs/接手文档.md`
