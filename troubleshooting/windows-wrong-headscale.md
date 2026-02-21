# Windows 客户端连错 Headscale 服务器 — 排查与修复

## 症状

Windows 客户端运行安装脚本后，`tailscale status` 显示连接的是旧的 Headscale 服务器，而非脚本指定的新服务器。出口 IP 显示为错误的服务器地址。

## 根因

Windows 设备之前已经安装过 Tailscale 并注册到另一个 Headscale 实例。即使脚本中使用了 `--reset` 参数，**旧的登录状态和节点注册信息并未完全清除**。

Tailscale 的 `--reset` 只重置网络偏好设置，不会清除已有的认证状态。设备仍然记住上次登录的控制服务器。

## 诊断方法

```powershell
# 查看当前连接的服务器
tailscale status

# 查看详细信息（包括 login server）
tailscale debug prefs
```

如果 `ControlURL` 显示的不是你期望的服务器，就是这个问题。

## 修复：安装脚本中加入完整清除

在连接新服务器之前，必须先 logout：

```powershell
# 1. 先完全登出（清除旧的认证状态）
& "C:\Program Files\Tailscale\tailscale.exe" logout

# 2. 等待登出完成
Start-Sleep -Seconds 3

# 3. 重新连接到正确的服务器
& "C:\Program Files\Tailscale\tailscale.exe" up `
    --login-server=https://hs.yourdomain.com `
    --authkey=your-preauth-key `
    --reset `
    --accept-routes `
    --unattended

# 4. 设置出口节点
& "C:\Program Files\Tailscale\tailscale.exe" set --exit-node=your-exit-node
```

## 完整的 Windows 安装脚本模板

```powershell
#Requires -RunAsAdministrator
# SAFENET VPN Installation Script Template
# Run as Administrator

$ErrorActionPreference = "Stop"
$SERVER = "https://hs.yourdomain.com"
$AUTHKEY = "your-preauth-key"
$EXIT_NODE = "your-exit-node"
$TAILSCALE = "C:\Program Files\Tailscale\tailscale.exe"

Write-Host "=== VPN Installation ===" -ForegroundColor Cyan

# Step 1: Download and install Tailscale (if not installed)
if (-not (Test-Path $TAILSCALE)) {
    Write-Host "Downloading Tailscale..." -ForegroundColor Yellow
    $installerUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-latest.exe"
    $installerPath = "$env:TEMP\tailscale-setup.exe"
    Invoke-WebRequest -Uri $installerUrl -OutFile $installerPath
    Start-Process -FilePath $installerPath -Args "/install /quiet" -Wait
    Start-Sleep -Seconds 5
}

# Step 2: Clear any previous Tailscale state (CRITICAL)
Write-Host "Clearing previous VPN state..." -ForegroundColor Yellow
try {
    & $TAILSCALE logout 2>$null
    Start-Sleep -Seconds 3
} catch {
    # Ignore if not logged in
}

# Step 3: Connect to server
Write-Host "Connecting to VPN server..." -ForegroundColor Yellow
& $TAILSCALE up `
    --login-server=$SERVER `
    --authkey=$AUTHKEY `
    --reset `
    --accept-routes `
    --unattended

Start-Sleep -Seconds 5

# Step 4: Set exit node
Write-Host "Setting exit node..." -ForegroundColor Yellow
& $TAILSCALE set --exit-node=$EXIT_NODE

# Step 5: Verify
Write-Host ""
Write-Host "=== Connection Status ===" -ForegroundColor Green
& $TAILSCALE status
Write-Host ""
Write-Host "VPN connected successfully!" -ForegroundColor Green
pause
```

## 客户部署注意事项

1. **永远在连接前先 `tailscale logout`** — 不管是新装还是重装
2. **不要假设客户机器是干净的** — 可能装过其他 Tailscale 网络
3. **脚本要幂等** — 多次运行结果一致，不会因为旧状态出问题
4. **验证步骤必须有** — 脚本最后显示 `tailscale status` 让客户确认

## 教训总结

1. **`--reset` ≠ 完全清除** — reset 只重置偏好，不清除认证
2. **`tailscale logout` 才是真正的断开** — 清除节点注册和认证状态
3. **企业部署脚本必须处理"非干净"环境** — 假设最坏情况
