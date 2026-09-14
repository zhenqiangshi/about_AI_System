

## 一、问题背景

| 项目 | 内容 |
|---|---|
| 现象 | `nvidia-smi` 报 `Failed to initialize NVML: Driver/library version mismatch` |
| 根因 | Ubuntu `unattended-upgrades` 自动升级 NVIDIA 驱动，用户态库更新后未重启，内核模块仍是旧版 |
| 触发时间 | 2026-09-11 06:07（`apt-daily-upgrade.timer` 触发） |
| 版本跨度 | 580.173.02 → 580.178.04 |
| 恢复方式 | 重启或重新加载内核模块，使版本对齐 |

### 判断自动 vs 手动升级

```bash
grep "Commandline:" /var/log/apt/history.log | tail -20
```

| `Commandline:` 内容 | 结论 |
|---|---|
| `/usr/bin/unattended-upgrade` | 自动升级 |
| `apt` / `apt-get` | 手动升级 |

---

## 二、脚本清单

建议统一放在 `/root/ops/` 下：

```
/root/ops/
├── disable-unattended-upgrades.sh    # 禁用自动升级
├── enable-unattended-upgrades.sh     # 恢复自动升级（回滚用）
└── check-gpu-stack.sh                # 检查驱动 + CUDA 栈
```

---

## 三、脚本一：`disable-unattended-upgrades.sh`

### 用途
禁用 Ubuntu 无人值守自动升级，防止 NVIDIA 驱动被意外替换。

### 脚本内容

```bash
#!/bin/bash
# disable-unattended-upgrades.sh
# 禁用 Ubuntu 无人值守自动升级，避免 NVIDIA 驱动被自动替换
# 适用: Ubuntu 22.04 / 24.04
# 恢复: 运行 enable-unattended-upgrades.sh

set -e

echo "===== 1. 禁用 systemd 服务与定时器 ====="
systemctl disable --now unattended-upgrades
systemctl disable --now apt-daily.timer
systemctl disable --now apt-daily-upgrade.timer

echo ""
echo "===== 2. 关闭 APT 周期任务开关 ====="
sed -i 's/"1"/"0"/g' /etc/apt/apt.conf.d/20auto-upgrades

echo ""
echo "===== 3. 验证 ====="
echo "--- is-enabled ---"
systemctl is-enabled unattended-upgrades apt-daily.timer apt-daily-upgrade.timer 2>&1

echo "--- is-active ---"
systemctl is-active unattended-upgrades apt-daily.timer apt-daily-upgrade.timer 2>&1

echo "--- timers ---"
systemctl list-timers --all | grep -E 'apt-daily|unattended' || echo "(无定时器，已关闭)"

echo "--- 20auto-upgrades ---"
cat /etc/apt/apt.conf.d/20auto-upgrades

echo "--- cron ---"
ls /etc/cron.daily/ /etc/cron.d/ 2>/dev/null | grep -iE 'apt|unattended' || echo "(cron 无相关任务)"

echo ""
echo "✅ 自动升级已禁用。"
echo "   日后所有升级需手动执行: sudo apt update && sudo apt upgrade"
```

### 验证标准

| 检查项 | 通过标准 |
|---|---|
| `is-enabled` | 三个都 `disabled` |
| `is-active` | timer 都 `inactive` |
| `list-timers` | 无输出 |
| `20auto-upgrades` | 两个都是 `"0"` |
| `cron` | 仅剩 `apt-compat`（无影响） |

### 实测输出样例

```
disabled
disabled
disabled
=== 2. is-active ===
inactive
inactive
inactive
=== 3. timers ===
(无定时器，已关闭)
=== 4. 20auto-upgrades ===
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
=== 5. cron ===
apt-compat
aptitude
```

### 时间实证（次日验证）

```bash
grep -E "Start-Date|unattended-upgrade" /var/log/apt/history.log | tail -6
date
```

- 最后一条 `Start-Date` 仍停在禁用当天 → ✅ 生效
- 出现禁用之后的新记录 → ❌ 未生效，需排查

---

## 四、脚本二：`enable-unattended-upgrades.sh`

### 用途
恢复自动升级（回滚用）。

### 脚本内容

```bash
#!/bin/bash
# enable-unattended-upgrades.sh
# 恢复 Ubuntu 无人值守自动升级

set -e

echo "===== 恢复 systemd 服务与定时器 ====="
systemctl enable --now unattended-upgrades
systemctl enable --now apt-daily.timer
systemctl enable --now apt-daily-upgrade.timer

echo ""
echo "===== 恢复 APT 周期任务开关 ====="
sed -i 's/"0"/"1"/g' /etc/apt/apt.conf.d/20auto-upgrades

echo ""
echo "===== 验证 ====="
systemctl is-enabled unattended-upgrades apt-daily.timer apt-daily-upgrade.timer
cat /etc/apt/apt.conf.d/20auto-upgrades

echo ""
echo "✅ 自动升级已恢复。"
```

### 验证标准

| 检查项 | 通过标准 |
|---|---|
| `is-enabled` | 三个都 `enabled` |
| `list-timers` | 能看到 `apt-daily*` |
| `20auto-upgrades` | 两个都是 `"1"` |

---

## 五、脚本三：`check-gpu-stack.sh`

### 用途
一键检查 NVIDIA 驱动、CUDA 栈、环境变量是否正常。

### 脚本内容

```bash
#!/bin/bash
# check-gpu-stack.sh
# 检查 NVIDIA 驱动 + CUDA 栈完整性

echo "===== 1. NVIDIA 驱动 (nvidia-smi) ====="
nvidia-smi

echo ""
echo "===== 2. 内核模块版本 ====="
cat /proc/driver/nvidia/version

echo ""
echo "===== 3. 用户态驱动包版本 ====="
dpkg -l | grep -E 'nvidia-driver|nvidia-utils|libnvidia-compute' | awk '{print $2, $3}'

echo ""
echo "===== 4. CUDA 运行时 (nvcc) ====="
nvcc --version 2>/dev/null || echo "nvcc 未安装 (仅驱动不需要)"

echo ""
echo "===== 5. CUDA 库版本 ====="
ls -l /usr/local/ | grep cuda

echo ""
echo "===== 6. 环境变量 ====="
echo "CUDA_HOME=$CUDA_HOME"
echo "LD_LIBRARY_PATH=$LD_LIBRARY_PATH"

echo ""
echo "===== 7. cuDNN (如有) ====="
ls /usr/local/cuda/lib64/libcudnn* 2>/dev/null | head || echo "cuDNN 未在默认路径找到"
```

### 关键检查项

| 检查项 | 正常值 |
|---|---|
| `nvidia-smi` | 不报错，列出 GPU |
| `/proc/driver/nvidia/version` | 与用户态库同版本 |
| `dpkg -l` nvidia 包 | 与内核模块同版本 |
| `nvcc --version` | 有则显示版本，无则说明只装驱动 |
| `CUDA_HOME` | `/usr/local/cuda` |
| `LD_LIBRARY_PATH` | 包含 `/usr/local/cuda-13.0/lib64` |

### 版本一致性判断

```bash
cat /proc/driver/nvidia/version    # 内核模块
nvidia-smi                          # 用户态库
```

两者版本号必须一致（如都为 `580.178.04`）。

---

## 六、CUDA 环境变量补齐

### 背景
`CUDA_HOME` 未设置，可能影响 CMake 等工具定位 CUDA。

### 补齐命令

```bash
sudo tee /etc/profile.d/cuda.sh > /dev/null <<'EOF'
# CUDA Toolkit (managed by alternatives, currently 13.0)
export CUDA_HOME=/usr/local/cuda
export PATH=$CUDA_HOME/bin:$PATH
EOF
```

### 说明

| 设计点 | 原因 |
|---|---|
| 用 `/usr/local/cuda` 而非 `cuda-13.0` | 软链接版本无关，将来切版本不用改 |
| 不加 `LD_LIBRARY_PATH` | 已有正确值，避免重复 |
| 放 `/etc/profile.d/` | 全局、持久、所有登录用户生效 |

### 验证

```bash
source /etc/profile.d/cuda.sh
echo "CUDA_HOME=$CUDA_HOME"
which nvcc
readlink -f $(which nvcc)
nvcc --version
```

期望输出：

```
CUDA_HOME=/usr/local/cuda
/usr/local/cuda/bin/nvcc
/usr/local/cuda-13.0/bin/nvcc
Cuda compilation tools, release 13.0, V13.0.48
```

> **注意**：环境变量是进程级，已在运行的 vLLM 任务不受影响，无需重启。

---

## 七、日常运维规范

### 手动升级 NVIDIA 驱动的标准流程

```bash
# 1. 检查有哪些更新
sudo apt update
apt list --upgradable | grep -i nvidia

# 2. 若有 NVIDIA 或内核更新，安排维护窗口
#    先停止业务（vLLM、推理服务等）

# 3. 执行升级
sudo apt upgrade

# 4. 重启
sudo reboot

# 5. 重启后验证
./check-gpu-stack.sh
nvidia-smi
```

### 禁止操作

| 操作 | 风险 |
|---|---|
| 任务运行中直接 `apt upgrade` NVIDIA 包 | 驱动版本错乱，业务中断 |
| 升级后不重启继续用 | `Driver/library version mismatch` |
| 直接 `rmmod nvidia` 卸载模块 | 大量 GPU 进程占用时会失败 |

---

## 八、故障速查表

| 现象 | 排查命令 | 处理 |
|---|---|---|
| `Driver/library version mismatch` | `cat /proc/driver/nvidia/version` + `nvidia-smi` | 重启 |
| 驱动升级后不生效 | `dkms status` | 确认 DKMS 编译成功 |
| `nvidia-smi` 找不到库 | `ldconfig -p \| grep nvidia-ml` | 检查 `LD_LIBRARY_PATH` |
| 自动升级又发生 | `grep "Commandline:" /var/log/apt/history.log` | 重跑禁用脚本 |
| CUDA 路径混乱 | `readlink -f /usr/local/cuda` | `update-alternatives --config cuda` |

---

## 九、本次处置记录（2026-09-11）

| 阶段 | 操作 | 结果 |
|---|---|---|
| 发现 | `nvidia-smi` 报版本不匹配 | 确认库 580.178，模块 580.173.02 |
| 定位 | 查 `dpkg.log` + `history.log` | 确认 `/usr/bin/unattended-upgrade` 自动升级 |
| 恢复 | 内核模块加载 580.178.04 | 版本对齐，`nvidia-smi` 正常 |
| 加固 | 禁用自动升级 | 三个 unit disabled，配置为 "0" |
| 补齐 | 设置 `CUDA_HOME` | `/usr/local/cuda` → `cuda-13.0` |
| 验证 | `check-gpu-stack.sh` | 全绿，8 卡 A100 在线，业务正常 |

---

**文档版本**：v1.0
**最后更新**：2026-09-11