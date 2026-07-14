# BBR + CAKE 中转服务器一键调优

面向 **Linux 中转 / 出口 / 共享节点** 的网络调优脚本：

- 开启 **BBR** + **CAKE** 队列
- **多方法自动测速**（http / speedtest / iperf3 / bench）
- 按峰值 × 0.92 **自动填写**上下行限速
- 一键应用并 **systemd 持久化**

> 需要 root，内核 ≥ 5.4，支持 `sch_cake` / `ifb`。  
> 测速会短时打满带宽，建议业务低峰执行。

---

## 一键安装（推荐）

在 **Linux 服务器** 上执行：

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning)
```

若系统没有 process substitution，可用：

```bash
curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning -o /tmp/Tuning && chmod +x /tmp/Tuning && sudo bash /tmp/Tuning
```

或：

```bash
wget -qO /tmp/Tuning https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning && chmod +x /tmp/Tuning && sudo bash /tmp/Tuning
```

### 自动完成

1. 多方法测速  
2. 汇总峰值并自动写入 `UP_RATE` / `DOWN_RATE`  
3. 应用 BBR + CAKE  
4. 安装到 `/usr/local/sbin` 并启用 systemd 开机生效  

**无需再分步执行 detect / apply / install。**

---

## 可选参数

```bash
# 指定 iperf3 对端（更贴近真实中转路径）
IPERF3_SERVER=1.2.3.4 bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning)

# 已知带宽，跳过测速
UP_RATE=90Mbit DOWN_RATE=900Mbit SKIP_DETECT=1 bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning)

# 只当次生效，不写 systemd
AUTO_INSTALL=0 bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning)

# 禁止远程 bench 脚本
ALLOW_REMOTE_BENCH=0 bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning)
```

---

## 常用命令（安装后）

```bash
bbr-cake-transit-exit status      # 查看状态
bbr-cake-transit-exit uninstall  # 卸载
bbr-cake-transit-exit auto       # 重新一键测速+调优
bbr-cake-transit-exit detect     # 仅测速
bbr-cake-transit-exit apply      # 仅应用
```

本地文件方式：

```bash
sudo bash Tuning          # 默认 = auto 一键
sudo bash Tuning status
sudo bash Tuning uninstall
```

---

## 测速说明

| 方法 | 说明 |
|------|------|
| http | 多 URL 下载估下行 |
| speedtest | 已安装 `speedtest` / `speedtest-cli` 时使用 |
| iperf3 | 需设置 `IPERF3_SERVER` |
| bench | 一键模式默认允许（第三方脚本，有供应链风险） |

- 结果为 **当前路径可用峰值估计**，不是运营商合同带宽  
- 限速默认 = 峰值 × **0.92**（`BANDWIDTH_SAFETY_FACTOR`）  
- 只测到单向时会镜像到另一向，便于 CAKE 双向整形  

---

## 卸载

```bash
sudo bbr-cake-transit-exit uninstall
```

或：

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/JackHONGhy/Tuning/main/Tuning) uninstall
```

---

## 环境要求

- Linux x86_64 / aarch64 等常见架构  
- root  
- 内核 ≥ 5.4，BBR 可用  
- `ip`、`tc`；建议有 `curl` 或 `wget`  
- 可选：`speedtest-cli`、`iperf3`、`python3`  

---

## 免责声明

本脚本会修改 sysctl、tc qdisc、iptables MSS 与 systemd 单元。请在自有服务器上使用，生产环境建议先在测试机验证。远程 `bench` 会执行第三方代码，请自行评估风险。

仓库：https://github.com/JackHONGhy/Tuning  
