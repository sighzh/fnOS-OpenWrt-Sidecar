# fnOS-OpenWrt-Sidecar

在 fnOS 上通过 Docker 运行 OpenWrt 旁路由，实现透明代理翻墙的个人网络方案。

---

## 网络架构

```
┌─────────────────────────────────────────────────────────────┐
│                  家庭局域网 192.168.1.0/24                    │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              本机 192.168.1.6                         │  │
│  │                                                       │  │
│  │  所有出站流量                                          │  │
│  │  DNS  → 192.168.1.2                                 │  │
│  │  网关 → macvlan0（192.168.1.3）→ 旁路由              │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  macvlan0@enp2s0  IP: 192.168.1.3             │  │  │
│  │  │  （虚拟网卡，用于绕过 macvlan 二层隔离限制）       │  │  │
│  │  └──────────────────────┬──────────────────────────┘  │  │
│  └─────────────────────────┼───────────────────────────┘  │
│                            │                              │
│  ┌─────────────────────────▼───────────────────────────┐  │
│  │        fnOS + Docker                                │  │
│  │        旁路由 OpenWrt  192.168.1.2                │  │
│  │        Passwall + sing-box 透明代理                  │  │
│  │        TProxy（fwmark 0x1 → 路由表100）              │  │
│  │        DNS 防污染 ✅                                 │  │
│  └──────────────────────────┬───────────────────────────┘  │
│                             │                              │
└─────────────────────────────┼──────────────────────────────┘
                              ▼
                   ┌──────────────────────┐
                   │    上级路由 / ISP     │
                   └──────────┬───────────┘
                              ▼
                   ┌──────────────────────┐
                   │       互联网          │
                   └──────────────────────┘
```

### 关键 IP 说明

| 角色 | IP | 说明 |
|------|----|------|
| 本机 | 192.168.1.6 | fnOS 宿主机，物理网卡 enp2s0 |
| macvlan 虚拟网卡 | 192.168.1.3 | 本机出口，绑定在 enp2s0 上 |
| 旁路由 OpenWrt | 192.168.1.2 | Docker 容器，透明代理出口 |
| 主路由 | 192.168.1.1 | 家用路由器，不做代理 |

---

## 为什么需要 macvlan0（3）？

macvlan 有一个特性：**同一张物理网卡上，macvlan 虚拟接口和物理接口之间无法直接二层通信**。

如果本机直接用 `enp2s0`（192.168.1.6）作为出口，流量发往 192.168.1.2 时在二层就被隔离，无法到达。

解决方案：创建 `macvlan0` 虚拟网卡（192.168.1.3），让本机流量从这张虚拟网卡出去，经过交换机绕一圈再到 192.168.1.2，正常通信。

```
enp2s0（192.168.1.6）  →  ❌ 无法直接访问 macvlan 设备
macvlan0（192.168.1.3）→  ✅ 可以访问 192.168.1.2
```

---

## 部署步骤

### 1. 在 fnOS 上启动 OpenWrt 容器

```bash
git clone https://github.com/sighzh/fnOS-OpenWrt-Sidecar.git
cd fnOS-OpenWrt-Sidecar
docker compose up -d
```

> 启动后访问 OpenWrt Web UI：`http://192.168.1.2`，安装并配置 Passwall 插件。

### 2. 本机配置 macvlan 网卡

```bash
# 创建 macvlan 虚拟网卡
sudo ip link add macvlan0 link enp2s0 type macvlan mode bridge
sudo ip addr add 192.168.1.3/24 dev macvlan0
sudo ip link set macvlan0 up

# 设置默认网关走 macvlan0
sudo ip route add default via 192.168.1.2 dev macvlan0

# 验证路由
ip route get 8.8.8.8
# 预期：8.8.8.8 via 192.168.1.2 dev macvlan0
```

### 3. 配置 DNS（关键，防止 DNS 污染）

```bash
# 临时修改
echo "nameserver 192.168.1.2" | sudo tee /etc/resolv.conf

# 永久生效（NetworkManager）
sudo nmcli connection modify "有线连接 1" ipv4.dns "192.168.1.2 8.8.8.8"
sudo nmcli connection modify "有线连接 1" ipv4.ignore-auto-dns yes
sudo nmcli connection up "有线连接 1"
```

### 4. 持久化 macvlan 配置（防重启失效）

创建 systemd 服务：

```bash
sudo tee /etc/systemd/system/macvlan-setup.service << 'EOF'
[Unit]
Description=Setup macvlan0 for OpenWrt sidecar
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c '\
  ip link add macvlan0 link enp2s0 type macvlan mode bridge && \
  ip addr add 192.168.1.3/24 dev macvlan0 && \
  ip link set macvlan0 up && \
  ip route add default via 192.168.1.2 dev macvlan0 metric 50'
ExecStop=/bin/bash -c '\
  ip route del default via 192.168.1.2 dev macvlan0 && \
  ip link del macvlan0'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable macvlan-setup
sudo systemctl start macvlan-setup
```

### 5. 验证

```bash
# 确认流量走旁路由
traceroute 8.8.8.8
# 第一跳应显示 192.168.1.2

# 确认 HTTPS 正常
curl -I https://www.google.com
# 预期：HTTP/2 200

# 确认 DNS 无污染
dig www.google.com
# 应返回真实 Google IP（142.x.x.x），而非 Facebook IP
```

---

## 常见问题

### DNS 被污染（Google 解析到 Facebook IP）

本机 DNS 指向主路由（192.168.1.1）时，主路由无防污染能力，会把 Google 等域名解析到错误 IP。

**修复：** 将 DNS 改为旁路由 192.168.1.2（Passwall 提供防污染 DNS）。

### SSL_ERROR_SYSCALL

通常是流量没有走旁路由（DNS 污染或路由规则不对）。先确认 `ip route get 8.8.8.8` 返回的出口是 `macvlan0`，再确认 DNS 服务器是 192.168.1.2。

### 重启后 macvlan0 消失

未持久化配置，按上方"持久化 macvlan 配置"步骤创建 systemd 服务。

---

## 参考

- [OpenWrt Docker 镜像](https://github.com/sulinggg/openwrt)
- Passwall 插件：OpenWrt → System → Software 安装
