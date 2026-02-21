---
name: docker-skill
description: Docker container management skill for deploying, managing, troubleshooting, and optimizing Docker containers and Docker Compose stacks. Use this skill whenever the user mentions Docker, containers, docker-compose, images, volumes, container deployment, or wants to run any service in Docker. Also trigger when deploying self-hosted tools (Termix, AdGuard, Uptime Kuma, OpenClaw, etc.), managing container networking/ports, debugging container issues, or doing Docker backups/migrations. Covers Docker Desktop on macOS and Docker Engine on Linux VPS.
---

# Docker Skill

## User Environment

### Docker Hosts
- **MacBook (macOS, Apple Silicon)**: Docker Desktop, primary local dev
- **Mac Mini (192.168.x.x, macOS)**: Docker Desktop, local services host (Termix on port 7070)
- **VPS-1 VPS (your.vps.ip, Ubuntu 24)**: Docker Engine, runs Termix (7070), Nexterm (6989), Headscale stack
- **VPS-2 VPS (your.vps.ip, Ubuntu 24)**: Docker Engine, runs Headscale stack
- **VPS-3 VPS (your.vps.ip, Ubuntu)**: Docker Engine, personal + work use

### Port Allocation Reference
#### VPS-1 (your.vps.ip)
| Port | Service |
|------|---------|
| 2233 | SSH |
| 3000 | Headplane (localhost only) |
| 6989 | Nexterm |
| 7070 | Termix |
| 8080 | Headscale (localhost only) |
| 9090 | Headscale metrics (localhost only) |

#### VPS-2 (your.vps.ip)
| Port | Service |
|------|---------|
| 2233 | SSH |
| 3000 | Headplane |
| 8080 | Headscale |
| 33331 | DERP |

#### Mac Mini (192.168.x.x)
| Port | Service |
|------|---------|
| 7070 | Termix |

### Naming Convention
Always use these names or IPs:
- your.vps.ip = VPS-1
- your.vps.ip = VPS-2
- your.vps.ip = VPS-3
- Never use "MY VPS", "SG VPS", "router" etc.

---

## Quick Reference Commands

### Container Lifecycle
```bash
# Run a container (detached, auto-restart, named, with volume and port)
docker run -d --name <name> -p <host>:<container> \
  -v <volume-name>:/path/in/container \
  --restart unless-stopped \
  <image>:<tag>

# Stop / Start / Restart
docker stop <name>
docker start <name>
docker restart <name>

# Remove container (must stop first, or use -f)
docker rm -f <name>

# View running containers
docker ps

# View all containers (including stopped)
docker ps -a

# View logs
docker logs <name>
docker logs -f <name>          # follow (live)
docker logs --tail 100 <name>  # last 100 lines

# Execute command inside running container
docker exec -it <name> /bin/sh
docker exec -it <name> bash

# Inspect container details
docker inspect <name>

# Container resource usage
docker stats
docker stats <name>
```

### Image Management
```bash
# List images
docker images

# Pull image
docker pull <image>:<tag>

# Remove image
docker rmi <image>:<tag>

# Remove unused images
docker image prune -a

# Remove all unused resources (images, containers, volumes, networks)
docker system prune -a
```

### Volume Management
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect <name>

# Create volume
docker volume create <name>

# Remove volume (data will be lost!)
docker volume rm <name>

# Remove unused volumes
docker volume prune
```

### Network Management
```bash
# List networks
docker network ls

# Create network
docker network create <name>

# Connect container to network
docker network connect <network> <container>

# Inspect network
docker network inspect <name>
```

---

## Docker Compose

### Basic compose.yaml Structure
```yaml
services:
  app:
    image: <image>:<tag>
    container_name: <name>
    ports:
      - "<host_port>:<container_port>"
    volumes:
      - <volume_name>:/path/in/container
    environment:
      - KEY=value
    restart: unless-stopped
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
    restart: unless-stopped

volumes:
  <volume_name>:
  db-data:
```

### Compose Commands
```bash
# Start all services (detached)
docker compose up -d

# Stop all services
docker compose down

# Stop and remove volumes (WARNING: deletes data)
docker compose down -v

# View logs
docker compose logs
docker compose logs -f <service>

# Rebuild and restart
docker compose up -d --build

# List running services
docker compose ps

# Execute command in service
docker compose exec <service> sh
```

---

## Common Deployment Patterns

### Self-Hosted Service (Single Container)
```bash
docker run -d --name <service> \
  -p <port>:<container_port> \
  -v <service>-data:/app/data \
  --restart unless-stopped \
  <image>:<tag>
```

### Localhost-Only Service (Not Exposed Externally)
```bash
docker run -d --name <service> \
  -p 127.0.0.1:<port>:<container_port> \
  -v <service>-data:/app/data \
  --restart unless-stopped \
  <image>:<tag>
```

### With Environment Variables
```bash
docker run -d --name <service> \
  -p <port>:<container_port> \
  -v <service>-data:/app/data \
  -e KEY1=value1 \
  -e KEY2=value2 \
  --restart unless-stopped \
  <image>:<tag>
```

---

## Backup & Migration

### Backup a Volume
```bash
# Create tarball from volume
docker run --rm \
  -v <volume-name>:/data \
  -v /tmp:/backup \
  alpine tar czf /backup/<backup-name>.tar.gz -C /data .
```

### Restore a Volume
```bash
# Stop container first
docker stop <container>

# Restore from tarball
docker run --rm \
  -v <volume-name>:/data \
  -v /tmp:/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/<backup-name>.tar.gz -C /data"

# Start container
docker start <container>
```

### Migrate Between Hosts
```bash
# On source: backup
docker run --rm -v <volume>:/data -v /tmp:/backup alpine tar czf /backup/<name>.tar.gz -C /data .

# Transfer
scp -P <port> <user>@<source_ip>:/tmp/<name>.tar.gz /tmp/

# On target: create container, stop, restore, start
docker run -d --name <service> -p <port>:<cport> -v <volume>:/app/data --restart unless-stopped <image>
docker stop <service>
docker run --rm -v <volume>:/data -v /tmp:/backup alpine sh -c "rm -rf /data/* && tar xzf /backup/<name>.tar.gz -C /data"
docker start <service>
```

---

## Troubleshooting

### Container Won't Start
```bash
# Check logs
docker logs <name>

# Check if port is in use
ss -tlnp | grep <port>        # Linux
lsof -i :<port>               # macOS

# Check container exit code
docker inspect <name> --format='{{.State.ExitCode}}'
```

### Connection Refused
- Verify port mapping: `docker ps` shows ports
- Check if service binds to 0.0.0.0 inside container (not 127.0.0.1)
- On VPS: check `ufw status` for firewall rules
- On macOS with Surge: ensure domain/IP has DIRECT rule if accessing locally

### High Memory Usage
```bash
# Check per-container usage
docker stats --no-stream

# Set memory limit
docker run -d --name <name> --memory=512m <image>

# In compose:
# deploy:
#   resources:
#     limits:
#       memory: 512M
```

### Docker Desktop Not Running (macOS)
```
failed to connect to the docker API at unix:///var/run/docker.sock
```
→ Open Docker Desktop app, wait for green status icon in menu bar.
→ Enable "Start Docker Desktop when you sign in" in Settings.

### Cleanup Disk Space
```bash
# See disk usage
docker system df

# Remove everything unused
docker system prune -a --volumes
```

---

## Security Best Practices

1. **Pin image versions**: Use `image:1.2.3` not `image:latest`
2. **Don't store secrets in images**: Use environment variables or Docker secrets
3. **Use `--restart unless-stopped`**: Auto-recovers from crashes but respects manual stops
4. **Limit resources**: Set `--memory` and `--cpus` for production containers
5. **Localhost binding**: For internal services, bind to `127.0.0.1:<port>` not `0.0.0.0:<port>`
6. **Regular updates**: `docker pull <image>:<tag>` then recreate container
7. **Use named volumes**: Easier to backup and manage than bind mounts

---

## Docker Desktop macOS Tips

- **Auto-start**: Settings → General → Start Docker Desktop when you sign in
- **Resource limits**: Settings → Resources → adjust CPU/Memory (default is fine for most)
- **Disk usage**: Settings → Resources → Disk image size
- **Docker socket**: Available at `unix:///var/run/docker.sock`
- **Compose V2**: Use `docker compose` (with space, not hyphen)

---

## Commonly Deployed Services

| Service | Image | Default Port | Notes |
|---------|-------|-------------|-------|
| Termix | ghcr.io/lukegus/termix:latest | 8080 | SSH管理 |
| AdGuard Home | adguard/adguardhome | 3000(web), 53(DNS) | DNS过滤 |
| Uptime Kuma | louislam/uptime-kuma | 3001 | 监控 |
| Nginx Proxy Manager | jc21/nginx-proxy-manager | 81(web), 80, 443 | 反向代理 |
| Portainer | portainer/portainer-ce | 9443 | Docker管理GUI |
| WireGuard | linuxserver/wireguard | 51820/udp | VPN |
| Headscale | headscale/headscale | 8080 | Tailscale控制服务器 |
| OpenClaw | See official docs | Varies | AI助手 |
