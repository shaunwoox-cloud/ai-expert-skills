# MikroTik RB5009 阻断 Tailscale 直连 — 完整排查

## 症状

所有家庭内网设备的 Tailscale 连接都无法建立 P2P 直连，全部走 DERP 中继，延迟高、带宽受限。

`tailscale status` 显示所有节点为 `relay` 而非 `direct`。

## 根因分析

RB5009 防火墙配置存在 **四个叠加问题**，任何一个都会影响 Tailscale，组合起来就是完全不可用。

### 问题 1：dstnat 规则冲突

两条 dstnat 规则把 UDP 41641 分别转发到两个不同的内网设备：

```routeros
# 规则 A：转发到设备 A（缺少 action 和 in-interface）
add chain=dstnat dst-port=41641 protocol=udp to-addresses=192.168.50.20 to-ports=41641

# 规则 B：转发到设备 B
add action=dst-nat chain=dstnat dst-port=41641 in-interface=pppoe-out1 protocol=udp to-addresses=192.168.50.7 to-ports=41641
```

**问题**：RouterOS 按顺序匹配，规则 A 先命中（而且因为没指定 `in-interface`，连 LAN 内部流量也会匹配），规则 B 永远不执行。

### 问题 2：srcnat 硬编码公网 IP

```routeros
add action=masquerade chain=srcnat out-interface=pppoe-out1 protocol=udp src-address=192.168.50.7 src-port=41641 to-addresses=202.184.141.156 to-ports=41641
```

**问题**：PPPoE 动态 IP 环境下，`to-addresses` 硬编码了一个旧的公网 IP。IP 变更后，Tailscale 发出的 UDP 包源地址被改写为错误的 IP，STUN 响应回不来。

### 问题 3：FastTrack 干扰 NAT 穿透

```routeros
/ip firewall filter
add action=fasttrack-connection chain=forward connection-mark=!no-fasttrack connection-state=established,related
```

**问题**：FastTrack 把已建立的 UDP 连接绕过了防火墙规则处理。Tailscale 的 NAT 穿透需要 **Endpoint-Independent Mapping**（即对外端口不变），但 FastTrack 后的连接跳过了 mangle 标记流程，导致端口映射不一致，STUN 打洞失败。

### 问题 4：Mangle 规则不完整

```routeros
/ip firewall mangle
add action=mark-connection chain=forward new-connection-mark=no-fasttrack protocol=udp src-address=192.168.50.7 src-port=41641
add action=mark-connection chain=forward dst-address=192.168.50.7 dst-port=41641 new-connection-mark=no-fasttrack protocol=udp
```

**问题**：只标记了设备 A（50.7）的 Tailscale 流量绕过 FastTrack。设备 B（50.20）上也运行 Tailscale，但它的 UDP 流量照样被 FastTrack 加速，打洞失败。

## 正确修复方案

### 第一步：确定哪个设备接收入站 Tailscale

两个设备都运行 Tailscale，但入站端口只能转发给一个。选定主设备后：

### 第二步：清除旧规则并重新配置

```routeros
# === 清除所有 Tailscale 相关规则 ===
/ip firewall filter remove [find comment~"Tailscale"]
/ip firewall nat remove [find comment~"Tailscale"]
/ip firewall mangle remove [find comment~"Tailscale"]

# === DSTNAT: 入站 UDP 41641 转发到指定设备 ===
/ip firewall nat add chain=dstnat action=dst-nat \
    in-interface=pppoe-out1 protocol=udp dst-port=41641 \
    to-addresses=192.168.50.20 to-ports=41641 \
    comment="Tailscale: Forward inbound UDP 41641"

# === Mangle: 所有 Tailscale 流量绕过 FastTrack ===
# 用 port= 同时匹配 src-port 和 dst-port
/ip firewall mangle add chain=forward action=mark-connection \
    new-connection-mark=no-fasttrack passthrough=yes \
    protocol=udp port=41641 \
    comment="Tailscale: Bypass fasttrack"

# === Filter: 允许入站 Tailscale ===
/ip firewall filter add chain=forward action=accept \
    in-interface=pppoe-out1 protocol=udp dst-port=41641 \
    dst-address=192.168.50.20 \
    comment="Tailscale: Accept inbound" \
    place-before=[find comment="drop all other forward"]
```

### 第三步：验证

```bash
# 在 Tailscale 设备上
tailscale netcheck
# 检查: UDP = true, MappingVariesByDestIP = false

tailscale ping <peer-ip>
# 应该看到 "via DERP" 变成 "direct"
```

## 额外注意：CGNAT 检查

如果 ISP 使用 CGNAT，以上所有修复都无效：

```bash
# 比较 PPPoE 获取的 IP 和外部显示的 IP
# RouterOS:
:put [/ip address get [find interface=pppoe-out1] address]

# 外部:
curl ifconfig.me
```

如果两个 IP 不同，你在 CGNAT 后面，端口转发不可能工作。需要联系 ISP 申请公网 IP，或者完全依赖 DERP 中继。

## 教训总结

1. **一个端口只转发给一个设备** — 多条 dstnat 匹配同一端口，只有第一条生效
2. **不要硬编码动态 IP** — PPPoE 用 masquerade 而非 srcnat with to-addresses
3. **FastTrack 和 VPN 天生冲突** — 任何需要 NAT 穿透的流量都必须绕过 FastTrack
4. **Mangle 标记要覆盖所有相关设备** — 漏掉一个设备就等于没标记
