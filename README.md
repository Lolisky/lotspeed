### lotspeed ml-tcp

<div align=center>
    <img src="https://github.com/Lolisky/lotspeed/raw/ml-tcp/logo.png" width="400" height="400" />
</div>

# <span style="color: red;">⚠️ AI瞎改的，不知道用了会不会爆炸 ⚠️</span>

> **v5.7 - Kernel 6.8+ 兼容版本**
>
> 本仓库基于 [uk0/lotspeed](https://github.com/uk0/lotspeed) 的 ml-tcp 分支，修复了 Linux 内核 6.8+ 的编译兼容性问题。

### branch explanation

* `ml-tcp`: lotspeed ml-tcp 基于学习历史记录的模式进行加速，并且洲际场景抖动不会降速避让。
* **支持内核版本**:
  - **4.9 - 6.7**: 使用 4 参数拥塞控制 API
  - **6.8+**: 使用 2 参数拥塞控制 API（自动检测）

---

## 快速安装

### 一键自动安装（推荐）

```bash
curl -fsSL https://raw.githubusercontent.com/Lolisky/lotspeed/ml-tcp/install.sh | sudo bash
#   或
wget -qO- https://raw.githubusercontent.com/Lolisky/lotspeed/ml-tcp/install.sh | sudo bash
```

安装完成后，使用 `lotspeed status` 查看状态。

---

## 手动编译安装

### 1. 克隆仓库并编译

```bash
# 克隆本仓库
git clone https://github.com/Lolisky/lotspeed.git
cd lotspeed

# 编译模块
make
```

### 2. 加载模块

```bash
# 加载内核模块
sudo insmod lotspeed.ko

# 设置为当前拥塞控制算法
sudo sysctl -w net.ipv4.tcp_congestion_control=lotspeed
sudo sysctl -w net.ipv4.tcp_no_metrics_save=1

# 查看是否生效
sysctl net.ipv4.tcp_congestion_control

# 查看加载日志
dmesg | tail -20
```

---

## 管理命令

安装后会创建 `/usr/local/bin/lotspeed` 管理脚本：

```bash
# 查看状态
lotspeed status

# 启动/停止/重启
lotspeed start
lotspeed stop
lotspeed restart

# 应用预设配置
lotspeed preset balanced     # 推荐
lotspeed preset conservative
lotspeed preset aggressive

# 设置参数
lotspeed set lotserver_rate 256000000
lotspeed set lotserver_min_cwnd 16

# 实时查看日志
lotspeed monitor

# 完全卸载
lotspeed uninstall
```

### 状态示例

```bash
$ lotspeed status
╔════════════════════════════════════════════════════════════════════╗
║                   LotSpeed v5.7 Status (ML-TCP)                    ║
╟────────────────────────────────────────────────────────────────────╢
║ Module Status                                               Loaded ║
║ Reference Count                                                  1 ║
║ Active Connections                                              05 ║
║ Active Algorithm                                          lotspeed ║
╟────────────────────────────────────────────────────────────────────╢
║                         Current Parameters                         ║
╟────────────────────────────────────────────────────────────────────╢
║ Global Rate Limit                          125.00 MB/s (1.00 Gbps) ║
║ Min CWND                                                16 packets ║
║ Max CWND                                             15000 packets ║
║ Fairness (Beta)                                                60% ║
║ Turbo Mode                                                Disabled ║
║ Safe Mode                                                  Enabled ║
║ FAST Alpha                                              20 packets ║
║ FAST Gamma                                                     50% ║
║ SS Exit Threshold                                              25% ║
║ High-Delay Mode                                            Enabled ║
║ HD Threshold                                              180000us ║
║ HD Reference RTT                                           80000us ║
║ HD Gamma Boost                                                 20% ║
║ HD Alpha Boost                                          10 packets ║
║ Brave Mode                                                 Enabled ║
║ Brave RTT Tolerance                                            25% ║
║ Brave Hold Time                                              400ms ║
║ Brave Floor                                                    85% ║
║ Brave Push                                                      8% ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## v5.7 更新内容

### 修复的问题

**Linux 内核 6.8+ 编译错误修复：**

1. **cong_control 函数签名适配**
   - 内核 6.8+ 使用 2 参数 API: `(struct sock *, const struct rate_sample *)`
   - 旧内核使用 4 参数 API: `(struct sock *, u32, int, const struct rate_sample *)`
   - 代码内置内核版本检测，通过 `LINUX_VERSION_CODE` 宏自动适配

2. **time_before 类型警告修复**
   - 添加显式类型转换修复第 389 行的编译警告

### 技术细节

```c
// 内核版本检测 (lotspeed.c 第 13-16 行)
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 8, 0)
#define LOTSPEED_USE_2PARAM_CONG_CONTROL 1
#endif

// 函数签名自动适配 (lotspeed.c 第 499-511 行)
#ifdef LOTSPEED_USE_2PARAM_CONG_CONTROL
static void lotspeed_cong_control(struct sock *sk, const struct rate_sample *rs)
#else
static void lotspeed_cong_control(struct sock *sk, u32, int, const struct rate_sample *rs)
#endif
```

---

## 测试与验证

### YouTube 测试

<div align=center>
    <img src="https://github.com/Lolisky/lotspeed/raw/ml-tcp/zeta-tcp.png" width="1024" height="768" />
</div>

### iperf3 丢包测试

```bash
# 禁用 LRO
ethtool -K eth0 lro off

# 模拟 16% 丢包
sudo tc qdisc add dev eth0 root netem loss 16%

# 测试命令（服务端）
iperf3 -4 -s -p 35201

# 测试命令（客户端）
iperf3 -c <server_ip> -p 35201 -R -t 30

# 取消丢包
sudo tc qdisc del dev eth0 root netem
```

---

## 速度测试对比

### 使用前

![before](https://github.com/Lolisky/lotspeed/raw/ml-tcp/img/b058ec2ebdb2a095d396cea05dccf499.png)

### 使用后

![after](https://github.com/Lolisky/lotspeed/raw/ml-tcp/img/f7525becdae16659ddfd54d99efe0f66.png)

---

## 技术特性

✅ 基于"时延+丢包"混合驱动的拥塞控制
✅ 学习型状态机
✅ 洲际场景适配
✅ **内核 6.8+ 完全兼容**

---

## 相关项目

PAC (Proactive ACK Control) for TCP Incast Congestion:
* https://github.com/Lolisky/lotspeed

---

## 致谢

- 原作者: [uk0](https://github.com/uk0)
- 原项目: https://github.com/uk0/lotspeed
- 本分支修复: Linux 内核 6.8+ 兼容性

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=Lolisky/lotspeed&type=timeline&logscale&legend=top-left)](https://www.star-history.com/#Lolisky/lotspeed&type=timeline&logscale&legend=top-left)
