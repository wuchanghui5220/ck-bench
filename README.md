# ck-bench - GPU RDMA 网络性能测试工具

> 基于 NVIDIA HPC-X ClusterKit 的 GPU 网络自动化基准测试工具

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Shell](https://img.shields.io/badge/shell-bash-green.svg)](https://www.gnu.org/software/bash/)
[![Version](https://img.shields.io/badge/version-2.0-green.svg)](CHANGELOG.md)

## 目录

- [功能特性](#功能特性)
- [系统要求](#系统要求)
- [快速开始](#快速开始)
- [使用指南](#使用指南)
  - [基础测试](#基础测试)
  - [拓扑检测](#拓扑检测)
  - [健康检查](#健康检查)
  - [Rail-by-Rail 测试](#rail-by-rail-测试)
- [参数说明](#参数说明)
- [网络架构](#网络架构)
- [输出说明](#输出说明)
- [故障排查](#故障排查)
- [最佳实践](#最佳实践)
- [高级功能](#高级功能)

---

## 功能特性

### 核心功能

- ✅ **自动化测试** - 自动检测 GPU 关联的 HCA 设备（基于 `nvidia-smi topo -m`）
- ✅ **双网络支持** - 支持 InfiniBand 和 RoCE 两种网络模式
- ✅ **拓扑映射** - GPU-NIC-Switch 完整拓扑关系展示
- ✅ **逐轨道测试** - Rail-by-Rail 独立测试每个 HCA，快速定位问题网卡
- ✅ **健康检查** - 网卡状态、链路质量、端口错误计数器全面检查
- ✅ **并行执行** - 最大 32 台服务器并发执行，支持 256+ 节点大规模集群
- ✅ **彩色输出** - 10 色调色板，不同服务器使用不同颜色，便于多节点分析
- ✅ **CSV 导出** - 支持 CSV 格式输出，便于数据分析和可视化
- ✅ **超时保护** - 10 秒连接超时，自动跳过无响应节点，避免测试卡住

### 性能优化

- **智能超时** - SSH/SCP 10 秒连接超时，命令级超时保护
- **并发收集** - 拓扑信息并行采集，大幅提升大规模集群测试效率
- **批量处理** - IB 模式批量查询，避免 Subnet Manager 过载
- **内存缓冲** - 彩色矩阵输出使用内存缓冲，无闪烁显示

### 网络诊断

- **链路状态** - 实时显示 Up/Down 状态，Down 状态网卡也会显示
- **端口错误** - 物理错误、链路降级、错误恢复计数器（mlxlink）
- **LLDP 检测** - RoCE 模式下自动发现交换机拓扑
- **IB 拓扑追踪** - 基于 ibtracert 的 InfiniBand 网络路径分析
- **设备优先级** - 自动优先选择 mlx5_gdr_* 设备（忽略 bond 设备）

---

## 系统要求

### 硬件要求

- **GPU 服务器**：NVIDIA GPU + Mellanox/NVIDIA HCA
- **网络**：InfiniBand 或 RoCE 网络
- **推荐配置**：ConnectX-7 及以上，支持 GPUDirect RDMA

### 软件依赖

```bash
# CPU服务器免密GPU节点
- CPU 免密➡️ GPU
- GPU 之间全免密
# 必需组件
- NVIDIA HPC-X (包含 ClusterKit)
- NVIDIA Driver & CUDA
- Mellanox OFED (MLNX_OFED 或 DOCA OFED)
- GPU 节点安装 numactl

# IB 模式额外要求
- opensm (Subnet Manager)
- ibutils (ibtracert, sminfo, ibstat)

# RoCE 模式额外要求
- lldpd (LLDP daemon，用于交换机发现)

# 健康检查要求
- mstflint (mlxlink 工具, 版本要求：mlxlink, mft 4.30.1-113)
```

### 运行环境

脚本支持三种运行环境，自动识别：

| 环境 | 识别条件 | 执行方式 |
|------|----------|----------|
| **GPU 服务器** | 有 `nvidia-smi` 且可执行 | 本地直接执行 |
| **CPU 服务器** | Linux 系统无 GPU | SSH 到 GPU 节点 |
| **Mac 客户端** | macOS 系统 | 通过 CPU 服务器跳板 |

---

## 快速开始

### 1. 安装部署

```bash
# 克隆仓库
git clone https://github.com/wuchanghui5220/ck-bench.git
cd ck-bench

# 配置 hostfile（每行一个 IP 地址或主机名）
cat > hostfile.txt << EOF
10.0.10.1
10.0.10.2
10.0.10.3
EOF

# 确保 GPU节点 HPC-X 已安装
export HPCX_HOME=/opt/hpcx-v2.25.1-gcc-doca_ofed-ubuntu22.04-cuda12-x86_64

# 赋予执行权限
chmod +x ck-bench.sh
```

### 2. 快速测试

```bash
# 自动检测 HCA，运行基准测试
./ck-bench.sh --auto-hca -G -cx7

# 查看 GPU-网卡-交换机拓扑
./ck-bench.sh --check-topology

# 检查网络健康状态
./ck-bench.sh --check-health-only -f hostfile.txt
```

### 3. 生产环境完整诊断

```bash
# Rail-by-Rail 完整诊断（推荐）
./ck-bench.sh --rbr -G -cx7 --check-topology --output-csv

# RoCE 网络拓扑检测
./ck-bench.sh --network-mode roce --check-topology -f hostfile.txt

# 端口健康检查（包含错误计数器）
./ck-bench.sh --network-mode roce --check-health-only -f hostfile.txt
```

---

## 使用指南

### 基础测试

#### 自动检测 HCA 测试

```bash
# ConnectX-7 优化测试（4 QPs）
./ck-bench.sh --auto-hca -G -cx7

# 使用自定义 hostfile
./ck-bench.sh -f my_hosts.txt --auto-hca -G -cx7

# 指定 HPC-X 路径
./ck-bench.sh -r /opt/hpcx --auto-hca -G -cx7

# 每节点 2 个进程
./ck-bench.sh --auto-hca -G -cx7 -p 2
```

#### 手动指定 HCA

```bash
# 单个 HCA
./ck-bench.sh -d "mlx5_0:1" -G -cx7

# 多个 HCA
./ck-bench.sh -d "mlx5_0:1,mlx5_1:1,mlx5_2:1,mlx5_3:1" -G -cx7
```

#### 压力测试

```bash
# 3 分钟压力测试
./ck-bench.sh --auto-hca -G -cx7 -z 3

# 30 分钟长时间测试
./ck-bench.sh --auto-hca -G -cx7 -z 30

# 注意：-z 不能与 --rbr 同时使用
```

---

### 拓扑检测

#### IB 模式拓扑

```bash
# 基础拓扑检查（默认使用 mlx5_0 查询 SM）
./ck-bench.sh --check-topology

# 指定查询设备（多子网环境必需）
./ck-bench.sh --check-topology --Ca mlx5_1

# 指定 hostfile
./ck-bench.sh --check-topology -f my_hosts.txt
```

**输出示例：**
```
==========================================
GPU-NIC-Switch Topology Mapping
==========================================

Host            GPU    NIC        Port       Switch
----------------------------------------------------------------------
GPU-1           GPU0   mlx5_0     Port 33    Compute-SU1-Leaf01-A03-40U
GPU-1           GPU1   mlx5_1     Port 33    Compute-SU1-Leaf02-A03-38U
GPU-1           GPU2   mlx5_2     Port 33    Compute-SU1-Leaf03-A03-36U
GPU-1           GPU3   mlx5_3     Port 33    Compute-SU1-Leaf04-A03-34U
GPU-1           GPU4   mlx5_6     Port 33    Compute-SU1-Leaf05-A03-32U
GPU-1           GPU5   mlx5_7     Port 33    Compute-SU1-Leaf06-A03-30U
GPU-1           GPU6   mlx5_8     Port 33    Compute-SU1-Leaf07-A03-28U
GPU-1           GPU7   mlx5_9     Port 33    Compute-SU1-Leaf08-A03-26U
==========================================
```

#### RoCE 模式拓扑

```bash
# LLDP 拓扑检测
./ck-bench.sh --network-mode roce --check-topology

# 过滤特定 HCA（使用环境变量）
export HCA_LIST="mlx5_gdr_1,mlx5_gdr_2,mlx5_gdr_3,mlx5_gdr_4"
./ck-bench.sh --network-mode roce --check-topology --hca_list $HCA_LIST

# 大规模集群（32 并发，支持 256+ 节点）
./ck-bench.sh --network-mode roce --check-topology -f all_256_nodes.txt
```

**输出示例（彩色显示）：**
```
==========================================
GPU-NIC-Switch Topology Mapping (RoCE)
==========================================
Host            GPU    HCA        NIC            Port                       Switch
--------------------------------------------------------------------------------------
gpu-1           GPU0   mlx5_gdr_1 enp26s0np0     FHGigabitEthernet 0/33     B11-44U-C-Leaf-007
gpu-1           GPU1   mlx5_gdr_2 enp60s0np0     FHGigabitEthernet 0/33     C06-44U-C-Leaf-008
gpu-1           GPU2   mlx5_gdr_3 enp77s0np0     FHGigabitEthernet 0/33     B10-44U-C-Leaf-006
gpu-1           GPU3   mlx5_gdr_4 enp94s0np0     FHGigabitEthernet 0/33     B09-44U-C-Leaf-005
gpu-1           GPU4   mlx5_gdr_5 enp156s0np0    FHGigabitEthernet 0/33     B03-44U-C-Leaf-001
gpu-1           GPU5   mlx5_gdr_6 enp188s0np0    FHGigabitEthernet 0/33     B04-44U-C-Leaf-002
gpu-1           GPU6   mlx5_gdr_7 enp204s0np0    FHGigabitEthernet 0/33     B05-44U-C-Leaf-003
gpu-1           GPU7   mlx5_gdr_8 enp220s0np0    FHGigabitEthernet 0/33     B08-44U-C-Leaf-004
==========================================
```

**拓扑错误码：**

| 错误码 | 说明 | 可能原因 |
|--------|------|----------|
| `LINK_DOWN` | 网卡链路 Down | 光模块故障、线缆问题、端口禁用 |
| `NO_LLDP` | 无 LLDP 邻居信息 | LLDP 未启用、交换机不支持 |
| `ROUTE_ERR` | IB 路由错误 | ibtracert 失败、网络不通 |
| `PARSE_ERR` | 交换机信息解析失败 | 输出格式异常 |
| `NO_LID` | 获取 LID 失败 | ibstat/sminfo 错误、SM 未运行 |

---

### 健康检查

#### 快速健康检查

```bash
# IB 模式健康检查
./ck-bench.sh --check-health-only -f hostfile.txt

# RoCE 模式健康检查（包含端口错误计数器）
./ck-bench.sh --network-mode roce --check-health-only -f hostfile.txt

# 静默模式（仅显示异常）
./ck-bench.sh --network-mode roce --check-health-only -q -f hostfile.txt
```

#### 检查项目

1. **网卡链路状态**
   - Up/Down 状态
   - 链路速率（25G/100G/200G/400G）
   - 主动/被动状态

2. **端口错误计数器**（mlxlink，RoCE 模式）
   - Effective Physical Errors（物理错误）
   - Link Down Counter（链路降级次数）
   - Link Error Recovery Counter（错误恢复次数）

3. **节点可达性**
   - SSH 连接状态
   - 命令执行超时检测

**输出示例：**
```
root@cpu:/x# HCA_LIST='mlx5_gdr_1,mlx5_gdr_2,mlx5_gdr_3,mlx5_gdr_4,mlx5_gdr_5,mlx5_gdr_6,mlx5_gdr_7,mlx5_gdr_8'
root@cpu:/x# ./ck-bench.sh --network-mode roce --check-health-only -f hostfile.txt.4  --hca_list $HCA_LIST

==========================================
Health Check Only Mode (Single Check)
==========================================

Checking health status on all nodes (no retry)...

Checking 4 nodes...

==========================================
Health Check Results: 4/4 nodes healthy
==========================================
----------------------------------------
Host            SSH    IB         GPU
----------------------------------------
10.20.2.33      ✓      ✓ (8)      ✓
10.20.2.34      ✓      ✓ (8)      ✓
10.20.2.35      ✓      ✓ (8)      ✓
10.20.2.36      ✓      ✓ (8)      ✓
----------------------------------------


==========================================
Checking PCIe Status...
HCA Filter: mlx5_gdr_1,mlx5_gdr_2,mlx5_gdr_3,mlx5_gdr_4,mlx5_gdr_5,mlx5_gdr_6,mlx5_gdr_7,mlx5_gdr_8
==========================================

---------------------------------------------------------------
Host            Interfaces    Status    Details
---------------------------------------------------------------
10.20.2.33      8             ✓ PASS  32GT/s/x16/RX:0/TX:0/PASS
10.20.2.34      8             ✓ PASS  32GT/s/x16/RX:0/TX:0/PASS
10.20.2.35      8             ✓ PASS  32GT/s/x16/RX:0/TX:0/PASS
10.20.2.36      8             ✓ PASS  32GT/s/x16/RX:0/TX:0/PASS
---------------------------------------------------------------

✓ 所有节点 PCIe 状态正常



==========================================
Checking Port Error Counters (mlxlink)...
HCA Filter: mlx5_gdr_1,mlx5_gdr_2,mlx5_gdr_3,mlx5_gdr_4,mlx5_gdr_5,mlx5_gdr_6,mlx5_gdr_7,mlx5_gdr_8
==========================================

-----------------------------------------------------------------------
Host            HCA         PhyErr    LinkDown  ErrRecov  Status
-----------------------------------------------------------------------
10.20.2.33      mlx5_gdr_1  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_2  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_3  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_4  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_5  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_6  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_7  0         0         0         ✓ PASS
10.20.2.33      mlx5_gdr_8  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_1  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_2  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_3  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_4  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_5  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_6  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_7  0         0         0         ✓ PASS
10.20.2.34      mlx5_gdr_8  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_1  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_2  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_3  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_4  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_5  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_6  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_7  0         0         0         ✓ PASS
10.20.2.35      mlx5_gdr_8  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_1  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_2  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_3  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_4  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_5  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_6  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_7  0         0         0         ✓ PASS
10.20.2.36      mlx5_gdr_8  0         0         0         ✓ PASS
-----------------------------------------------------------------------

✓ 所有节点端口误码正常（全部为0）


==========================================
✓ All nodes healthy!
==========================================
Logs saved to:
  - /x/results/health_check_20251216_184619.log
  - /x/results/health_check.csv
```

---

### Rail-by-Rail 测试

#### 什么是 Rail-by-Rail？

Rail-by-Rail（逐轨道）测试模式会**独立测试每个 HCA**，生成详细的性能报告和汇总表。

**优势：**
- ✅ 快速定位故障 HCA
- ✅ 对比不同 HCA 的性能差异
- ✅ 验证每条 GPU-NIC 通路
- ✅ 生成 CSV 汇总报告
- ✅ 即时显示彩色矩阵

**与普通模式的区别：**
- 普通模式：所有 HCA 同时测试，一个结果
- RbR 模式：每个 HCA 单独测试，N 个结果 + 汇总表

#### 基础用法

```bash
# 标准 RbR 测试
./ck-bench.sh --rbr -G -cx7

# 完整诊断（推荐）
./ck-bench.sh --rbr -G -cx7 --check-topology --output-csv

# 静默模式（仅显示汇总）
./ck-bench.sh --rbr -G -cx7 -q

# 使用自定义 hostfile
./ck-bench.sh --rbr -G -cx7 -f my_hosts.txt
```

#### 输出结构

```
results/rbr_<timestamp>_<host-range>/
├── summary.csv              # 汇总表（所有 HCA 性能对比）
├── topology.txt             # 拓扑映射（如果使用 --check-topology）
├── mlx5_gdr_1/
│   └── <timestamp>/
│       ├── latency.json
│       ├── latency.txt
│       ├── bandwidth.json
│       └── bandwidth.txt
├── mlx5_gdr_2/
│   └── <timestamp>/
│       └── ...
...
├── mlx5_gdr_8/
│   └── <timestamp>/
│       └── ...
```

#### 汇总表示例

**表格格式（默认）：**
```
==========================================
Rail-by-Rail Test Summary
==========================================
Rail           Latency(usec)    Bandwidth(MB/s)
------------------------------------------
mlx5_gdr_1            1.79            98446.8
mlx5_gdr_2            1.80            98347.0
mlx5_gdr_3            1.79            98402.1
mlx5_gdr_4            1.80            98383.7
mlx5_gdr_5            1.79            98421.3
mlx5_gdr_6            1.80            98389.5
mlx5_gdr_7            1.79            98456.2
mlx5_gdr_8            1.80            98394.1
==========================================
```

**CSV 格式（--output-csv）：**
```csv
Rail,Latency(usec),Bandwidth(MB/s)
mlx5_gdr_1,1.79,98402.1
mlx5_gdr_2,1.79,98383.7
mlx5_gdr_3,1.80,98421.3
mlx5_gdr_4,1.79,98389.5
mlx5_gdr_5,1.80,98456.2
mlx5_gdr_6,1.79,98394.1
mlx5_gdr_7,1.80,98421.7
mlx5_gdr_8,1.79,98402.9
```

#### 参数限制

| 组合 | 是否允许 | 原因 |
|------|----------|------|
| `--rbr` + `-z` | ❌ 否 | 压力测试需要所有 HCA 一起，RbR 用于快速诊断 |
| `--rbr` + `--check-topology` | ✅ 是 | 拓扑信息与测试结果一起保存 |
| `--rbr` + `--output-csv` | ✅ 是 | CSV 输出到标准输出并保存到文件 |
| `--rbr` + `-q` | ✅ 是 | 静默模式仅显示最终汇总 |
| `--rbr` + `--loop-test` | ✅ 是 | 每轮循环执行完整 RbR 测试 |

---

## 参数说明

### 基础参数

| 参数 | 简写 | 说明 | 默认值 |
|------|------|------|--------|
| `--hostfile` | `-f` | 节点列表文件 | `hostfile.txt` |
| `--hpcx_dir` | `-r` | HPC-X 安装路径 | 自动检测 |
| `--hca_list` | `-d` | HCA 列表（如 `mlx5_0:1,mlx5_1:1`） | - |
| `--auto-hca` | - | 自动检测 GPU-direct HCA | 关闭 |
| `--ppn` | `-p` | 每节点进程数 | 1 |

### 网络参数

| 参数 | 简写 | 说明 | 默认值 |
|------|------|------|--------|
| `--gpudirect` | `-G` | 启用 GPUDirect RDMA | 关闭 |
| `--connectx-7` | `-cx7` | ConnectX-7 优化（4 QPs）| 关闭 |
| `--traffic` | `-z` | 压力测试时长（分钟）| - |
| `--network-mode` | - | 网络模式：`ib` 或 `roce` | `ib` |

### 功能参数

| 参数 | 简写 | 说明 |
|------|------|------|
| `--rail-by-rail` | `--rbr` | 逐轨道测试模式 |
| `--check-topology` | - | 检查 GPU-NIC-Switch 拓扑 |
| `--check-health-only` | - | 仅健康检查（不运行基准测试）|
| `--Ca` | - | IB 拓扑查询使用的 HCA 设备（默认：mlx5_0）|
| `--output-csv` | - | CSV 格式输出（RbR 模式）|
| `--quiet` | `-q` | 静默模式（仅显示汇总）|
| `--loop-test` | - | 循环测试次数 |
| `--auto-numa` | - | 自动 NUMA 绑定 |
| `--view` | - | 查看历史结果（彩色矩阵）|

### 高级参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--auto-reboot` | 循环测试时自动重启节点 | 关闭 |
| `--reset-optics` | 重置光模块（不重启服务器）| 关闭 |
| `--auto-remove-bad-nodes` | 自动移除故障节点 | 关闭 |
| `--min-nodes` | 最少节点数要求 | 2 |
| `--reboot-interval` | 重启间隔（秒）| 1 |
| `--optics-interval` | 光模块重置间隔（秒）| 2 |

---

## 网络架构

### 典型部署架构

```
┌─────────────┐
│  Mac Client │  开发者笔记本
│  (可选)     │
└──────┬──────┘
       │ SSH
       ↓
┌─────────────────┐
│  CPU Server     │  管理节点/跳板机
│  10.20.4.4      │  - 脚本执行入口
│  /x/            │  - 结果收集存储
└────────┬────────┘  - SSH 跳转
         │ SSH
         ↓
┌────────────────────────────────┐
│  GPU Servers (10.20.2.x)       │  计算节点集群
│  ┌──────────┬──────────────┐   │
│  │ GPU 0-7  │  HCA 0-7     │   │  - HPC-X 安装路径
│  │          │  (mlx5_gdr_*)│   │  - 临时目录: /tmp/clusterkit/
│  └──────────┴──────────────┘   │
└────────────────────────────────┘
         │
         ↓ GPUDirect RDMA
   IB/RoCE Network (200G/400G)
         │
         ↓
┌────────────────────┐
│  Leaf Switches     │  网络交换机
│  - LLDP enabled    │  - LLDP 协议
│  - IB SM running   │  - IB Subnet Manager
└────────────────────┘
```

### 文件路径规范

| 位置 | 路径 | 用途 |
|------|------|------|
| **CPU Server** | `/x/ck-bench.sh` | 主脚本 |
| **CPU Server** | `/x/hostfile.txt` | 节点列表 |
| **CPU Server** | `/x/results/` | 测试结果存储 |
| **GPU Server** | `/opt/hpcx-*/` | HPC-X 安装目录 |

### GPU-HCA 自动映射

脚本解析 `nvidia-smi topo -m` 输出，识别 PIX (PCIe bridge) 连接：

```
典型映射关系：
GPU0 → mlx5_gdr_1
GPU1 → mlx5_gdr_2
GPU2 → mlx5_gdr_3
GPU3 → mlx5_gdr_4
GPU4 → mlx5_gdr_5
GPU5 → mlx5_gdr_6
GPU6 → mlx5_gdr_7
GPU7 → mlx5_gdr_8
```

**智能设备选择：**
- ✅ 优先选择 `mlx5_gdr_*` 设备
- ✅ 忽略 `mlx5_bond_*` 聚合设备
- ✅ 忽略管理网卡（如 mlx5_08）

---

## 输出说明

### 结果目录结构

```
results/
├── rbr_<timestamp>_<host-range>/    # Rail-by-Rail 测试结果
│   ├── summary.csv                  # 性能汇总（推荐导出）
│   ├── topology.txt                 # 拓扑信息
│   ├── mlx5_gdr_1/
│   │   └── <timestamp>/
│   │       ├── latency.json         # 延迟数据（JSON）
│   │       ├── latency.txt          # 延迟数据（文本）
│   │       ├── bandwidth.json       # 带宽数据（JSON）
│   │       └── bandwidth.txt        # 带宽数据（文本）
│   ├── mlx5_gdr_2/
│   │   └── ...
│   └── mlx5_gdr_8/
│       └── ...
├── topology_<timestamp>.txt         # 独立拓扑检查
└── <timestamp>/                     # 普通测试结果
    ├── latency.json
    ├── latency.txt
    ├── bandwidth.json
    └── bandwidth.txt
```

### 彩色输出说明

#### 10 色调色板（拓扑显示）

不同服务器使用不同颜色循环显示，便于区分：

```
服务器 1:  绿色
服务器 2:  黄色
服务器 3:  蓝色
服务器 4:  品红
服务器 5:  青色
服务器 6:  亮绿
服务器 7:  亮黄
服务器 8:  亮蓝
服务器 9:  亮品红
服务器 10: 亮青
服务器 11: 绿色（循环）
...
```

#### 5 级性能颜色（矩阵显示）

**带宽矩阵：**
- 🟢 绿色 (≥ 98%) - 优秀
- 🔵 青色 (≥ 96%) - 良好
- 🟡 黄色 (≥ 94%) - 可接受
- 🟣 品红 (≥ 92%) - 性能下降
- 🔴 红色 (< 92%) - 需要检查

**延迟矩阵：**
- 🟢 绿色 (≤ 2.0 μs) - 优秀
- 🔵 青色 (≤ 2.5 μs) - 良好
- 🟡 黄色 (≤ 3.0 μs) - 可接受
- 🟣 品红 (≤ 4.5 μs) - 延迟较高
- 🔴 红色 (> 4.5 μs) - 需要检查

### CSV 格式

使用 `--output-csv` 时：
- **输出到标准输出**（可重定向或管道）
- **同时保存到 summary.csv**（在结果目录中）

```bash
# 导出到自定义文件
./ck-bench.sh --rbr -G -cx7 --output-csv > my_results.csv

# 管道到 Python 分析
./ck-bench.sh --rbr -G -cx7 --output-csv | python3 analyze.py

# 导入到 Excel
./ck-bench.sh --rbr -G -cx7 --output-csv > results.csv
# 然后在 Excel 中打开 results.csv
```

---

## 故障排查

### 常见问题

#### 1. 脚本执行卡住不动

**现象：**
- 执行后长时间无输出
- 一直等待某个节点

**原因：**
- SSH 无法连接到远程节点
- 节点网络不通或挂起

**解决方案：**
```bash
# 已内置 10 秒超时保护，会自动跳过无响应节点

# 手动测试节点连接
ssh -o ConnectTimeout=10 10.0.10.1 "echo OK"

# 使用健康检查快速定位问题节点
./ck-bench.sh --check-health-only -f hostfile.txt
```

#### 2. GPU 列显示 N/A

**现象：**
- 拓扑输出中 GPU 列显示 N/A
- 某些 HCA 无法映射到 GPU

**原因：**
- `nvidia-smi` 命令失败或超时
- GPU 同时映射到多个 NIC（bond 设备覆盖）

**解决方案：**
```bash
# 手动检查 GPU 拓扑
ssh 10.0.10.1 "nvidia-smi topo -m"

# 查看 NIC Legend 部分
ssh 10.0.10.1 "nvidia-smi topo -m | grep -A 20 'NIC Legend'"

# 脚本已优化：自动优先选择 mlx5_gdr_* 设备
# 如果仍有问题，检查设备命名
ssh 10.0.10.1 "ibdev2netdev"
```

#### 3. RoCE 模式显示 NO_LLDP

**现象：**
- Switch 列显示 `NO_LLDP`
- Port 列显示 `N/A`

**原因：**
- lldpd 服务未运行
- 交换机未启用 LLDP
- LLDP 数据库为空

**解决方案：**
```bash
# 检查 lldpd 服务状态
ssh 10.0.20.1 "systemctl status lldpd"

# 启动 lldpd 服务
ssh 10.0.20.1 "sudo systemctl start lldpd"
ssh 10.0.20.1 "sudo systemctl enable lldpd"

# 检查 LLDP 邻居信息
ssh 10.0.20.1 "lldpcli show neighbors"

# 如果为空，等待几分钟后重试（LLDP 需要时间）
sleep 60
ssh 10.0.20.1 "lldpcli show neighbors"

# 交换机端启用 LLDP（参考交换机文档）
```

#### 4. IB 拓扑显示 ROUTE_ERR

**现象：**
- Switch 列显示 `ROUTE_ERR`
- Port 列显示 `ibtracert failed`

**原因：**
- Subnet Manager 未运行
- 使用了错误的 CA 设备查询 SM
- 多子网环境路由不通
- 网络分区或隔离

**解决方案：**
```bash
# 检查 SM 状态
ssh 10.0.20.1 "sminfo"

# 检查 ibstat
ssh 10.0.20.1 "ibstat"

# 多子网环境需指定正确的 CA 设备
./ck-bench.sh --check-topology --Ca mlx5_4

# 手动测试 ibtracert
ssh 10.0.20.1 "
  LID=\$(ibstat mlx5_0 | grep 'Base lid' | awk '{print \$3}')
  SM_LID=\$(sminfo --Ca mlx5_0 | grep 'sm lid' | grep -oP '\d+')
  ibtracert --Ca mlx5_0 \$LID \$SM_LID
"

# 检查网络连通性
ssh 10.0.20.1 "ibping -C mlx5_0 -L <remote_lid>"
```

#### 5. 网卡显示 Down 状态

**现象：**
- Port 列显示 `Down`
- Switch 列显示 `LINK_DOWN`

**原因：**
- 光模块故障
- 线缆问题
- 交换机端口禁用
- HCA 硬件问题

**解决方案：**
```bash
# 检查网卡状态
ssh 10.0.10.1 "ibdev2netdev"

# 检查物理链路
ssh 10.0.10.1 "mlxlink -d mlx5_gdr_8"

# 查看错误计数器
ssh 10.0.10.1 "mlxlink -d mlx5_gdr_8 -c"

# 重置光模块
ssh 10.0.10.1 "
  for port in \$(mst status | grep 'mt.*pciconf' | awk '{print \$1}'); do
    mlxlink -d \${port} --port_type PCIE --link_mode_force DISABLED
    sleep 2
    mlxlink -d \${port} --port_type PCIE --link_mode_force ENABLED
  done
"

# 检查交换机端口状态
```

#### 6. Rail-by-Rail 与压力测试冲突

**现象：**
- 使用 `--rbr` 和 `-z` 同时报错

**原因：**
- 参数互斥，不能同时使用

**解决方案：**
```bash
# ✗ 错误用法
./ck-bench.sh --rbr -G -cx7 -z 3

# ✓ 正确用法 1：RbR 快速诊断
./ck-bench.sh --rbr -G -cx7

# ✓ 正确用法 2：压力测试使用普通模式
./ck-bench.sh --auto-hca -G -cx7 -z 30
```

#### 7. 大规模集群测试慢

**现象：**
- 256 台服务器测试非常慢
- 拓扑收集耗时过长

**解决方案：**
```bash
# 脚本已优化，最大 32 并发
# 但仍需注意：

# 1. 分批测试
split -l 32 all_256_nodes.txt batch_
for batch in batch_*; do
    ./ck-bench.sh --rbr -G -cx7 -f $batch
    sleep 60  # 批次间等待
done

# 2. 仅拓扑检查（更快）
./ck-bench.sh --network-mode roce --check-topology -f all_256_nodes.txt

# 3. 健康检查（最快）
./ck-bench.sh --check-health-only -f all_256_nodes.txt
```

### 调试技巧

```bash
# 1. 启用详细输出
bash -x ./ck-bench.sh --auto-hca -G -cx7 2>&1 | tee debug.log

# 2. 检查远程节点日志
ssh 10.0.20.1 "dmesg | tail -100"

# 3. 检查 OFED 日志
ssh 10.0.20.1 "journalctl -u openibd -n 100 --no-pager"

# 4. 检查 HCA 状态
ssh 10.0.20.1 "
  echo '=== ibstat ==='
  ibstat
  echo '=== ibdev2netdev ==='
  ibdev2netdev
  echo '=== mlxlink ==='
  for dev in \$(ibdev2netdev | awk '{print \$1}' | sort -u); do
    echo \"--- \$dev ---\"
    mlxlink -d \$dev 2>&1 | head -20
  done
"

# 5. 测试 HPC-X 环境
ssh 10.0.20.1 "
  source /opt/hpcx-*/hpcx-init.sh
  hpcx_load
"
```

---

## 最佳实践

### 1. 日常运维

```bash
# 每日健康检查（快速）
./ck-bench.sh --check-health-only -f production.txt -q > daily_$(date +%Y%m%d).log

# 每周性能基线（完整）
./ck-bench.sh --rbr -G -cx7 --output-csv -f production.txt > weekly_$(date +%Y%m%d).csv

# 拓扑变更验证
./ck-bench.sh --network-mode roce --check-topology > topo_after_change.txt
diff topo_before_change.txt topo_after_change.txt
```

---

## 高级功能

### 循环测试

```bash
# 测试模式：5 轮循环（无重启）
./ck-bench.sh --auto-hca -G -cx7 -z 3 --loop-test 5

# 生产模式：5 轮循环 + 自动重启
./ck-bench.sh --auto-hca -G -cx7 -z 30 --loop 5 --auto-reboot

# 自定义重启间隔（2 秒）
./ck-bench.sh --auto-hca -G -cx7 -z 30 --loop 5 --auto-reboot --reboot-interval 2

# IPMI 电源循环（硬重启）
./ck-bench.sh --auto-hca -G -cx7 -z 30 --loop 5 --auto-reboot --reboot-method ipmi

# 光模块重置（不重启服务器）
./ck-bench.sh --auto-hca -G -cx7 -z 30 --loop-test 5 --reset-optics --optics-interval 1

# 自动移除故障节点
./ck-bench.sh --auto-hca -G -cx7 -z 30 --loop 10 --auto-reboot --auto-remove-bad-nodes --min-nodes 4
```

### NUMA 绑定

```bash
# 自动 NUMA 绑定（降低延迟）
./ck-bench.sh --rbr -G -cx7 --auto-numa

# 手动指定 NUMA 策略
./ck-bench.sh --auto-hca -G -cx7 --numa-policy node0
./ck-bench.sh --auto-hca -G -cx7 --numa-policy node1
./ck-bench.sh --auto-hca -G -cx7 --numa-policy none
```

### 历史结果查看

```bash
# 查看 RbR 会话所有轨道
./ck-bench.sh --view results/rbr_20251207_120000_GPU-1-GPU-9/

# 查看特定轨道
./ck-bench.sh --view results/rbr_20251207_120000_GPU-1-GPU-9/mlx5_0/

# 查看特定结果文件
./ck-bench.sh --view results/rbr_20251207_120000_GPU-1-GPU-9/mlx5_0/20251207_120015/bandwidth.txt
./ck-bench.sh --view results/20251207_120000/latency.txt
```

**功能：**
- 彩色矩阵显示（5 级颜色编码）
- 内存缓冲，无闪烁
- 支持任何历史结果目录

### 环境变量控制

```bash
# 强制指定运行模式
CK_FORCE_MODE=cpu_server ./ck-bench.sh --auto-hca -G -cx7

# 自定义 CPU 服务器
CK_CPU_SERVER=192.168.1.100 CK_CPU_USER=admin ./ck-bench.sh --auto-hca -G -cx7

# 自定义远程目录
CK_REMOTE_DIR=/opt/clusterkit ./ck-bench.sh --auto-hca -G -cx7

# 自定义 HPC-X 路径
HPCX_HOME=/opt/hpcx ./ck-bench.sh --auto-hca -G -cx7
```

---

## 拓扑分析工具

项目包含 `analyze_topology.py` 用于深度分析拓扑文件。

### 功能

- ✅ Rail 对齐检查（GPU ID → Leaf 交换机映射）
- ✅ 连接错误检测（NO_LID, ROUTE_ERR, WRONG_SWITCH）
- ✅ 端口一致性检查
- ✅ CSV 导出

### 使用方法

```bash
# 基础分析
python3 analyze_topology.py results/topology_latest.txt

# 导出 CSV
python3 analyze_topology.py results/topology_latest.txt --export rail_analysis.csv

# 分析 RbR 结果中的拓扑
python3 analyze_topology.py results/rbr_*/topology.txt --export analysis.csv
```

### Rail 对齐规则

| SU (Scale Unit) | GPU ID | 预期 Leaf 交换机 |
|-----------------|--------|------------------|
| **SU1** | GPU0-GPU7 | Leaf01-Leaf08 |
| **SU2** | GPU0-GPU7 | Leaf09-Leaf16 |
| **SU3** | GPU0-GPU7 | Leaf17-Leaf24 |

**计算公式：** `Expected Leaf = (SU - 1) × 8 + GPU_ID + 1`

---

## 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

---

### 代码规范

- 遵循现有的 Bash 脚本规范
- 为复杂逻辑添加注释
- 更新 README.md 文档（中英文）
- 确保 macOS 兼容性（避免 GNU 特定扩展）
- 使用 `shellcheck` 检查脚本

---

## 作者

**Vincent Wu**

- 邮箱: [Vincentwu@zhengytech.com](mailto:Vincentwu@zhengytech.com)
- 组织: Zhengy Technology

---

## 致谢

- 基于 NVIDIA HPC-X ClusterKit 构建
- 感谢 NVIDIA Networking 团队
- 感谢 HPC 和 GPU 计算社区的支持

---

## 相关资源

### 官方文档

- [NVIDIA HPC-X Documentation](https://docs.nvidia.com/networking/display/hpcxv221)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [InfiniBand Architecture](https://www.infinibandta.org/)
- [RDMA Core Documentation](https://github.com/linux-rdma/rdma-core)

### 相关工具

- [nvidia-smi](https://developer.nvidia.com/nvidia-system-management-interface) - GPU 管理和监控
- [lldpd](https://lldpd.github.io/) - LLDP 守护进程
- [opensm](https://www.openfabrics.org/) - InfiniBand Subnet Manager

### 推荐阅读

- [GPUDirect RDMA Best Practices](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html#best-practices)
- [InfiniBand Tuning Guide](https://community.mellanox.com/s/article/performance-tuning-for-mellanox-adapters)
- [NCCL Performance Tuning](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)

---

## 更新日志

### v2.0 (2025-12-16)

**新增功能：**
- ✨ RoCE 网络模式支持（LLDP 拓扑检测）
- ✨ 健康检查功能（--check-health-only）
- ✨ 并行拓扑收集（最大 32 并发）
- ✨ 10 秒超时保护机制
- ✨ 彩色输出（10 色调色板）
- ✨ 网卡 Down 状态显示
- ✨ 智能设备选择（优先 mlx5_gdr_*）

**优化改进：**
- ⚡ 大幅提升大规模集群测试速度
- ⚡ 修复 GPU 映射逻辑（两阶段处理）
- ⚡ 优化 SSH 连接管理
- ⚡ 改进错误处理和超时机制

**文档更新：**
- 📚 完整中文文档
- 📚 详细故障排查指南
- 📚 最佳实践和使用场景

### v1.0 (2024-12-01)

- 🎉 首次发布
- ✅ InfiniBand 模式支持
- ✅ Rail-by-Rail 测试
- ✅ 彩色矩阵显示
- ✅ 循环测试功能

---
