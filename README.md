# BBR + CAKE 中转服务器一键调优 v3.1

面向 **Linux 中转 / 出口 / 共享节点** 的生产向网络调优脚本。  
**推荐环境：Debian / Ubuntu**（其它带 systemd + iproute2 的 Linux 一般也可用）。

---

## 一键安装（Debian / Ubuntu）

以 **root** 在服务器上执行（会测速，建议业务低峰）：

```bash
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo bash
```

> 极简系统若无 `wget`：先执行 `apt-get update && apt-get install -y wget`  
> 需要 root、内核 ≥ 5.4、支持 `sch_cake` / `ifb`。

### 自动完成

1. 环境检查 + 地区识别（`asia` / `eu` / `us` / `far`）  
2. 多轮测速 → 中位聚合 → ×安全系数 → 上限保护  
3. CPU / RTT 策略  
4. 应用 BBR + CAKE  
5. systemd 开机恢复 + **北京时间每日 03:00 重测**

---

## 特性

| 能力 | 说明 |
|------|------|
| BBR + CAKE | 拥塞控制 + AQM/整形 |
| 多轮测速 | 默认 3 轮；bench 仅第 1 轮 |
| 中位聚合 | `AGGREGATE_MODE=median` |
| 安全折扣 | 约 ×0.85（美/远路径更保守） |
| 地区差异化 | `asia` / `eu` / `us` / `far` |
| 位置识别 | `REGION=auto` 地理 API，失败则 ping RTT |
| 上限 / CPU | `MAX_RATE`、核数软上限、高负载关 ingress |
| 可视化 | 阶段横幅 + 实时输出 + 进度文件 |
| 每日重测 | 北京时间 03:00；失败保留旧限速 |

---

## 地区档位

| REGION | 典型场景 | CAKE_RTT | 安全系数 |
|--------|----------|----------|----------|
| `asia` | 日/港/新/韩/国内等 | 65ms | 0.85 |
| `eu` | 欧洲 | 100ms | 0.85 |
| `us` | 北美 / 美亚中转 | 150ms | 0.82 |
| `far` | 澳新/南美/非洲/中东等 | 200ms | 0.80 |
| `auto` | 地理识别 → RTT 分档（默认） | 见上 | 见上 |

```bash
# 手动指定地区（可选，装在一键命令前导出也可）
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo REGION=asia bash
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo REGION=us bash
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo GEO_DETECT=0 bash
```

> 识别的是 **服务器出口 IP 位置**，不是用户位置。美机服务亚洲用户请用 `REGION=us` 或手写 `CAKE_RTT`。

---

## 可选参数

```bash
# 硬上限
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo MAX_RATE=2000Mbit bash

# 已知带宽，跳过测速
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo UP_RATE=90Mbit DOWN_RATE=900Mbit SKIP_DETECT=1 bash

# 关闭每日重测
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo DAILY_RETEST=0 bash
```

| 变量 | 默认 | 含义 |
|------|------|------|
| `BANDWIDTH_SAFETY_FACTOR` | 按地区 | 测速折扣 |
| `AGGREGATE_MODE` | `median` | median / max / min |
| `DETECT_ROUNDS` | `3` | 快方法轮数 |
| `MAX_RATE` | 空 | 硬上限 |
| `DAILY_RETEST` | `1` | 北京时间每日重测 |
| `RETEST_HOUR` | `3` | 重测小时 |
| `REGION` | `auto` | asia / eu / us / far / auto |
| `GEO_DETECT` | `1` | 是否查询地理位置 |
| `LIVE_OUTPUT` | `1` | 实时打印过程 |

---

## 安装后命令

```bash
bbr-cake-transit-exit status      # 状态 / 进度 / 定时器 / 地区
bbr-cake-transit-exit retune      # 立刻重测并应用
bbr-cake-transit-exit apply       # 用已有配置应用
bbr-cake-transit-exit uninstall   # 卸载
```

进度与日志：

```bash
cat /var/cache/bbr-cake-transit-exit/progress.txt
tail -f /var/log/bbr-cake-transit-exit/retune.log
```

---

## 仓库文件

| 文件 | 说明 |
|------|------|
| `Tuning` | 主脚本（一键安装拉取此文件） |
| `README.md` | 本说明 |
