# Wi-Fi 下 Tailscale 不通，切 5G 正常 — 排查指南

## 症状

连接 Wi-Fi 后 Tailscale 无法连接到远端出口节点，无法访问内网资源。切换到 5G 移动数据后立即恢复正常。

## 原因分析

Wi-Fi 不通但 5G 正常，说明问题在 **本地路由器/网络环境**，不是 Tailscale 本身。

### 最可能的原因

**1. 路由器 Symmetric NAT**

家用路由器如果使用 Symmetric NAT，每个目标地址分配不同的外部端口。Tailscale 的 STUN 探测会发现 `MappingVariesByDestIP: true`，P2P 打洞失败。如果 DERP 中继也被阻断，就完全不通。

**2. 路由器防火墙/安全功能**

很多路由器默认开启：
- **SPI 防火墙** — 把 Tailscale 的 STUN/UDP 包当攻击丢弃
- **DoS 防护** — 限制 UDP 连接数，Tailscale 探测被限流
- **UDP Flood Protection** — 直接丢弃大量 UDP 包

**3. DNS 被劫持**

路由器拦截或重定向 DNS 查询，导致 Tailscale 无法解析：
- `controlplane.tailscale.com`（控制面）
- DERP 服务器域名

## 排查步骤

### 第一步：检查 Tailscale 状态

```bash
tailscale status
tailscale netcheck
```

重点看：
- 是否 `offline` 或 `idle`
- `netcheck` 里 UDP 是否可用
- `MappingVariesByDestIP: true` = Symmetric NAT
- 用的哪个 DERP 节点，延迟多少

### 第二步：测试基本连通性

```bash
# 能否解析 Tailscale 控制面
nslookup controlplane.tailscale.com

# DERP 是否可达
curl -v https://your-derp-server:443

# 直接 ping 远端 Tailscale IP
ping 100.64.x.x
```

### 第三步：检查路由器设置

登录路由器管理界面，找以下设置并关闭测试：
- 防火墙 → DoS Protection / UDP Flood Protection
- 安全 → SPI Firewall
- NAT → ALG (Application Layer Gateway)
- DNS → DNS Rebind Protection / DNS Proxy

## 临时解决方案

```bash
# 强制走 DERP relay（如果 DERP 本身可达）
tailscale ping <远端节点的tailscale-ip>

# 手动改 DNS 绕过路由器 DNS 劫持
# macOS:
networksetup -setdnsservers Wi-Fi 1.1.1.1 8.8.8.8

# Linux:
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

## 永久修复

取决于路由器型号：
- 关闭不必要的安全功能（DoS/SPI/UDP flood）
- 确保 NAT 不是 Symmetric 类型
- 不要劫持 DNS

如果路由器太旧无法调整，考虑换一个支持 Full Cone NAT 的路由器。

## 教训总结

1. **换网络后 Tailscale 断连，优先查本地路由器** — 不是 Tailscale 的问题
2. **5G 正常说明远端没问题** — 问题 100% 在本地网络出口
3. **家用路由器的"安全"功能经常是 VPN 的敌人** — SPI、DoS 防护、UDP 限制都会干扰
