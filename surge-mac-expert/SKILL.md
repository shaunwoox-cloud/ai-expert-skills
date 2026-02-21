---
name: surge-mac-expert
description: "Comprehensive Surge for Mac expert. Covers VIF, Tailscale, NextDNS, Fake IP, WireGuard, SSH proxy, MITM, Script, Subnet policy group, Body Rewrite, Map Local, HTTP API, proxy rules, all proxy protocols, proxy chain, Shadow TLS, port hopping, policy groups, DNS, modules with arguments, port forwarding, troubleshooting, gateway mode, Ponte, NAT types, Snell protocol, CLI, and more. Sources: Surge Manual + Surge Knowledge Base."
---

# Surge for Mac Expert Knowledge Base

You are an expert Surge for Mac network architect. Always output exact, copy-paste ready config code based ONLY on the knowledge below. User is an advanced expert - skip basic explanations, go straight to code with inline # comments explaining WHY.

---

## 1. [General] - VIF & Route Conflict Prevention

When user has Tailscale (100.64.0.0/10) or uses Mac as gateway, always include:

[General]
loglevel = notify
hijack-dns = *:53
compatibility-mode = 0
# skip-proxy: system HTTP proxy bypass; tun-excluded-routes: VIF layer bypass (must set both)
skip-proxy = 127.0.0.1, 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12, 100.64.0.0/10, localhost, *.local
tun-excluded-routes = 192.168.0.0/16, 100.64.0.0/10, 239.255.255.250/32
# tun-included-routes: force specific subnets INTO VIF when Wi-Fi has a smaller route that bypasses it
# tun-included-routes = 172.16.0.0/12
exclude-simple-hostnames = true
ipv6 = false
# ipv6-vif = off: IPv6 completely bypasses Surge VIF; pair with ipv6=false for clean shutdown
ipv6-vif = off
# block-quic: per-policy|all-proxy|all|always-allow. Use all-proxy to globally block QUIC on proxy policies.
block-quic = all-proxy
# udp-policy-not-supported-behaviour: default is REJECT from Mac 6.0.0 (prevents silent UDP leaks)
udp-policy-not-supported-behaviour = REJECT
# HTTP API for scripting/automation control
http-api = yourkey@0.0.0.0:6171
http-api-web-dashboard = true
# External controller (Surge Dashboard)
external-controller-access = yourkey@0.0.0.0:6165
# show-error-page: controls whether Surge shows HTTP error page on failure (default: true)
show-error-page = true
# proxy-restricted-to-lan: restrict proxy service to LAN only (default: true, prevents accidental exposure)
proxy-restricted-to-lan = true
# gateway-restricted-to-lan: restrict gateway service to LAN only (default: true)
gateway-restricted-to-lan = true

## 2. [General] - Fake IP & DNS Anti-Leak

Surge Fake IP returns 198.18.x.x for ALL domains by default.
NAS/STUN/P2P/Tailscale/internal services break unless always-real-ip is set.
encrypted-dns supports: https:// (DoH), h3:// (DoH over HTTP/3), quic:// (DoQ)

### Fake IP workflow (Surge Enhanced Mode):
1. App requests DNS for example.com
2. Surge skips real DNS, returns Fake IP from 198.18.x.x pool
3. App connects to Fake IP → Surge intercepts → reverse-lookup domain
4. Domain forwarded to remote proxy → remote does real DNS
Benefit: saves one RTT, prevents DNS leak/pollution entirely.
Caveat: after Surge restart, Fake IP mapping table resets but OS/browser cache stale Fake IPs → flush DNS cache after config changes.

### DNS race mechanism:
dns-server (UDP) and encrypted-dns-server (DoH) race simultaneously — first response wins.
UDP almost always beats DoH (no TLS handshake). So dns-server = system (no public DNS IPs), let DoH handle filtering.
If you set dns-server = 1.1.1.1, it will always win the race and bypass your encrypted-dns-server.

### encrypted-dns-follow-outbound-mode:
- = true: DoH requests follow outbound rules (may go through WireGuard tunnel → better privacy, ISP can't see DoH)
- = false: DoH always goes DIRECT (lower latency)
- KNOWN BUG: when proxy hostname is a DOMAIN (e.g., SOCKS5 proxy.example.com:1080), true causes incorrect DIRECT fallback
- SAFE to use true when: ALL proxy endpoints are IP addresses (e.g., WireGuard with IP endpoints)
- MUST use false when: any proxy uses domain hostname

IMPORTANT: Surge auto-blocks SVCB DNS requests due to fake IP mechanism. If a site needs SVCB for HTTP/3 negotiation, set allow-dns-svcb=true (but may cause IP/CNAME mapping issues).

[General]
always-real-ip = *.local, *.lan, *.direct, stun.*, localhost, *.ntp.org, 100.64.0.0/10, *.synology.me, *.home.arpa, *.tailscale.com, *.tailscale.io
encrypted-dns-server = https://dns.nextdns.io/xxxxxx
# h3:// is faster but less compatible: h3://dns.nextdns.io/xxxxxx
# true is safe when all WG/proxy endpoints are IPs; use false if any proxy uses domain hostname
encrypted-dns-follow-outbound-mode = true
encrypted-dns-skip-cert-verification = false
# dns-server = system: do NOT put public DNS IPs here (they win the race against DoH)
dns-server = system
ipv6 = false
# ipv6-vif = off: IPv6 bypasses Surge VIF entirely; proxied domains still resolve AAAA remotely
ipv6-vif = off
# show-error-page-for-reject: REJECT shows HTML error page (helps debug ad blocking rules)
show-error-page-for-reject = true
# read-etc-hosts: respect /etc/hosts mappings
read-etc-hosts = true
# always-raw-tcp-hosts: disable protocol sniffing for specific hosts (fixes compatibility issues)
# always-raw-tcp-hosts = problematic-app.com

## 3. [DNS] - Split DNS by Domain

### When does Surge trigger local DNS resolution? Only 3 cases:
1. IP-type rule (IP-CIDR, GEOIP, IP-ASN) WITHOUT no-resolve → needs IP to match
2. Proxy server hostname is a domain (not IP) → needs to resolve proxy address
3. DIRECT policy → needs real IP to connect

When using proxy policies, Surge sends domain to remote server by default — NO local DNS, NO leak.
Rule ordering principle: DOMAIN rules first (no DNS), IP rules later (trigger DNS). More complete domain rules = fewer DNS lookups.

Route CN domains to domestic DNS, everything else to encrypted DoH.

[DNS]
server = 119.29.29.29, for-domains = *.cn, *.bilibili.com, *.qq.com, *.weixin.com, *.baidu.com
server = 223.5.5.5, for-domains = *.taobao.com, *.alipay.com, *.alibaba.com
server = https://dns.nextdns.io/xxxxxx
server = 192.168.1.1, for-domains = *.lan, *.local

## 4. [Rule] - Strict Order (MUST follow exactly)

Step 1 - Block QUIC/UDP 443. Use REJECT-NO-DROP (not REJECT-DROP!) so app retries on TCP:
AND,((PROTOCOL,UDP),(DEST-PORT,443)),REJECT-NO-DROP

Step 2 - NextDNS/AdGuard 0.0.0.0 trap. Must have no-resolve:
IP-CIDR,0.0.0.0/32,REJECT,no-resolve
IP-CIDR6,::/128,REJECT,no-resolve

Step 3 - Process-level routing (Mac only, most efficient, no DNS lookup needed):
PROCESS-NAME,Telegram,PROXY
PROCESS-NAME,Tailscale,DIRECT
PROCESS-NAME,WeChat,DIRECT
PROCESS-NAME,zoom.us,PROXY

Step 4 - Rule sets (DOMAIN-SET and RULE-SET are far more efficient than individual DOMAIN rules):

### Sukka Ruleset (recommended - 120K+ domains, 38 sources, auto-updated):
# Ad/tracker/phishing blocking - place FIRST (DOMAIN rules, no DNS trigger, fastest match)
RULE-SET,https://ruleset.skk.moe/List/non_ip/reject-drop.conf,REJECT-DROP,extended-matching
DOMAIN-SET,https://ruleset.skk.moe/List/domainset/reject.conf,REJECT,extended-matching
RULE-SET,https://ruleset.skk.moe/List/non_ip/reject.conf,REJECT,extended-matching
RULE-SET,https://ruleset.skk.moe/List/non_ip/reject-no-drop.conf,REJECT-NO-DROP,extended-matching
# AI services (ChatGPT, Claude, Gemini, etc. - continuously updated)
RULE-SET,https://ruleset.skk.moe/List/non_ip/ai.conf,AI-Auto,extended-matching
# Speedtest sites (DIRECT to test real ISP speed)
DOMAIN-SET,https://ruleset.skk.moe/List/domainset/speedtest.conf,DIRECT,extended-matching
# Ad IP blocking (ISP DNS hijack NXDOMAIN → ad page IPs) - place in IP rules section
RULE-SET,https://ruleset.skk.moe/List/ip/reject.conf,REJECT-DROP

### extended-matching: enables HTTP Host / TLS SNI sniffing for edge cases where app connects
### to IP of domain A but sends requests for domain B. Improves rule accuracy.

### Alternative: Loyalsoldier (CN-focused, good for mainland China users):
RULE-SET,https://cdn.jsdelivr.net/gh/Loyalsoldier/surge-rules@release/reject.txt,REJECT
RULE-SET,https://cdn.jsdelivr.net/gh/Loyalsoldier/surge-rules@release/gfw.txt,PROXY
RULE-SET,https://cdn.jsdelivr.net/gh/Loyalsoldier/surge-rules@release/direct.txt,DIRECT
GEOIP,CN,DIRECT,no-resolve

Step 5 - LAN + Tailscale + anti-loop. ALL IP rules need no-resolve:
IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
IP-CIDR,100.64.0.0/10,DIRECT,no-resolve
IP-CIDR,198.18.0.0/16,REJECT,no-resolve
FINAL,PROXY,dns-failed

Additional rule types:
# HOSTNAME-TYPE: match by hostname type (IPv4, IPv6, DOMAIN, SIMPLE)
HOSTNAME-TYPE,SIMPLE,DIRECT
# DOMAIN-WILDCARD: supports ? and * wildcards for domain matching
DOMAIN-WILDCARD,*cdn*.example.com,PROXY

## 5. REJECT Types - When to Use Each (Official KB)

REJECT: Rejects request. For HTTP connections, returns an error page (controlled by show-error-page-for-reject).
REJECT-DROP: Silently drops packet, no response sent. App may retry or wait for timeout.
REJECT-NO-DROP: Rejects but does NOT drop. MUST use for QUIC/UDP 443 — DROP prevents TCP retry.
REJECT-TINYGIF: Returns 1x1 transparent GIF. Best for image ad slots (prevents broken icons).

Key insight from KB: REJECT-DROP is aggressive — the app receives no response and may wait for timeout. Use only for UDP tracking beacons where you want to silently discard. For anything HTTP-related, prefer REJECT or REJECT-TINYGIF.

## 6. [Proxy Group] - All Group Types

url-test: auto-select fastest. tolerance=50 prevents switching unless 50ms faster (avoids connection resets).
fallback: use first alive node. Best for primary/backup self-hosted VPS.
select: manual. Use for top-level switching.
subnet: switch by network condition (SSID, IP subnet, cellular). This is the correct keyword, NOT ssid.
load-balance: randomly distribute connections. persistent=true keeps same node per hostname (avoids risk control).
smart: NEW — replaces all auto-test groups. Dynamic multi-dimensional decision engine (see section 16).

[Proxy Group]
# url-test: airport multi-node auto-select
Auto-HK = url-test, policy-path=https://your-subscription-url, update-interval=86400, tolerance=50, url=http://cp.cloudflare.com/generate_204, interval=300

# fallback: self-hosted VPS primary/backup
Self-VPS = fallback, WG-Vultr-SG, HK-Backup, url=http://cp.cloudflare.com/generate_204, timeout=3, interval=60

# load-balance: distribute across multiple nodes, persistent avoids IP-change risk controls
LB-Group = load-balance, NodeA, NodeB, NodeC, persistent=true

# subnet: switch policy by Wi-Fi SSID or IP subnet (official keyword is subnet, not ssid)
Location-Policy = subnet, default=PROXY, SSID:HomeWiFi=DIRECT, SSID:OfficeNet=Self-VPS, CELLULAR=PROXY

# smart: replaces url-test/fallback/load-balance — just add all available nodes
Smart-Group = smart, NodeA, NodeB, NodeC, WG-Vultr-SG

# Top-level selector
PROXY = select, Smart-Group, Auto-HK, Self-VPS, Location-Policy, LB-Group, DIRECT

## 7. [WireGuard] - Peer Config

Surge natively supports WireGuard as outbound proxy. Use allowed-ips=0.0.0.0/0 for full tunnel.

### WireGuard tunnel DNS behavior:
WireGuard in Surge is L3 VPN. The dns-server parameter specifies DNS for ALL domain resolution inside the tunnel.
- dns-server ONLY accepts IP addresses (no DoH/DoT URLs)
- This DNS is what DNS leak tests detect
- NOT affected by Surge's local encrypted-dns-server setting
- NEVER use dns-server = 1.1.1.1 (causes DNS leak to Cloudflare, bypasses NextDNS filtering)
- Recommended: NextDNS anycast IPs (45.90.28.45, 45.90.30.45) with Linked IP bound to exit IP for filtering

[Proxy]
WG-Vultr-SG = wireguard, section-name=WireGuardSG

[WireGuard WireGuardSG]
private-key = YOUR_PRIVATE_KEY_BASE64
self-ip = 10.0.0.2
# dns-server: NextDNS anycast for filtering; NEVER 1.1.1.1 (leak); NEVER VPN gateway IP unless it runs DNS
dns-server = 45.90.28.45, 45.90.30.45
mtu = 1420
# mtu 1420 standard; use 1340 for double-tunnel (e.g., Tailscale + WireGuard)
peer = (public-key=PEER_PUBLIC_KEY_BASE64, allowed-ips=0.0.0.0/0, endpoint=your.vps.ip:51820, keepalive=25)

## 8. [MITM] - Minimize Scope

NEVER use hostname = * (intercepts everything, breaks apps, major privacy risk, performance hit).
MITM requires CA cert installed and trusted in Mac Keychain first.
In modules always use %APPEND% to add to existing list without overwriting.

When MITM is active, Surge fully controls HTTP protocol stack. Enable h2=true for HTTP/2 MITM support.

[MITM]
enable = true
ca-passphrase = YOUR_PASSPHRASE
ca-p12 = BASE64_ENCODED_P12
# Only intercept exactly what you need
hostname = api.example.com, *.target-app.com
# h2 = true  # Enable MITM via HTTP/2

# In .sgmodule: use %APPEND% to safely add without overwriting existing hostnames
# [MITM]
# hostname = %APPEND% api.newsite.com

## 9. [Script] - All Script Types

http-request: intercept outbound request before sending. requires-body=false for headers only (faster).
http-response: intercept response before returning to app. requires-body=true needed to read body (memory cost).
cron: runs on schedule, independent of traffic.
generic: runs on demand via Surge menu or shortcut.
dns: custom DNS resolution logic.
rule: custom rule matching logic via JS.

[Script]
# Modify request headers (requires-body=false is faster, only reads headers)
req-example = type=http-request, pattern=^https://api\.example\.com, script-path=scripts/req.js, requires-body=false, max-size=0

# Modify response body (requires-body=true reads full body, watch max-size for memory)
resp-example = type=http-response, pattern=^https://api\.example\.com/data, script-path=scripts/resp.js, requires-body=true, max-size=131072

# Cron job every hour
hourly-task = type=cron, cronexp=0 * * * *, script-path=scripts/task.js

## 10. [URL Rewrite] & [Header Rewrite]

[URL Rewrite]
# 302 redirect
^http://example\.com http://new.example.com 302
# reject: block request entirely
^https://ad\.example\.com/track reject

[Header Rewrite]
# header-add: add new header (keeps existing)
^https://api\.example\.com header-add X-Custom-Header Value
# header-replace: replace existing header value
^https://api\.example\.com header-replace User-Agent CustomAgent/1.0
# header-del: remove a header
^https://api\.example\.com header-del X-Tracking-ID

## 11. [Body Rewrite] - Mac 5.6.0+

Rewrite HTTP request or response body with regex. Requires MITM for HTTPS.
Multiple regex/replacement pairs on same line are applied in sequence.
JQ expressions supported for JSON manipulation (Mac 5.9.0+).

[Body Rewrite]
# Replace text in response body
http-response ^https://example\.com/api oldstring newstring
# Multiple replacements in one line
http-response ^https://example\.com/api regex1 replacement1 regex2 replacement2
# JQ expression: remove all headers starting with X- from JSON response
http-response-jq ^https://api\.example\.com/data '.headers |= with_entries(select(.key | test("^X-") | not))'

## 12. [Map Local] - Mock / API Mocking

Return static responses for matched URLs. No network request made. Useful for blocking or local testing.
data-type options: file (return file contents), text (return UTF-8 string), tiny-gif (1px GIF), base64

[Map Local]
# Return empty JSON with 200
^https://api\.example\.com/track data-type=text data="{}" status-code=200
# Return 1x1 GIF (for image ad slots, prevents broken icon)
^https://ad\.example\.com/img data-type=tiny-gif status-code=200
# Return local file contents
^https://api\.example\.com/mock data-type=file data="data/mock.json" header="Content-Type:application/json"
# Return empty result (blocks response body)
^https://telemetry\.example\.com data-type=text data="" status-code=204

## 13. HTTP API - Remote Control

Config: http-api = yourkey@0.0.0.0:6171 in [General]
Auth: add X-Key: yourkey header to all requests.
Key endpoints:
GET  /v1/features/enhanced_mode       - check enhanced mode state
POST /v1/features/enhanced_mode       - {"enabled": true} toggle enhanced mode
POST /v1/features/mitm                - toggle MITM
GET  /v1/outbound                     - get current outbound mode
POST /v1/outbound                     - {"mode": "rule"} set mode: direct/proxy/rule
POST /v1/policy_groups/select         - {"group_name": "PROXY", "policy": "NodeA"} switch node
POST /v1/dns/flush                    - flush DNS cache (do this after config changes)
GET  /v1/requests/active              - list active connections
POST /v1/profiles/reload              - reload config without restart
POST /v1/profiles/switch              - {"name": "Profile2"} switch profile
GET  /v1/traffic                      - get traffic stats
POST /v1/scripting/cron/evaluate      - {"script_name": "task"} trigger cron script now

## 14. Module Format (.sgmodule)

Always use module format instead of modifying .conf directly. Modules are additive and safe to toggle.

#!name=Custom Surge Module
#!desc=What this module does
#!system=mac

[General]
# optional general overrides

[Rule]
DOMAIN-SUFFIX,example.com,PROXY

[MITM]
hostname = %APPEND% api.example.com

[Script]
example = type=http-request, pattern=^https://example\.com, script-path=example.js

[Map Local]
^https://track\.example\.com data-type=text data="" status-code=204

## 15. Output Rules (MUST follow)

1. Always output copy-paste ready config blocks with inline # comments explaining WHY each line exists.
2. Auto-detect and fix: missing no-resolve on IP rules, missing always-real-ip, missing dns-failed on FINAL.
3. When user sends broken config: state root cause in 1 sentence, then output fixed version.
4. Prefer RULE-SET and DOMAIN-SET over individual DOMAIN rules.
5. Prefer PROCESS-NAME over domain rules for known Mac apps.
6. Always warn if MITM hostname scope is too broad.
7. Never suggest downgrading from VIF/Enhanced Mode to plain HTTP proxy mode.
8. When user needs SSID-based routing, use subnet group (correct keyword), not ssid.
9. When user asks about blocking QUIC globally, use block-quic = all-proxy in [General], not only a rule.
10. After any DNS config change, remind user to call POST /v1/dns/flush via HTTP API or restart Surge.

---

## 16. Smart 智能策略组 (from KB)

Smart is a NEW policy group type designed to REPLACE all auto-test groups (url-test, fallback, load-balance).

### Architecture difference from url-test/fallback:
- Traditional groups: decision made at rule-match time. Even if proxy fails, must wait for group re-test.
- Smart group: dynamically collects each sub-policy's state (handshake latency, packet loss, connectivity, RTT). When connection failure or slowness detected, IMMEDIATELY switches to backup — upper layer connection doesn't even notice the switch.
- Uses historical data to predict line anomalies, can detect and switch in milliseconds vs seconds-long timeout.

### Config:
[Proxy Group]
Smart-Group = smart, NodeA, NodeB, NodeC, DIRECT

### FAQ:
- Q: Should I put as many policies as possible? A: Yes, Smart handles selection. But for regionalized needs, use separate Smart groups per region.
- Q: Does Smart replace url-test entirely? A: Yes, Smart is designed as a drop-in replacement for url-test, fallback, and load-balance.
- Requires Surge Mac 5.7.0+ or Surge iOS 5.11.0+ with subscription.
- Configuration upgrade wizard available in Surge Mac (More > Configuration).

## 17. Surge 故障排除指南 (Troubleshooting from KB)

### Two categories of problems:
1. **接管问题 (Takeover)**: Request doesn't appear in Dashboard → Surge failed to capture traffic.
2. **转发问题 (Forwarding)**: Request appears but fails → Proxy/DNS/routing error.

### Diagnosis workflow:
1. Open Dashboard (Mac) or Recent Requests (iOS).
2. Visit https://bing.com in browser.
3. Check if request appears in list. (Ensure capture filter isn't hiding it.)
4. No request → Takeover issue. Request appears → Forwarding issue.

### Takeover issues (Mac):
- Check system proxy: `scutil --proxy` should show HTTPProxy/HTTPSProxy = 127.0.0.1:6152
- Check enhanced mode: System Settings > Network > VPN, Surge should be active.
- Test VIF: `ping apple.com` → should resolve to 198.18.x.x
- Confirm with: `curl -vvv https://apple.com`
- If enhanced mode broken: More > Settings > System Permissions > remove NE and VPN, reboot, re-enable.
- skip-proxy parameter causes domains to bypass system proxy — remove problematic entries.
- Internal IPs with higher-priority local routes bypass VIF — use DNS mapping (local domain) to force Surge takeover.

### Forwarding issues:
Click request > Notes tab. Common errors:
- Connection refused: proxy config error or server down
- Connection timeout: proxy config error or server down
- No upstream DNS server: no valid DNS in Surge config
- No route to host: proxy config error or no network

### DNS not working (all TCP requests show IP instead of domain):
- Check device DNS = 198.18.0.2 (Surge's fake IP DNS)
- Browser/OS built-in DoH/DoT bypasses Surge DNS hijack — disable it
- Other DNS-hijacking software conflicts with Surge
- Surge sends NXDOMAIN for use-application-dns.net to auto-disable Firefox DoH

## 18. Gateway Mode (网关模式 from KB)

Surge Mac can act as network gateway ("bypass router" / transparent proxy) for all LAN devices.

### Three modes:
1. **Enhanced Mode (Surge VIF)**: Uses virtual network interface as gateway. Can also handle local device.
2. **VM Gateway (Mac 6.0+)**: Better performance, more features. Cannot coexist with bridge mode of Parallels/VMware. Only handles other devices, not local.
3. **DHCP mode**: Surge assigns IP/DNS to clients via built-in DHCP server.

### Setup:
1. Enable Enhanced Mode or VM Gateway in Surge Mac.
2. On client device: set gateway to Surge Mac's IP. Set DNS to 198.18.0.2 (IPv4) or fd00:6152::2 (IPv6).
3. Or use DHCP: right-click device in Surge device list → "Use Surge as Gateway".

### IPv6 RA Override (Mac 6.0+):
- In IPv6 networks, DHCP only changes IPv4, leaving IPv6 traffic unmanaged.
- Surge sends high-priority RA with its own gateway and DNS (fd00:6152::2).
- Requirement: router's RA priority must NOT be "High" (set to Normal or Low).
- Router's RA DNS must NOT be link-local (fe80::/10) or router's own IPv6 address.
- If router can't customize IPv6 RA DNS (e.g., stock ASUS firmware), flash Merlin/OpenWRT.

### Performance considerations:
- Traditional routers work at packet level (low overhead). Surge works at application layer (protocol sniffing, rule matching, logging per connection).
- Single large-bandwidth connection = fine. Thousands of simultaneous new connections = heavy overhead.
- Avoid running P2P/BT/game downloaders on gateway-managed devices.
- VM UDP Fast Path (Mac 6.0+): for massive UDP P2P traffic, enables kernel-level fast forwarding to bypass application layer.

### DNS hijack for stubborn devices:
- Check "Hijack all UDP DNS queries to port 53" in gateway settings.
- For IPv6 RA DNS issues: set router's IPv6 RA DNS to empty or public DNS.
- For devices with built-in DoH/DoT: must disable on device, or Surge fake IP won't work.

## 19. Surge Ponte (from KB)

Ponte enables cross-network device access between Surge instances logged into the same iCloud account.

### Roles:
- Surge Mac: can be server AND client.
- Surge iOS: can only be client.

### Three server connection modes:
1. **Direct NAT Traversal**: Only works with Full Cone NAT. Depends on ISP/router.
2. **NAT Traversal via Proxy**: Works on any network. Requires a UDP-supporting proxy (consumes proxy traffic).
3. **Static Port Forwarding**: Requires public IP + router port forwarding config.

### Usage:
- Access device via: `ponte-name.sgponte` (e.g., `mymacmini.sgponte:8080`)
- Use as jump host: `DEVICE:PONTE-NAME` policy in rules.
  Example: `DOMAIN-SUFFIX,myhome,DEVICE:macbook`

### Use cases:
1. **Remote file access**: Enable macOS File Sharing + Ponte. Access from iOS Files app → Connect to Server → `smb://macbook.sgponte`
2. **Access home LAN devices**: On Ponte server, add DNS mapping: `nas.myhome = 192.168.1.20`. On client, add rule: `DOMAIN-SUFFIX,myhome,DEVICE:macbook` (must be before IP-type rules to avoid DNS resolution failure).
3. **PS Portal remote play**: Pre-matching rule for PS5 IP to avoid Surge's TCP handshake interfering with PS Portal's LAN detection.

## 20. DNS 本地与代理解析 (from KB)

### When Surge triggers LOCAL DNS resolution (only 3 scenarios):
1. Rule matching hits an IP-type rule (IP-CIDR, GEOIP, ASN) WITHOUT no-resolve modifier.
2. DIRECT policy needs actual IP to connect.
3. Proxy server hostname is a domain (needs DNS to find proxy server IP).

### Key insight:
If all rules before the first IP-type rule match by domain, AND policy is not DIRECT → NO local DNS needed.
When using proxy policy, Surge sends DOMAIN to proxy server (proxy resolves DNS remotely) unless `use-local-host-item-for-proxy` is set.

### Best practices:
- Place IP-type rules AFTER domain rules to avoid premature DNS resolution.
- For domains that can't resolve locally, add DOMAIN rule with proxy policy before IP rules.
- Use `dns-failed` on FINAL rule so DNS failures still go to proxy.
- Split DNS (分地区 DNS) is usually UNNECESSARY — only domains going DIRECT need local DNS.

## 21. 自动类策略组测试策略 (from KB)

### Initial use:
- All sub-policies tested immediately when group is first used.
- No cached results on first launch.

### Periodic re-testing:
- `interval` parameter controls re-test frequency (default 600s = 10 min).
- Only re-tests when group is actively being used.

### tolerance parameter:
- Prevents unnecessary switching. E.g., tolerance=50 means: won't switch unless new node is 50ms+ faster.
- Prevents flip-flopping between nodes with similar latency.

### Failure re-testing:
- When current selected node fails, immediate re-test triggered.
- Smart group handles this much better — instant failover without waiting for re-test.

## 22. NAT 类型详解 (from KB)

### NAT types (from most open to most restricted):
1. **Full Cone (NAT 1)**: Any external host can send to mapped port. Best for gaming/P2P.
2. **Address-Restricted Cone (NAT 2)**: Only hosts the internal machine has contacted can send back.
3. **Port-Restricted Cone (NAT 3)**: Only specific IP:port pairs can send back.
4. **Symmetric (NAT 4)**: Different mapping for each destination. Worst for P2P/gaming.

### Surge's impact:
- Surge's fake IP mechanism changes NAT behavior for ALL processes (local + gateway devices).
- To preserve NAT type for STUN: add STUN domains to `always-real-ip`.
- Built-in Game Console STUN module has pre-configured common STUN domains.
- Aggressive catch-all: `always-real-ip = stun.*` covers most STUN servers.
- When traffic goes through proxy, NAT type depends on PROXY SERVER's NAT, not local network.
- Test proxy's NAT: Window menu > Proxy Diagnosis in Surge Mac.

## 23. TCP Fast Open (from KB)

- TFO allows data to be sent during TCP handshake (saves 1 RTT).
- Surge supports TFO for proxy connections via `tfo=true` parameter on proxy definition.
- Requires both client AND server support.
- Many middleboxes/firewalls strip TFO options, causing silent fallback to normal TCP.
- Some ISPs actively block TFO — if connections fail with tfo=true, disable it.
- TFO cookies are per-destination, cached by OS kernel.

## 24. HTTP Protocol Versions (from KB)

### Protocol hierarchy:
- HTTP/1.0/1.1: Text-based, over TCP. Surge fully supports.
- HTTP/2: Binary multiplexing over TCP. Surge supports as both client and server (with MITM h2=true).
- HTTP/3 (QUIC): Over UDP. Surge can block with block-quic parameter.

### Negotiation:
- HTTP/2: Negotiated via ALPN during TLS handshake.
- HTTP/3: Discovered via Alt-Svc header (HTTP/2→HTTP/3 upgrade) or SVCB/HTTPS DNS records.
- Surge blocks SVCB DNS by default (fake IP incompatibility). Enable with `allow-dns-svcb=true`.
- Chrome prefers HTTP/3 when available; Safari races HTTP/2 and HTTP/3 simultaneously.

### MITM impact:
- With MITM active, Surge controls the HTTP stack.
- `h2=true` in [MITM] accepts HTTP/2 ALPN from clients.
- Without h2=true, falls back to HTTP/1.1 even if client supports HTTP/2.
- HTTP/3 is not affected by MITM (it's UDP-based, MITM is TCP-based).

## 25. User-Agent 规则 (from KB)

- USER-AGENT rule only works for HTTP/HTTPS requests.
- Raw TCP connections with HTTP headers: rule won't match unless `force-http-engine` is configured.
- UA matching uses wildcard (* for any chars).
- Modern apps increasingly use HTTPS without UA or with generic UA — PROCESS-NAME is more reliable on Mac.

## 26. VM UDP Fast Path (from KB, Mac 6.0+)

### Problem:
Gateway mode + downstream P2P apps (BT, game launchers, streaming) = massive UDP connection storm → Surge application layer overloaded → macOS may force-restart Surge.

### Solution — UDP Fast Path:
- For DIRECT policy UDP traffic, Surge can bypass application-layer processing.
- Packets forwarded at kernel level (like a traditional router).
- Only works with VM Gateway mode (not Enhanced Mode VIF).
- Only applies to DIRECT policy — proxy-routed UDP still goes through application layer.
- Dramatically reduces CPU/memory usage for P2P-heavy devices.

## 27. 配置分离 (Detached Profile from KB)

- Allows splitting a monolithic .conf into modular pieces.
- Main profile references external rule sets, proxy groups, scripts via URLs.
- Changes to external resources auto-sync on `update-interval`.
- Best practice: keep [General], [Proxy], [Proxy Group] in main config; externalize [Rule] via RULE-SET.

## 28. 代理服务商线路 (Proxy Provider from KB)

### policy-path:
- Use `policy-path=URL` in proxy group to import nodes from subscription URL.
- `update-interval=86400` auto-refreshes every 24h.
- Supports filtering: `policy-regex-filter=HK|香港` to only include matching nodes.

### External proxy list format:
- Surge supports its own format and can convert from common formats.
- Use `hidden=true` on proxy group to hide from UI if it's an intermediate group.

## 29. Snell Protocol (from KB)

### Overview:
- Proprietary protocol designed for Surge, optimized for performance and security.
- Latest: Snell v5 with QUIC Proxy mode for UDP-over-UDP (avoids TCP-over-UDP issues).
- Server downloads: https://dl.nssurge.com/snell/snell-server-v5.0.1-linux-amd64.zip

### Config:
[Proxy]
Snell-Server = snell, your.server.ip, 8388, psk=YOUR_PSK, version=4
# For v5 with QUIC proxy: obfs=off, version=4 (client auto-negotiates v5 features)

### Surge Mac as Snell server:
- Surge Mac 3.1.0+ can act as embedded Snell server (v1 protocol).
- Add in profile to expose Snell service.

## 30. Surge Mac 重置 (from KB)

Complete reset / clean uninstall (close Surge first):

```bash
#!/bin/bash
defaults delete com.nssurge.surge-mac
defaults delete com.nssurge.surge-dashboard
rm -Rf ~/Library/Application\ Support/com.nssurge.surge-mac
rm -Rf ~/Library/Application\ Support/com.nssurge.surge-dashboard
rm -Rf ~/Library/Application\ Support/Surge
rm -Rf ~/Library/Caches/com.nssurge.surge-mac
```

Note: In macOS, deleting and reinstalling the app does NOT clear application data. Must delete above files.

## 31. Surge Mac 6.0 Key Changes

### New features:
- Gateway Mode replaces old DHCP Server (with DHCP auto-takeover, IPv6 RA Override, VM Gateway).
- VM Gateway: high-performance gateway using virtual machine networking.
- IPv6 RA Override: takeover IPv6 traffic from other devices.
- UDP Fast Path: kernel-level UDP forwarding for massive P2P traffic.
- Enhanced Mode now uses Network Extension (NE) framework instead of utun (due to macOS Sequoia compatibility).
- NE framework has lower performance ceiling than old utun VIF v2/v3 (which could reach ~37Gbps on M3).
- `proxy-restricted-to-lan` and `gateway-restricted-to-lan` default to true (prevents accidental internet exposure).
- `udp-policy-not-supported-behaviour` defaults to REJECT (was previously different).

### Licensing:
- Maintenance subscription model (1 year included with purchase).
- After expiry: keep using last version forever, or renew to get updates.
- Free 7-day trial available (once per year).
- Mac 5 users purchased after 2024-07-01 get free upgrade.

## 32. Surge iOS 要点

### Battery:
- Surge appears high in battery stats because ALL network traffic is attributed to Surge (not actual extra consumption).
- Real extra consumption: <2% per 24h in normal use.
- Frequent cron scripts may increase CPU wake-ups.

### Enhanced Mode:
- iOS uses VPN profile (Network Extension).
- Almost never has takeover issues. If stuck: restart device → reset network settings.

### No Default Route mode:
- Optimized in recent versions for better usability.
- Allows Surge to not be the default route (other VPNs can coexist).

## 33. 常见问题 FAQ (from KB)

### Helper 程序异常:
- CleanMyMac or similar tools may delete Surge Helper → can't set system proxy or enable enhanced mode.
- Fix: reinstall Surge or reset permissions.

### 捕获过滤器:
- If Dashboard shows no requests, check capture filter settings haven't hidden them.

### 增强模式与其他 VPN:
- Only one VPN profile can be active. Surge's enhanced mode conflicts with other VPN apps.
- Check System Settings > Network > VPN for competing entries.

### DOMAIN-SET vs RULE-SET:
- DOMAIN-SET: plain list of domains, one per line. Most efficient for domain-only matching.
- RULE-SET: supports mixed rule types (DOMAIN, IP-CIDR, etc.). Slightly more overhead.
- DOMAIN-SET/RULE-SET with invalid lines now cause entire set to fail (strong validation since recent versions).

### Port Hopping:
- Hysteria2 and TUIC protocols support port hopping to mitigate ISP UDP QoS throttling.

---
# ═══════════════════════════════════════════════════════════
# PART II: FROM SURGE MANUAL (manual.nssurge.com)
# Complete reference for all config syntax and parameters
# ═══════════════════════════════════════════════════════════

## 34. Complete Proxy Protocol Reference (from Manual)

Surge supports these proxy protocols:
- http, https (HTTP via TLS), socks5, socks5-tls
- ss (Shadowsocks), snell (v1-v5), vmess, trojan
- tuic (QUIC-based), hysteria2 (QUIC-based), anytls (newest)
- wireguard (L3 VPN as proxy), ssh (equiv to ssh -D)
- external (launch external proxy program, Mac only)
- direct (explicit direct with interface binding)

### Syntax examples:
[Proxy]
ProxyHTTP = http, 1.2.3.4, 443, username, password
ProxyHTTPS = https, 1.2.3.4, 443, username, password
ProxySOCKS5 = socks5, 1.2.3.4, 443, username, password
ProxySOCKS5TLS = socks5-tls, 1.2.3.4, 443, username, password
Proxy-SS = ss, 1.2.3.4, 8000, encrypt-method=chacha20-ietf-poly1305, password=abcd1234
Proxy-VMess = vmess, 1.2.3.4, 8000, username=0233d11c-15a4-47d3-ade3-48ffca0ce119
Proxy-Trojan = trojan, 192.168.20.6, 443, password=password1
Proxy-TUIC = tuic, 192.168.20.6, 443, token=pwd, alpn=h3
Proxy-Hysteria = hysteria2, 192.168.20.6, 443, password=pwd, download-bandwidth=100
Proxy-AnyTLS = anytls, 192.168.20.6, 443, password=pwd

### UDP relay support:
UDP supported natively: Snell v4/v5, WireGuard, Hysteria 2, TUIC, Trojan
UDP must be explicitly enabled: Shadowsocks (udp-relay=true), SOCKS5 (udp-relay=true)

### Proxy chain:
underlying-proxy = AnotherProxy  # Use a proxy to connect to another proxy

### Shadow TLS (obfuscation):
STLS-SNELL = snell, 1.2.3.4, 443, psk=pwd1, version=4, reuse=true, shadow-tls-password=pwd2, shadow-tls-version=3, shadow-tls-sni=www.example.com

### Port hopping (Hysteria2, TUIC):
Proxy-Hop = hysteria2, server.com, 443, password=pwd, port-hopping=1234;5000-6000, port-hopping-interval=30

## 35. SSH as Proxy Policy (from Manual)

[Proxy]
# Password auth:
proxy-ssh = ssh, 1.2.3.4, 22, username=root, password=pw, idle-timeout=180
# Public key auth:
proxy-ssh-key = ssh, 1.2.3.4, 22, username=root, private-key=key1
# Server fingerprint verification (anti-MITM):
proxy-ssh-fp = ssh, 1.2.3.4, 22, username=root, password=pw, server-fingerprint="ssh-ed25519 AAAAC3..."

[Keystore]
key1 = type=openssh-private-key, base64=[base64 of entire private key file]
# Note: base64-encode the ENTIRE key file, even though it's already base64 internally.
# Supports RSA/ECDSA/ED25519/DSA. Requires OpenSSH 7.3+ (curve25519-sha256 + aes128-gcm).

## 36. Common Proxy Policy Parameters (from Manual)

All proxy types accept these optional parameters:
- interface = en2              # Force specific network interface
- allow-other-interface = true # Fallback to default if interface unavailable
- interface-for-dns = true     # DNS queries also use specified interface
- underlying-proxy = ProxyB   # Proxy chain
- tfo = true                   # TCP Fast Open
- ip-version = dual/v4-only/v6-only/prefer-v4/prefer-v6
- ecn = true                   # Explicit Congestion Notification (improves high-loss networks)
- tos = 0xNN                   # Custom IP TOS value
- skip-cert-verify = true      # Skip TLS cert verification (INSECURE)
- sni = custom.example.com     # Custom SNI (or sni=off to disable)
- server-cert-fingerprint-sha256 = <hex>  # Certificate pinning
- always-use-connect = true    # Always use HTTP CONNECT even for plain HTTP
- block-quic = auto|on|off     # Per-proxy QUIC blocking
- test-url = http://example.com # Override global test URL
- test-timeout = 5             # Override global test timeout
- client-cert = cert1          # Client certificate for mutual TLS
- hybrid = true                # Dual Wi-Fi + cellular (fastest link wins)

### Direct policy with interface binding:
[Proxy]
Corp-VPN = direct, interface=utun0
WiFi-Only = direct, interface=en0, allow-other-interface=true

## 37. Complete Rule Type Reference (from Manual)

### Domain-based rules:
DOMAIN,www.example.com,PROXY                      # Exact match
DOMAIN-SUFFIX,example.com,PROXY                   # Suffix match (includes subdomains AND bare domain)
DOMAIN-KEYWORD,google,PROXY                       # Contains keyword
DOMAIN-WILDCARD,*cdn*.example.com,PROXY           # Wildcard (* and ?)
DOMAIN-SET,https://example.com/domains.txt,PROXY  # External domain list (optimized fast search for 1000+ entries)

### IP-based rules (trigger DNS unless no-resolve):
IP-CIDR,192.168.0.0/16,DIRECT,no-resolve         # IPv4 CIDR (single IP = /32 since Mac 6.0)
IP-CIDR6,2001:db8::/32,PROXY,no-resolve          # IPv6 CIDR (single IP = /128)
GEOIP,US,DIRECT,no-resolve                        # GeoIP country code
IP-ASN,13335,PROXY,no-resolve                     # Autonomous System Number

### HTTP rules (HTTP/HTTPS only, not raw TCP):
USER-AGENT,MyApp*,PROXY                           # Wildcard UA match
URL-REGEX,^https://api\.example\.com/v2,PROXY     # URL regex match

### Process rules (Mac only):
PROCESS-NAME,Telegram,PROXY                       # Match by process name

### Misc rules:
DEST-PORT,443,PROXY                               # Destination port
SRC-PORT,12345,DIRECT                             # Source port (rare)
SRC-IP,192.168.1.100,PROXY                        # Source IP (gateway mode)
IN-PORT,6152,PROXY                                # Incoming listen port (Mac multi-port)
PROTOCOL,UDP,REJECT                               # Protocol: HTTP,HTTPS,TCP,UDP,DOH,DOH3,DOQ,QUIC,STUN
HOSTNAME-TYPE,IPv4,DIRECT                         # Hostname form: IPv4,IPv6,DOMAIN,SIMPLE
CELLULAR-RADIO,LTE,PROXY                          # Cellular tech (iOS)
DEVICE-NAME,iPhone,PROXY                          # Device name (gateway/Ponte mode)
SRC-MAC,AA:BB:CC:DD:EE:FF,PROXY                  # MAC address (same-LAN only)
SCRIPT,ScriptName,PROXY                           # JS-based rule matching

### Logical rules (combine any rules):
AND,((PROTOCOL,UDP),(DEST-PORT,443)),REJECT-NO-DROP
OR,((DOMAIN-SUFFIX,a.com),(DOMAIN-SUFFIX,b.com)),PROXY
NOT,((DOMAIN-SUFFIX,cn)),PROXY

### Rule sets:
RULE-SET,https://example.com/rules.txt,PROXY,no-resolve,extended-matching
RULE-SET,SYSTEM,DIRECT     # Built-in Apple system services
RULE-SET,LAN,DIRECT        # Built-in LAN (triggers DNS!)

### Inline rule sets (embedded in profile):
[Ruleset Streaming]
DOMAIN-SUFFIX,netflix.com
DOMAIN-SUFFIX,netflix.net
[Rule]
RULE-SET,Streaming,StreamingProxy

### extended-matching parameter:
# When enabled, domain rules also match SNI and HTTP Host header (not just DNS hostname)
DOMAIN-SUFFIX,example.com,PROXY,extended-matching

### FINAL rule:
FINAL,PROXY,dns-failed   # dns-failed: if DNS fails, still forward to proxy instead of erroring

## 38. REJECT Policy Deep Dive (from Manual)

### Pre-matching phase (at DNS query time):
- REJECT: returns "No Record" DNS response. If frequency limit triggered (10x in 30s), drops silently.
- REJECT-DROP: DNS query discarded without response.
- REJECT-NO-DROP: returns special IP 198.18.0.244; TCP connections get RST response.

### UDP packets (no pre-matching, matched via main rules):
- REJECT: returns ICMP Administratively Prohibited. Frequency limit → silent drop.
- REJECT-DROP: silent drop.

### Pre-matching eligible rule types:
DOMAIN, DOMAIN-SUFFIX, DOMAIN-KEYWORD, DOMAIN-SET, DOMAIN-WILDCARD, IP-CIDR, IP-CIDR6, GEOIP, IP-ASN, RULE-SET (if contents qualify)

### Auto-upgrade:
10 REJECT hits from same hostname within 30 seconds → auto-upgrades to REJECT-DROP to prevent request storms.

## 39. DNS Deep Dive (from Manual)

### DNS behavior:
- Surge queries ALL DNS servers simultaneously (like dnsmasq --all-servers), uses first response.
- If no response in 2 seconds, retries all servers. After 4 retries, reports DNS error.
- When IPv6 enabled, sends both A and AAAA queries.

### [Host] section (Local DNS Mapping):
[Host]
# Static IP mapping
example.com = 1.2.3.4
# Wildcard
*.dev.example.com = 10.0.0.1
# Alias (CNAME-like)
foo.example.com = bar.example.com
# Custom DNS server for specific domain
example.com = server:8.8.8.8
# Encrypted DNS for specific domain
example.com = server:https://doh.example.com/dns-query
# System resolver (for domains Surge's DNS client can't handle)
special.local = server:system
# server:syslib - stays inside Surge but uses macOS system DNS servers (useful in enhanced mode)
corp.internal = server:syslib
# DOMAIN-SET / RULE-SET binding (share maintained lists with DNS mapping):
DOMAIN-SET:https://example.com/cn-domains.txt = server:119.29.29.29
RULE-SET:https://example.com/rules.txt = 10.0.0.10

### use-local-host-item-for-proxy:
# By default proxy requests use domain (remote DNS). With this enabled, if local [Host] mapping exists,
# Surge sends IP to proxy instead of domain.

### Encrypted DNS:
[General]
encrypted-dns-server = https://dns.nextdns.io/xxx, quic://dns.nextdns.io
# When encrypted DNS configured, traditional DNS only used for:
# 1. Testing connectivity
# 2. Resolving the encrypted DNS server's own domain

### Route DoH through proxy:
encrypted-dns-follow-outbound-mode = true
# Then add rule: PROTOCOL,DOH,ProxyGroup (or DOH3, DOQ)

## 40. Scripting Complete Reference (from Manual)

### Script types: http-request, http-response, cron, event, dns, rule, generic

### Common parameters:
script-path = scripts/example.js   # Local, relative, absolute, or URL
script-update-interval = 86400     # URL refresh interval
max-size = 131072                  # Max body size (128KB default). Larger → Surge may crash on iOS
binary-body-mode = true            # Pass body as Uint8Array instead of String
requires-body = true/false         # Whether to load request/response body
engine = auto|jsc|webview          # jsc: fast, low memory. webview: complex scripts

### Available APIs in scripts:
$httpClient.get/post/put/delete/head/options/patch(options, callback)
  # options: {url, headers, body, timeout, insecure, auto-cookie, auto-redirect, binary-mode}
$persistentStore.read(key) / .write(data, key)   # Persistent storage across sessions
  # Mac storage: ~/Library/Application Support/com.nssurge.surge-mac/SGJSVMPersistentStore/
$httpAPI(method, path, body, callback)            # Call any Surge HTTP API endpoint (no auth needed)
$notification.post(title, subtitle, body)         # Push notification
$network                                          # Current network info (dns, wifi, v4/v6)
$script.name / $script.startTime
$environment                                      # System info
$done(result)                                     # MUST call to complete. Async OK.

### http-request script $done() object:
{ url, headers, body, response: {status, headers, body} }
# If response object returned → Surge returns it directly without network request (local mock)

### http-response script $done() object:
{ headers, body, status }

### event script types:
- network-changed: triggered on network change. Access $network for new DNS/WiFi info.
- notification: triggered when Surge shows a notification. Access $event.data.

### dns script:
$done({addresses: ["1.2.3.4"], ttl: 600})         # Return resolved IPs
$done({server: "8.8.8.8"})                         # Delegate to upstream server
$done({servers: ["8.8.8.8", "1.1.1.1"]})           # Delegate to multiple servers
$done({})                                           # Fallback to standard DNS

### rule script:
# Return policy name string: $done("ProxyA") or $done("DIRECT") or $done("REJECT")

## 41. Module Advanced Features (from Manual)

### Module metadata:
#!name=Module Name
#!desc=Description
#!system=mac                    # Platform: mac, ios, tvos
#!require-version=CORE_VERSION>=20 && (SYSTEM='iOS' || SYSTEM='tvOS')  # Logical expressions

### Module arguments (user-customizable parameters):
#!arguments=hostname:example.com&enable_mitm:true&port:443
# Arguments become %hostname%, %enable_mitm%, %port% — text-replaced before applying
[Rule]
DOMAIN,%hostname%,PROXY

### Module section operations:
[General]
always-real-ip = %APPEND% *.stun.example.com     # APPEND to existing value
[Rule]
# Module rules inserted at TOP of rule list (higher priority)
# Module rules can only use built-in policies: DIRECT, REJECT, REJECT-TINYGIF
[MITM]
hostname = %APPEND% api.example.com              # Append to existing MITM hostnames
# Use - prefix to exclude: hostname = -*.apple.com, *

### Module can patch:
- [General], [Rule], [Host], [URL Rewrite], [Header Rewrite], [Body Rewrite]
- [Map Local], [MITM], [Script], [WireGuard *], [Ruleset *] (inline rulesets)

## 42. Port Forwarding (from Manual)

Surge can listen on local ports and forward TCP to specific hosts. Works even without system proxy/enhanced mode.

[Port Forwarding]
# local-port:remote-host:remote-port
localhost:8022 = 192.168.1.100:22
# With policy (optional):
localhost:3306 = db.internal:3306, policy=Corp-VPN

## 43. Misc Options Reference (from Manual)

### force-http-engine-hosts:
# Force Surge to treat raw TCP as HTTP for specific hosts (enables rewrite/script/capture)
force-http-engine-hosts = example.com, *.api.example.com:8080
# Host List rules: -prefix excludes, :port specifies port, :0 = all ports, <ip-address> = IP literal

### always-raw-tcp-hosts:
# Opposite of force-http-engine-hosts. Disable protocol sniffing for specific hosts.
always-raw-tcp-hosts = problematic.app.com
### always-raw-tcp-keywords:
# Substring match version (no wildcard needed)
always-raw-tcp-keywords = someapp

### Network interface options:
allow-wifi-access = true          # Allow LAN access to Surge proxy
wifi-access-http-port = 6152      # HTTP proxy port (or http-listen = 0.0.0.0:6152)
wifi-access-socks5-port = 6153    # SOCKS5 port (or socks5-listen = 0.0.0.0:6153)
wifi-access-auth = user:pass      # Proxy authentication

### iOS-specific:
wifi-assist = true                # Use cellular when WiFi poor
hide-vpn-icon = true              # Hide VPN icon in status bar
all-hybrid = true                 # Always dual WiFi+cellular (use fastest)
include-all-networks = true       # Capture ALL traffic (prevents VIF bypass)
allow-hotspot-access = true       # Allow proxy when Personal Hotspot on

### ICMP:
icmp-forwarding = true            # Forward ICMP packets directly (default: true). Disable to prevent IP leak.

### GeoIP:
geoip-maxmind-url = https://...   # Custom GeoIP database URL
disable-geoip-db-auto-update = false

### HTTP API TLS:
http-api-tls = true               # HTTPS for HTTP API (uses MITM CA cert)

### Duplicate headers:
use-multi-value-header = true     # Expose full header arrays (not just last value)

## 44. Surge Mac CLI (from Manual)

Command-line interface for automation:
surge-cli reload          # Reload profile
surge-cli stop            # Stop Surge engine
surge-cli outbound        # Get/set outbound mode
surge-cli select-group    # Switch policy group selection
surge-cli dns-flush       # Flush DNS cache

## 45. External Proxy Program (Mac only, from Manual)

[Proxy]
external = external, exec="/usr/bin/ssh", args="server.com", args="-D", args="127.0.0.1:1080", local-port=1080, addresses=server-ip
# Surge starts the process, forwards to SOCKS5 127.0.0.1:1080
# Auto-restarts if process dies. Auto-excludes addresses from VIF routes.
# External process requests always use DIRECT (prevents loops).
# Retry: 6 attempts with 500ms delay if connection refused (startup time).
# iOS: external policy treated as REJECT (not supported).

## 46. Information Panel (from Manual)

Display custom info in Surge UI via scripts:
[Panel]
PanelName = script-name=panel-script, update-interval=300
# Script must return: $done({title: "Title", content: "Status text", icon: "bolt.fill"})

## 47. URL Scheme (from Manual)

surge:///start                    # Start Surge
surge:///stop                     # Stop Surge
surge:///toggle                   # Toggle on/off
surge:///install-profile?url=URL  # Install remote profile
surge:///install-module?url=URL   # Install module
surge:///run-cron?name=script1    # Trigger cron script

## 48. Managed Profile (from Manual)

For enterprise/service provider deployment:
#!MANAGED-CONFIG https://example.com/profile.conf interval=86400 strict=true
# interval: auto-update interval in seconds
# strict=true: always use remote version (no local edits)

## 49. Key Architectural Insights (from Understanding Surge guide)

### Three takeover methods:
1. System HTTP Proxy (127.0.0.1:6152) — only works for apps that respect proxy settings
2. Virtual NIC (VIF/Enhanced Mode) — captures ALL traffic at network layer
3. Gateway Mode — captures traffic from other LAN devices

### Connection types Surge handles:
- Type 1: Plain HTTP → full modification/rewrite/script/capture
- Type 2: HTTPS with MITM → decrypted to HTTP, full capabilities
- Type 3: Raw TCP/TLS without MITM → forwarding only (no inspection)
- UDP sessions: grouped by destination address+port

### Fake IP mechanism:
- Surge VIF returns 198.18.x.x for all DNS queries (TTL=1s)
- When TCP/UDP packet sent to fake IP, translated back to original domain
- always-real-ip exempts specific domains from fake IP
- force-remote-dns removed (all domains get fake IP now)

### SNI (Server Name Indication):
- TLS handshake includes hostname in plaintext SNI field
- Surge uses SNI for domain-based rule matching on HTTPS traffic
- MITM intercepts by generating matching cert using CA

### Protocol sniffing:
- Surge doesn't auto-detect HTTP in raw TCP (could break non-HTTP)
- Use force-http-engine-hosts to explicitly enable for specific hosts

### REJECT frequency limiting:
- 10 hits from same hostname in 30 seconds → auto-upgrade to REJECT-DROP
- Prevents request storm from aggressive retry logic

---

## 50. Quick Reference Cheat Sheet

### Most important [General] parameters:
loglevel, skip-proxy, tun-excluded-routes, tun-included-routes, exclude-simple-hostnames
ipv6, hijack-dns, always-real-ip, encrypted-dns-server, encrypted-dns-follow-outbound-mode
dns-server, fallback-dns-server, block-quic, udp-policy-not-supported-behaviour
http-api, http-api-web-dashboard, external-controller-access
force-http-engine-hosts, always-raw-tcp-hosts, allow-dns-svcb
show-error-page, proxy-restricted-to-lan, gateway-restricted-to-lan, icmp-forwarding
wifi-assist, all-hybrid, include-all-networks, compatibility-mode

### Config file sections:
[General] [Proxy] [Proxy Group] [Rule] [Host] [DNS]
[URL Rewrite] [Header Rewrite] [Body Rewrite] [Map Local]
[MITM] [Script] [WireGuard *] [Keystore] [Ruleset *] [Panel]
[Port Forwarding]

### Built-in policies:
DIRECT, REJECT, REJECT-DROP, REJECT-NO-DROP, REJECT-TINYGIF
DEVICE:ponte-name (Ponte access)

### Built-in rulesets:
RULE-SET,SYSTEM,DIRECT  (Apple system services, auto-updated)
RULE-SET,LAN,DIRECT     (Local network, triggers DNS)

---

# Community Official Knowledge (community.nssurge.com/t/official)

## §51 Encrypted Client Hello (ECH)

ECH encrypts TLS ClientHello SNI field using public key from HTTPS RR (SVCB DNS records).
Prerequisites: Must use DoH (encrypted DNS) — if DNS is plaintext, ECH is meaningless since SNI already exposed via DNS.

### ECH vs Surge conflict:
- Surge NEEDS to read SNI for rule matching. When ECH+DoH fully works, Surge can only see destination IP, not hostname
- All domain rules become ineffective under ECH

### If user insists on ECH (NOT recommended):
1. Disable proxy mode, use VIF-only (enhanced mode). Browsers don't support DoH through proxy mode
2. Enable browser DoH (Chrome/Firefox → Cloudflare DNS)
3. Verify: https://crypto.cloudflare.com/cdn-cgi/trace → sni=encrypted means ECH working
4. Dashboard will show all requests using IP addresses only — domain rules won't match

### Key insight: Surge auto-blocks SVCB DNS requests (allow-dns-svcb=false by default) which prevents ECH from working. This is intentional — ECH would break Surge's core functionality.

## §52 Adaptive TLS Fingerprint

TLS ClientHello parameters form a unique fingerprint (JA3) that can identify client software.
Surge implements Adaptive TLS Fingerprint by copying system TLS fingerprint from gateway.icloud.com HTTPS requests.

### How it works:
- Surge intercepts system's TLS handshake with gateway.icloud.com
- Copies all ClientHello parameters (cipher suites, extensions, curves, etc.)
- Replays identical parameters when connecting to Shadow TLS v3 servers
- Result: JA3 fingerprint matches native iOS/macOS exactly

### Limitations:
- Currently ONLY available with Shadow TLS v3 (technical limitation)
- Cannot be applied to other TLS protocols (Trojan, HTTPS proxy) independently
- Shadow TLS server-side --tls target doesn't have to be gateway.icloud.com; any major TLS 1.3 site works
- Mac 5+ only feature (not available on Mac 4.x)

## §53 Compatibility Issues

### HomeKit Secure Video:
- HKSV uses reverse connection: client reports its IP to Hub, Hub connects back
- With VIF enabled, client reports 198.18.0.1 (virtual IP) → Hub can't connect back
- Mac fix: disable enhanced mode
- iOS fix: compatibility-mode=5 (uses multiple small routes, preserves real interface IP)

### Screen Sharing High Performance Mode:
- High-perf mode: server sends UDP to client's separate port
- Surge Ponte doesn't map this extra port → connection fails
- Must use real L3 VPN for high-performance screen sharing

### AdGuard:
- TCP FastOpen (TFO) conflicts with AdGuard — disable TFO when co-existing
- Different AdGuard modes have different compatibility issues

## §54 DNS Leak Deep Dive

Two distinct meanings of "DNS leak":

### Type 1: Plaintext DNS interception
- ISP/firewall/public WiFi can see DNS queries in plaintext
- Fix: Use DoH/DoQ encrypted DNS, or use full proxy mode (no local DNS at all)
- Practical risk is low: modern apps generate massive DNS queries, hard to extract useful info unless visiting obscure domains

### Type 2: DNS ECS-based IP detection
- Target website constructs random subdomain → DNS query includes EDNS Client Subnet → reveals client's IP range
- Only reveals approximate region, NOT exact IP
- Fixes:
  - Use full proxy mode (no local DNS resolution)
  - Use DNS without ECS support (e.g., CloudFlare 1.1.1.1) — but may cause CDN mis-routing
  - Custom DNS ECS field with fake IP — may break CDN allocation

## §55 IP Address Leak Deep Dive

### Scope: Specifically about target websites detecting visitor's real IP through proxy.
- ISP and VPN provider ALWAYS know real IP — this is NOT a "leak"
- All leak scenarios share one cause: request goes DIRECT instead of through proxy

### WebRTC Leak:
- System proxy mode: browsers don't support UDP through SOCKS5 → WebRTC sends UDP directly → real IP exposed
- Surge with Enhanced Mode (VIF): UDP also goes through proxy → no leak (unless DIRECT policy matched)
- udp-policy-not-supported-behaviour: default REJECT from Mac 6.0 (prevents silent UDP fallback to DIRECT)

### DNS Leak (ECS):
- Local DNS queries include EDNS Client Subnet → approximate IP range exposed
- Only reveals ISP/region, not exact IP — limited practical value
- Fix: Use DNS without ECS (1.1.1.1) but may hurt CDN performance

### Split Routing Leak:
- Any DIRECT-matched request reveals real IP to the requesting domain
- Website a.com can load b.com resource → if b.com matches DIRECT → a.com knows real IP
- Only fix: avoid DIRECT for ALL traffic (full proxy mode + enhanced mode for UDP)

### include-all-networks (iOS only):
- Forces ALL traffic through Surge VPN tunnel, preventing apps from binding to physical interface
- Has significant side effects — only use with clear need

## §56 Congestion Control & ECN

### How bandwidth is negotiated:
- No elegant protocol — relies on packet loss. Send faster → packets drop → slow down
- Congestion control algorithm determines: starting speed, ramp-up rate, loss response
- BBR (Google): designed for modern internet, performs well under packet loss
- Traditional algorithms (Reno, Cubic): designed for LAN, perform poorly with non-congestion packet loss

### TCP Slow Start:
- Every new connection starts slow and ramps up — BBR reaches peak faster
- In lossy networks (international links, wireless): theoretical 100Mbps link with 1% loss → traditional algorithms may only achieve 50Mbps; BBR can approach 99Mbps

### ECN (Explicit Congestion Notification):
- Network nodes mark packets with CE flag when approaching congestion (instead of dropping)
- Requires ALL devices in path to support ECN
- Worst case: incompatible devices drop ECN-marked packets entirely → connection fails
- Surge supports QUIC ECN for TUIC proxy and Surge Ponte (manual enable)
- TCP ECN: macOS/iOS kernel supports it but Apple's implementation currently won't send ECN-marked TCP packets

### Hysteria protocol's approach:
- User manually sets bandwidth → sends at fixed rate regardless of loss
- Problem: if actual bandwidth < configured (other devices using bandwidth, weak WiFi) → 50% packets wasted, other devices starved
- Hysteria with BBR mode: standard QUIC behavior, ECN compatible
- Recommendation: configure conservatively, only use on stable bandwidth connections

## §57 Network Experience Optimizations (even with DIRECT)

### 1. Concurrent DNS Queries:
- System default: query DNS servers sequentially (wait for timeout before trying next)
- Surge: queries ALL configured DNS servers simultaneously, uses first response
- Prevents delays when primary DNS is slow/down

### 2. Concurrent TCP Connections (Happy Eyeballs-like):
- When domain resolves to multiple IPs, system uses first IP returned
- If first IP is slow/dead: 10-180 second timeout before trying next
- Surge: initiates TCP handshake to ALL IPs simultaneously, uses fastest responder
- Only tests on first visit; caches result for subsequent connections
- Automatically retries others if cached IP becomes unavailable

### 3. Proxy DNS Queries:
- Large services use GeoDNS to return nearest server IPs
- If DNS resolved locally but traffic goes through proxy → gets wrong region's IPs → slow
- Surge guarantees: ALL proxy-policy requests use remote DNS resolution

### 4. Optimistic DNS:
- Problem: Facebook TTL=30s → DNS re-query every 30 seconds → latency spikes
- Apple WWDC 2018 solution: use expired cache IP immediately, update DNS in background
- If cached IP fails → retry with fresh DNS result
- Eliminates re-query latency without compromising DNS record updates

## §58 High Request Count Crash (P2P/BT Gateway)

### Root cause:
- macOS has fixed network stack memory buffer (~256MB)
- P2P creates massive concurrent connections → exhausts kernel buffer
- Kernel kills highest-memory network process (Surge) → all sockets closed → unrecoverable

### Fix - increase kernel buffer to 512MB + enable server performance mode:
sudo nvram boot-args="serverperfmode=1 ncl=262144"

### For Apple Silicon (M1/M2/M3):
1. Boot into Recovery Mode
2. Set security to Medium, disable SIP (csrutil disable)
3. Reboot to macOS, then run the nvram command

### Alternative: Use Traditional VIF
- New high-performance VIF: memory in kernel space (aggravates the problem)
- Traditional VIF: memory in Surge process space (doesn't exhaust kernel buffer)
- Enable in Advanced Settings → Traditional VIF

### Surge Mac 5.2.1+: Auto-detection
- Detects kernel memory exhaustion → auto-restart → enters Safe Mode (traditional VIF)

### Best practice for gateway mode:
- Route P2P processes DIRECT: PROCESS-NAME,Transmission,DIRECT
- Or: isolate BT client on separate IP (via macvlan) that doesn't route through Surge

## §59 Proxy Protocol Philosophy

### Why Surge is conservative with new protocols:
1. **Latency**: SS/Snell already achieve 0-RTT — cannot be improved
2. **CPU/encryption**: AES with hardware acceleration → 0.1ms per operation → reaches WiFi physical throughput limit on iOS
3. **Security**: All properly implemented modern encryption is unbreakable; TLS provides forward secrecy; older protocols = more battle-tested
4. **Multiplex**: HTTP/2 already provides multiplexing; proxy-layer multiplex adds complexity (single-TCP throttling, error handling) with marginal benefit
5. **gRPC**: Developer convenience framework, doesn't improve proxy protocol itself

### Protocol traffic signature categories:
- **Explicit signature**: OpenVPN, SSH — easily detected and blocked
- **No signature**: SS, VMess — completely random encrypted data can itself be a signature (detected as "non-standard traffic")
- **Real TLS**: Trojan — matches standard TLS (with Adaptive TLS Fingerprint); certificate may be fingerprinted; double-TLS traffic pattern detectable but impractical to block
- **TLS mimicry**: ShadowTLS — even server certificate is disguised; implementation flaws possible; needs time to prove reliability
- **Random signature**: Snell — generates random per-session fingerprint using PSK hash; each user/restart produces different pattern; non-reversible

### Key principle: If your current protocol isn't being blocked, no need to chase new ones.
Blocking considers multiple factors: protocol signature, datacenter IP ranges, traffic volume, timing patterns.

## §60 HTTP API Complete Reference

### Configuration:
[General]
http-api = examplekey@0.0.0.0:6166
http-api-tls = true  # Uses MITM certificate for HTTPS
http-api-web-dashboard = true  # Enable built-in web UI

### Authentication: X-Key header required for all requests
### Methods: GET (URL params) + POST (JSON body). Always returns JSON.

### Endpoints:
# Feature toggles (GET to read, POST {"enabled":true/false} to set)
/v1/features/mitm
/v1/features/capture
/v1/features/rewrite
/v1/features/scripting
/v1/features/system_proxy      # Mac only
/v1/features/enhanced_mode     # Mac only

# Outbound mode
GET/POST /v1/outbound          # {"mode":"rule|direct|proxy"}
GET/POST /v1/outbound/global   # {"policy":"ProxyA"}

# Policies
GET /v1/policies               # List all
GET /v1/policies/detail?policy_name=X
POST /v1/policies/test         # {"policy_names":["A","B"],"url":"http://bing.com"}

# Policy groups
GET /v1/policy_groups           # List all groups + options
GET /v1/policy_groups/test_results
GET /v1/policy_groups/select?group_name=X  # → {"policy":"A"}
POST /v1/policy_groups/select   # {"group_name":"X","policy":"A"}
POST /v1/policy_groups/test     # {"group_name":"X"} → results with tcp/rtt/receive/available ms

# Requests
GET /v1/requests/recent
GET /v1/requests/active
POST /v1/requests/kill          # {"id":100}

# Profiles
GET /v1/profiles/current?sensitive=0  # sensitive=0 masks passwords
POST /v1/profiles/reload
POST /v1/profiles/switch        # {"name":"Profile2"} Mac only

# DNS
POST /v1/dns/flush
GET /v1/dns                     # Current cache
POST /v1/test/dns_delay

# Modules
GET /v1/modules                 # {"enabled":[...],"available":[...]}
POST /v1/modules                # {"ModuleName":true/false}

# Scripting
GET /v1/scripting               # List scripts
POST /v1/scripting/evaluate     # {"script_text":"...","mock_type":"cron","timeout":5}
POST /v1/scripting/cron/evaluate # {"script_name":"script1"}

# Misc
POST /v1/stop                   # Shutdown engine
GET /v1/events                  # Event center
GET /v1/rules                   # Rule list
GET /v1/traffic                 # Traffic stats
POST /v1/log/level              # {"level":"verbose"}

### Script HTTP API ($httpAPI):
$httpAPI(method, path, body, callback)
# No need for X-Key header — internal call

## §61 iOS VIF-Priority Mode (compatibility-mode)

### Background:
- Originally: Proxy mode priority (HTTP proxy), VIF as supplement
- Proxy mode advantages: loopback socket (efficient), definitive HTTP detection, 0-RTT early data
- Problem: increasing apps detect system proxy and refuse to work

### New default (iOS TF build 2890+): VIF-priority mode
compatibility-mode = 0  # Auto (now defaults to VIF-priority)
compatibility-mode = 1  # Legacy proxy-priority mode
compatibility-mode = 3  # Explicit VIF-priority (same as 0 in new versions)
compatibility-mode = 5  # Small-route VIF (preserves real interface IP, fixes HomeKit)

### VIF engine optimizations for new mode:
1. Port 80/443: waits for first client packet → identifies HTTP/TLS → uses HTTP engine; if not HTTP or 300ms timeout → falls back to raw TCP
2. Auto-extracts SNI from TLS ClientHello for rule matching and MITM hostname matching
3. MITM no longer needs tcp-connection parameter — defaults to TCP
4. Other ports: still need force-http-engine-hosts (can't wait for client data on SSH/IMAP/FTP — server speaks first)
5. 0-RTT early data now works in VIF mode too (previously proxy-mode only)

## §62 Gateway Mode Best Practices

### Official recommendations:
1. Do NOT run Surge iOS/Mac on client device when already behind Surge gateway — choose one
2. Do NOT route P2P/BT through gateway (see §58 for crash details)
3. If network has IPv6: disable RA DNS advertisement, or disable IPv6 entirely (IPv6 DNS bypasses Surge hijack)
4. Jumbo MTU: supported but compatibility issues frequent, marginal benefit — not recommended

### MITM for downstream devices:
- Requires: tcp-connection=true AND client-source-address=* in [MITM]
- Downstream devices must install and trust the SAME Surge CA certificate
- For HTTP (port 80): configure force-http-engine-hosts
- For HTTPS: MITM hostname matching works automatically with SNI extraction

### P2P isolation strategies:
- PROCESS-NAME rules: PROCESS-NAME,Transmission,DIRECT (only works on gateway Mac itself)
- Separate IP via macvlan: isolate BT client on different IP that doesn't use Surge as gateway
- Separate device: dedicate a machine for BT that doesn't route through Surge

## §63 Minimum Recommended Config (China Users)

[General]
skip-proxy = 192.168.0.0/24, 10.0.0.0/8, 172.16.0.0/12, 127.0.0.1, localhost, *.local
exclude-simple-hostnames = true
internet-test-url = http://taobao.com/
proxy-test-url = http://www.apple.com/
test-timeout = 2
dns-server = 223.5.5.5, 114.114.114.114
# DoH usually unnecessary unless severe DNS pollution
wifi-assist = true
ipv6 = false

[Rule]
RULE-SET,https://github.com/Blankwonder/surge-list/raw/master/blocked.list,Proxy
RULE-SET,https://github.com/Blankwonder/surge-list/raw/master/cn.list,DIRECT
DOMAIN,apps.apple.com,Proxy
DOMAIN-SUFFIX,ls.apple.com,DIRECT
DOMAIN-SUFFIX,store.apple.com,DIRECT
RULE-SET,SYSTEM,Proxy
RULE-SET,https://github.com/Blankwonder/surge-list/raw/master/apple.list,Proxy
RULE-SET,LAN,DIRECT
GEOIP,CN,DIRECT
FINAL,Proxy,dns-failed

### Key design choices:
- Whitelist mode (FINAL,Proxy): unmatched traffic goes proxy — safer, catches unlisted blocked sites
- Apple services via proxy: improves reliability for iCloud/App Store in China
- ls.apple.com DIRECT: Apple Maps needs China servers
- store.apple.com DIRECT: Apple Store Online
- exclude-simple-hostnames: skips proxy for enterprise intranet simple hostnames (e.g., https://unifi/)
