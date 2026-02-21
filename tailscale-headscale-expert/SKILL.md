---
name: tailscale-headscale-expert
description: "Expert-level Tailscale and Headscale knowledge base. Covers NAT traversal (STUN/ICE/DERP), WireGuard mesh networking, Headscale self-hosted control server deployment and configuration (v0.28+), ACL policies (groups/tags/autogroups), DERP relay setup and optimization, MagicDNS, exit nodes, subnet routing, pre-auth keys, OIDC integration, Headplane web UI, client management (Windows/macOS/Linux), netcheck diagnostics, and troubleshooting connectivity issues. Use this skill whenever the user mentions Tailscale, Headscale, Headplane, DERP, tailnet, mesh VPN, NAT traversal, exit node, subnet routing, ACL policy, MagicDNS, or any VPN mesh networking topic. Also trigger for your-company enterprise deployment, client onboarding scripts, and multi-VPS Headscale architectures."
---

# Tailscale & Headscale Expert Knowledge Base

Expert-level reference for Tailscale mesh VPN and Headscale self-hosted control server. Output exact, copy-paste ready configs. User is an advanced network engineer running enterprise VPN services — skip basics, go straight to solutions with inline comments.

---

## 1. Architecture Overview

### How Tailscale Works
- **Control plane** (coordination server): Exchanges node keys, distributes DERP map, ACL policies, DNS config
- **Data plane** (WireGuard): Direct peer-to-peer encrypted tunnels between devices
- **DERP relays**: Fallback when direct P2P fails; also used for initial connection negotiation
- **All traffic is end-to-end encrypted** — DERP servers relay ciphertext, can never decrypt

### Connection Flow
```
1. Device → Control server: Register, get peer list + DERP map
2. Device → DERP relay: Exchange disco messages with peer
3. STUN probes: Discover external IP:port mappings
4. NAT traversal: Simultaneous UDP hole punching
5. Success → Direct WireGuard P2P tunnel
   Failure → DERP relay (encrypted, TCP-based, higher latency)
```

### Headscale vs Tailscale SaaS
| Feature | Tailscale SaaS | Headscale |
|---------|---------------|-----------|
| Control server | Tailscale-hosted | Self-hosted |
| ACL policies | Web UI + HuJSON | HuJSON file or database |
| DERP | Global network | Self-hosted + optional Tailscale DERP |
| Auth | SSO/OIDC built-in | OIDC optional, pre-auth keys |
| MagicDNS | ✅ | ✅ (v0.22+) |
| Price | Free tier / paid plans | Free (open source) |

---

## 2. NAT Traversal Deep Dive

### NAT Types (from easiest to hardest)
1. **Full Cone (EIM/EIF)**: Any external host can send to mapped port → trivial traversal
2. **Address-Restricted Cone**: Only hosts that device has sent to can reply → needs coordination
3. **Port-Restricted Cone**: Only specific ip:port pairs → needs simultaneous open
4. **Symmetric (Hard NAT)**: Different external port per destination → nearly impossible to traverse P2P

### STUN Protocol
```
Client (192.168.1.5:12345) → NAT → (2.2.2.2:54321) → STUN Server (5.5.5.5:3478)
STUN Server responds: "I see you as 2.2.2.2:54321"
Client now knows its external endpoint
```
- Tailscale runs STUN on every DERP server (UDP 3478)
- `tailscale netcheck` reports NAT type and DERP latency

### Why Direct Connection Fails
- **Both peers behind symmetric/hard NAT**: Random port per destination, can't predict
- **UDP blocked**: Corporate firewalls, some ISPs
- **Double NAT**: ISP CGNAT + home router NAT = compounded difficulty
- **Firewall blocking unsolicited UDP**: Windows Defender, corporate IPS

### DERP (Designated Encrypted Relay for Packets)
- Runs over HTTPS (TCP 443) — works on almost any network
- Relays WireGuard-encrypted packets (zero visibility into content)
- Each client picks a "home DERP" based on lowest latency
- All initial connections go through DERP, then upgrade to P2P if possible
- Limitations: TCP-based → higher latency, rate-limited, shared resource

### Diagnosing NAT Issues
```bash
# Full network diagnostics
tailscale netcheck

# Key output to look for:
# UDP: true/false (is UDP working at all?)
# MappingVariesByDestAddr: true = HARD NAT (symmetric)
# HairPinning: true/false
# DERP latency to each region
# Nearest DERP: region ID

# Check specific peer connection
tailscale status
# "direct" = P2P connection established
# "relay" = going through DERP
# "idle" = no recent traffic

# Detailed peer info
tailscale ping <peer-ip>
# Shows whether direct or relayed, latency

# Debug connection to specific peer
tailscale ping --verbose <peer-ip>
```

---

## 3. Headscale Server Configuration (v0.28+)

### Installation (Debian/Ubuntu)
```bash
VERSION=$(curl -s "https://api.github.com/repos/juanfont/headscale/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/' | sed 's/v//')
wget -O headscale.deb "https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"
dpkg -i headscale.deb
```

### Core config.yaml
```yaml
# /etc/headscale/config.yaml
server_url: https://your-domain.example.com  # Must be reachable by clients
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

database:
  type: sqlite
  path: /var/lib/headscale/db.sqlite

# IP allocation ranges (DO NOT change after deployment)
prefixes:
  v4: 100.64.0.0/10    # Tailscale CGNAT range
  v6: fd7a:115c:a1e0::/48

# Node key expiry (0 = never)
# default_host_info_change_expiry: 0

# DNS Configuration
dns:
  magic_dns: true
  base_domain: example.com   # MUST differ from server_url domain
  nameservers:
    global:
      - 1.1.1.1              # Clients use these for non-MagicDNS queries
      - 8.8.8.8
    # Split DNS: route specific domains to specific nameservers
    # restricted:
    #   internal.company.com:
    #     - 10.0.0.53
  search_domains: []
  override_local_dns: true    # If true, global nameservers must have at least one entry
  extra_records: []
  # Dynamic records from file (alternative to extra_records):
  # extra_records_path: /var/lib/headscale/extra-records.json

# ACL Policy
policy:
  mode: file                  # "file" or "database"
  path: /etc/headscale/acls.json

# OIDC (optional)
# oidc:
#   issuer: https://accounts.google.com
#   client_id: YOUR_CLIENT_ID
#   client_secret_path: /etc/headscale/oidc_secret
#   expiry: 180d
#   allowed_domains:
#     - yourdomain.com

# DERP
derp:
  server:
    enabled: false              # Set true to use embedded DERP
    region_id: 999
    region_code: headscale
    region_name: Headscale Embedded
    stun_listen_addr: 0.0.0.0:3478
  urls:
    - https://controlplane.tailscale.com/derpmap/default  # Include Tailscale's DERP
  paths:
    - /etc/headscale/derp.yaml  # Custom DERP servers
  auto_update_enabled: true
  update_frequency: 24h
```

### v0.28 Breaking Changes
- **Tags as identity**: Tags and user ownership are mutually exclusive
- **Wildcard `*` in ACL**: Now resolves to CGNAT range (100.64.0.0/10) not all IPs (0.0.0.0/0)
- **Minimum client version**: v1.74.0
- **Commands changed**: `headscale nodes routes list/enable` replaces `headscale routes`

---

## 4. ACL Policies (Access Control Lists)

### Concept
- Default: **DENY ALL** — nothing can communicate unless explicitly allowed
- Rules are **stateful** — if A→B is allowed, return traffic B→A is implicit
- No "deny" rules — only "accept" rules that open specific paths
- ACLs override user boundaries — all machines can communicate if ACL permits

### Basic Structure (HuJSON)
```json
{
  // Groups: organize users
  "groups": {
    "group:admin": ["admin"],
    "group:clients": ["clients"]
  },

  // Tag owners: who can assign tags
  "tagOwners": {
    "tag:server": ["group:admin"],
    "tag:exit-node": ["group:admin"]
  },

  // ACL rules
  "acls": [
    // Admin can access everything
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    // Clients can only use exit nodes (internet access)
    {
      "action": "accept",
      "src": ["group:clients"],
      "dst": ["autogroup:internet:*"]
    },
    // Clients can ping each other (optional)
    {
      "action": "accept",
      "src": ["group:clients"],
      "dst": ["group:clients:*"]
    }
  ]
}
```

### Autogroups (Headscale supported)
| Autogroup | Meaning | Usage |
|-----------|---------|-------|
| `autogroup:internet` | Internet via exit nodes | dst only |
| `autogroup:member` | All personal (untagged) devices | src or dst |
| `autogroup:tagged` | All tagged devices | src or dst |
| `autogroup:self` | Same user's own devices | dst only (perf warning) |
| `autogroup:danger-all` | All nodes including tagged | src or dst |

### Enterprise Example (your-company pattern)
```json
{
  "groups": {
    "group:admin": ["admin"],
    "group:clients": ["clients"]
  },
  "tagOwners": {
    "tag:exit-node": ["group:admin"],
    "tag:server": ["group:admin"]
  },
  "acls": [
    // Admin: full access to all nodes
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    // Clients: internet only through exit nodes
    {
      "action": "accept",
      "src": ["group:clients"],
      "dst": ["autogroup:internet:*"]
    },
    // Clients cannot see or access other clients' devices
    // (no rule = denied by default)

    // Admin can SSH to tagged servers
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["tag:server:22"]
    }
  ]
}
```

### ACL Management Commands
```bash
# Validate policy file
headscale policy check --file /etc/headscale/acls.json

# Get current policy (database mode)
headscale policy get --bypass-grpc-and-access-database-directly > policy.json

# Set policy (database mode)
headscale policy set --bypass-grpc-and-access-database-directly --file policy.json

# Reload after file change (file mode)
systemctl reload headscale
# or
kill -HUP $(pidof headscale)
```

---

## 5. DERP Server Configuration

### Custom DERP Map (derp.yaml)
```yaml
# /etc/headscale/derp.yaml
regions:
  900:
    regionid: 900
    regioncode: my-derp
    regionname: Malaysia DERP
    nodes:
      - name: "900a"
        regionid: 900
        hostname: "your-derp-server-ip"  # or domain name
        ipv4: "your-derp-server-ip"
        derpport: 33331           # DERP relay port
        stunport: 3478            # STUN port
        insecurefortests: true    # Skip TLS verify (self-signed)
  901:
    regionid: 901
    regioncode: sg-derp
    regionname: Singapore DERP
    nodes:
      - name: "901a"
        regionid: 901
        hostname: "your-headscale-server-ip"
        ipv4: "your-headscale-server-ip"
        derpport: 33331
        stunport: 3478
        insecurefortests: true
```

### Standalone DERP Server (derper)
```bash
# Install
go install tailscale.com/cmd/derper@latest

# Run with systemd
# /etc/systemd/system/derper.service
[Unit]
Description=Tailscale DERP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/derper \
  --hostname=your-domain.com \
  --a=:33331 \
  --stun \
  --stun-port=3478 \
  --verify-clients \
  --verify-client-url=http://127.0.0.1:8080/verify
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### DERP Firewall Rules
```bash
ufw allow 33331/tcp    # DERP relay (HTTPS)
ufw allow 3478/udp     # STUN
ufw allow 41641/udp    # WireGuard direct connections
```

### --verify-clients
- Requires tailscaled running on same machine, joined to the Headscale network
- Only authenticated tailnet members can use the relay
- Without it: anyone can use your DERP as relay (bandwidth theft)
- Verify URL points to Headscale: `--verify-client-url=http://127.0.0.1:8080/verify`
- `--verify-client-url-fail-open`: If Headscale unreachable, allow access (default: true)

---

## 6. Node & Key Management

### Users
```bash
headscale users create admin
headscale users create clients
headscale users list
headscale users rename old-name new-name
headscale users destroy <name>   # WARNING: removes all nodes
```

### Pre-Auth Keys
```bash
# Create reusable key (for multiple devices)
headscale preauthkeys create --user clients --reusable --expiration 8760h

# Create one-time key
headscale preauthkeys create --user admin --expiration 8760h

# List keys
headscale preauthkeys list
headscale preauthkeys list --user clients
```

### Node Management
```bash
# List all nodes
headscale nodes list

# Register pending node
headscale nodes register --key <nodekey> --user clients

# Delete node
headscale nodes delete --identifier <node-id>

# Rename node
headscale nodes rename --identifier <node-id> <new-name>

# Move node to different user
headscale nodes move --identifier <node-id> --user admin

# Set tags on node
headscale nodes tag --identifier <node-id> --tags tag:exit-node,tag:server
```

### Route Management (Exit Nodes & Subnets)
```bash
# List all routes
headscale nodes routes list

# Enable exit node routes
headscale nodes routes enable --identifier <node-id> --route 0.0.0.0/0
headscale nodes routes enable --identifier <node-id> --route ::/0

# Enable subnet route
headscale nodes routes enable --identifier <node-id> --route 192.168.1.0/24

# Disable route
headscale nodes routes disable --identifier <node-id> --route 0.0.0.0/0
```

### API Keys (for Headplane)
```bash
# Create API key
headscale apikeys create --expiration 8760h

# List API keys
headscale apikeys list

# Expire (revoke) API key
headscale apikeys expire --prefix <key-prefix>
```

---

## 7. Client Configuration

### macOS
```bash
# Connect to Headscale
tailscale up --login-server=https://your-headscale.com --authkey=hskey-auth-xxx --accept-routes

# Set exit node
tailscale set --exit-node=<node-name-or-ip> --exit-node-allow-lan-access=true

# Check status
tailscale status
tailscale netcheck

# Switch accounts (multiple Headscale servers)
# Tailscale Settings → Accounts → switch
```

### Windows (PowerShell)
```powershell
# Install silently
$installerUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-latest.exe"
$installerPath = "$env:TEMP\tailscale-setup.exe"
Invoke-WebRequest -Uri $installerUrl -OutFile $installerPath
Start-Process -FilePath $installerPath -ArgumentList "/quiet" -Wait

# Wait for service
Start-Sleep -Seconds 30

# Connect
& "C:\Program Files\Tailscale\tailscale.exe" up `
  --login-server=https://your-headscale.com `
  --authkey=hskey-auth-xxx `
  --reset --accept-routes --unattended

# Set exit node
& "C:\Program Files\Tailscale\tailscale.exe" set `
  --exit-node=<node-name> `
  --exit-node-allow-lan-access=true
```

### Linux (Exit Node Server)
```bash
# Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf

# Connect and advertise as exit node
tailscale up \
  --login-server=https://your-headscale.com \
  --authkey=hskey-auth-xxx \
  --advertise-exit-node \
  --accept-routes

# Verify
tailscale status
curl ifconfig.me
```

---

## 8. MagicDNS

### How It Works
- Tailscale client installs local DNS resolver at 100.100.100.100
- Queries for `<hostname>.<base_domain>` → resolved to Tailscale IP
- All other queries → forwarded to configured global nameservers
- FQDNs: `nodename.base_domain` (v0.23+ removed username from FQDN)

### Configuration
```yaml
# In Headscale config.yaml
dns:
  magic_dns: true
  base_domain: vpn.example.com
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
    # Split DNS for internal domains
    restricted:
      internal.corp.com:
        - 10.0.0.53
  # Custom records
  extra_records:
    - name: "dashboard.vpn.example.com"
      type: "A"
      value: "100.64.0.2"
```

### NextDNS Integration
```yaml
dns:
  nameservers:
    global:
      - https://dns.nextdns.io/YOUR_CONFIG_ID
```
Headscale auto-adds node metadata to queries for per-device NextDNS analytics.

---

## 9. Headplane (Web UI)

### Docker Deployment
```bash
docker run -d --name headplane \
  -p 3000:3000 \
  -e HEADSCALE_URL=http://127.0.0.1:8080 \
  -e API_KEY=hskey-api-xxxxx \
  --restart unless-stopped \
  ghcr.io/tale/headplane:latest
```

### API Key for Headplane
```bash
headscale apikeys create --expiration 8760h
# Save the FULL key immediately — cannot be retrieved later
```

---

## 10. Troubleshooting

### Connection Issues
```bash
# Step 1: Check client is connected
tailscale status
# Look for "idle", "active", "direct", "relay"

# Step 2: Network diagnostics
tailscale netcheck
# Check: UDP working? NAT type? DERP reachable?

# Step 3: Ping specific peer
tailscale ping <peer-ip>
# Shows: direct or DERP, latency, path

# Step 4: Check DERP connectivity
tailscale debug derp-map
# Shows all DERP regions and connectivity

# Step 5: Server-side check
headscale nodes list
# Verify node is registered, online, not expired
```

### Common Issues

**Node shows "offline" but is connected:**
- Clock skew between client and server
- Key expired: `headscale nodes list` check Expired column
- Client version too old (v0.28 requires ≥ v1.74.0)

**Always relayed, never direct:**
- Both peers behind symmetric NAT → expected, use closer DERP
- UDP blocked by firewall → open UDP 41641
- `MappingVariesByDestAddr: true` in netcheck = hard NAT

**MagicDNS not working:**
- `--accept-dns=true` must be set on client (default)
- base_domain must differ from server_url
- At least one global nameserver required if override_local_dns is true

**ACL not taking effect:**
- Restart/reload Headscale after changing acl file
- Check policy syntax: `headscale policy check --file acls.json`
- v0.28: wildcard `*` = tailnet only, not all IPs

**Exit node not working:**
- IP forwarding must be enabled on exit node server
- Route must be approved: `headscale nodes routes enable`
- Client must set: `tailscale set --exit-node=<name>`

### Performance Tuning
```yaml
# Headscale config tuning (rarely needed)
tuning:
  # Batch node updates to reduce CPU (default: 250ms)
  batched_change_delay: 250ms
  # Node update channel buffer (default: 32)
  node_mapsession_channel_size: 32
```

---

## 11. Backup & Recovery

### Headscale Database
```bash
# Backup SQLite database
cp /var/lib/headscale/db.sqlite /backup/headscale-$(date +%Y%m%d).sqlite

# Backup config
cp /etc/headscale/config.yaml /backup/
cp /etc/headscale/acls.json /backup/
cp /etc/headscale/derp.yaml /backup/

# Restore: stop headscale, copy files back, start headscale
```

### Client Re-registration
If control server is rebuilt from scratch:
- All clients need to re-authenticate (`tailscale up --login-server=... --authkey=... --reset`)
- WireGuard keys are re-negotiated automatically
- Tailscale IPs may change unless database is restored


## 12. Deployment Reference Template

### Headscale Server Setup
```
# Replace with your own values:
HEADSCALE_SERVER_IP=your.server.ip
HEADSCALE_DOMAIN=hs.yourdomain.com
HEADSCALE_PORT=8080
DERP_PORT=33331
HEADPLANE_PORT=3000
```

### Key Inventory Template
```
# Admin pre-auth key (reusable):  headscale preauthkeys create --user admin --reusable --expiration 8760h
# Client pre-auth key (reusable): headscale preauthkeys create --user clients --reusable --expiration 8760h
# Headplane API key:              headscale apikeys create --expiration 90d
```

### Firewall Ports
| Port | Protocol | Purpose |
|------|----------|---------|
| 8080/tcp | HTTP | Headscale API (localhost or reverse proxy) |
| 3000/tcp | HTTP | Headplane Web UI |
| 33331/tcp | HTTPS | DERP relay |
| 3478/udp | STUN | NAT traversal assistance |
| 41641/udp | WireGuard | Direct P2P connections |
| 9090/tcp | HTTP | Headscale metrics (localhost only) |
