# BBR + CAKE 中转服务器一键调优 v3.3.1

面向 **Linux 中转 / 出口 / 共享节点** 的生产向网络调优脚本。  
**推荐环境：Debian / Ubuntu**（其它带 systemd + iproute2 的 Linux 一般也可用）。

---

## 一键安装（Debian / Ubuntu）

以 **root** 在服务器上执行（会测速，建议业务低峰）。  
**推荐先下载再执行**（比管道更稳，避免 stdin 被占、中途断流）：

```bash
wget -qO /tmp/Tuning https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning && sudo bash /tmp/Tuning
```

等价管道写法（一般也可用）：

```bash
wget -qO- https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning | sudo bash
```

> 极简系统若无 `wget`：先执行 `apt-get update && apt-get install -y wget`  
> 需要 root、内核 ≥ 5.4、支持 `sch_cake` / `ifb`。  
> 若卡在「阶段 1/5」：看是否出现 `[ERROR]`；可先 `sudo bash /tmp/Tuning selfcheck`。

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
| 多轮测速 | 默认 2 轮；样本够则提前结束 |
| 中位聚合 | `AGGREGATE_MODE=median` |
| 默认测速 | http + speedtest + iperf3；**bench 仅样本不足时 fallback** |
| 安全折扣 | 约 ×0.85（美/远路径更保守） |
| 速率校验 | 规范化 `100`/`100M`/`100Mbit`，非法限速不写入 tc |
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
bbr-cake-transit-exit selfcheck   # 环境自检（BBR/cake/依赖）
bbr-cake-transit-exit retune      # 立刻重测并应用
bbr-cake-transit-exit apply       # 用已有配置应用
bbr-cake-transit-exit cleanup     # 清运行时规则 + 临时文件（保留配置/服务）
bbr-cake-transit-exit uninstall   # 完全卸载
```

### 自动清理（测速结束后）

| 会清理 | 不清理 |
|--------|--------|
| `/tmp/bbr-cake.*` 会话目录 | 限速配置 `/etc/default/...` |
| 下载样本、bench 工作区 | systemd 服务与每日重测定时器 |
| 超过 2h 的残留临时目录 | sysctl、安装脚本 |
| 过大的 retune 日志（保留尾部） | 测速缓存、进度文件 |
| Ctrl+C 中断时尽量删临时文件 | 正在生效的 CAKE（cleanup 才拆） |

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
