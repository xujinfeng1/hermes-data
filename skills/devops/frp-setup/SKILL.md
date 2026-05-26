---
name: frp-setup
description: "Set up frp (fast reverse proxy) to expose local services to the internet via a VPS."
version: 1.1.0
author: Hermes Agent
platforms: [linux, macos]
metadata:
  hermes:
    tags: [frp, tunnel, reverse-proxy, vps, ecs, remote-access]
---

# frp 内网穿透

Use frp to expose local services (like Hermes WebUI) to the public internet
through a VPS with a public IP. Architecture:

```
Phone (outside) → VPS:8080 (frps) → Home Mac:9119 (frpc) → Hermes dashboard
```

## Prerequisites

- A VPS with public IP (Alibaba Cloud ECS, Tencent Cloud, etc.)
- SSH access to the VPS (root or sudo)
- frp binaries for both server (VPS) and client (local machine)

## Server Setup (frps on VPS)

```bash
# SSH into VPS
ssh root@<VPS_IP>

# Download frp (latest from GitHub)
cd /tmp
curl -fsSL -o frp.tar.gz \
  "https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_linux_amd64.tar.gz"
tar xzf frp.tar.gz
cp frp_0.68.1_linux_amd64/frps /usr/local/bin/
mkdir -p /etc/frp
```

### frps config (`/etc/frp/frps.toml`)

```toml
bindPort = 7000
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "your-password"
```

### systemd service (`/etc/systemd/system/frps.service`)

```ini
[Unit]
Description=frp server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable frps
systemctl start frps
```

## Client Setup (frpc on local machine)

```bash
# Download frp (macOS arm64)
curl -fsSL -o /tmp/frp.tar.gz \
  "https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_darwin_arm64.tar.gz"
tar xzf /tmp/frp.tar.gz
cp frp_0.68.1_darwin_arm64/frpc /usr/local/bin/
chmod +x /usr/local/bin/frpc
```

### frpc config

```toml
serverAddr = "<VPS_IP>"
serverPort = 7000

[[proxies]]
name = "webui"
type = "tcp"
localIP = "127.0.0.1"
localPort = 9119
remotePort = 8080
```

```bash
frpc -c frpc.toml
```

## Firewall / Security Group

On Alibaba Cloud (and similar cloud providers), the VPS has a **security group**
that blocks incoming traffic by default. Open these ports in the security group
**inbound rules**:

| Port | Protocol | Purpose |
|------|----------|---------|
| 7000 | TCP | frp control connection |
| 7500 | TCP | frp admin dashboard |
| 8080 | TCP | Forwarded service (Hermes WebUI) |

Console path: ECS → instance → Security Group → Inbound Rules → Add.

Also check iptables on the VPS:
```bash
iptables -L INPUT -n | head -10
# If needed:
iptables -I INPUT -p tcp --dport 7000 -j ACCEPT
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```

## Access

After setup, the local service is accessible at `http://<VPS_IP>:8080`.
The frp admin dashboard is at `http://<VPS_IP>:7500` (login with webServer credentials).

## Access

After setup, the local service is accessible at `http://<VPS_IP>:<remotePort>`.

## macOS LaunchAgent 持久化

When running frpc on macOS, use a LaunchAgent plist so it auto-restarts on crash/reboot:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermes.frpc</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/xu/.local/bin/frpc</string>
        <string>-c</string>
        <string>/Users/xu/.hermes/frpc.toml</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key>
    <string>/Users/xu/.hermes/logs/frpc.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/xu/.hermes/logs/frpc.log</string>
</dict>
</plist>
```

Save to `~/Library/LaunchAgents/com.hermes.frpc.plist`, then:
```bash
launchctl load ~/Library/LaunchAgents/com.hermes.frpc.plist
launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist  # to stop
```

## Pitfalls

- **Proxy already exists / port already used**: When frpc disconnects abruptly (kill -9), frps holds the session for ~90s (heartbeat timeout). During this window, new connections with the same proxy names or ports will fail. Fix: wait 90s, or use completely unique proxy names AND remote ports.
- **KeepAlive respawning**: If frpc is managed by LaunchAgent with KeepAlive=true, killing the process will immediately respawn it. Use `launchctl unload` first, then edit config, then `launchctl load`.
- **Security group**: On Alibaba Cloud, ports like 8888 may not be in the ECS security group. Ports 80 and 8080 are commonly open. Test with a known-working port first.

- **LaunchAgent auto-restart (macOS)**: Hermes manages frpc via `~/Library/LaunchAgents/com.hermes.frpc.plist` with `KeepAlive=true`. Killing frpc causes instant respawn. Always use `launchctl unload/load` instead of `kill`/`pkill`.

- **Proxy session timeout**: When frpc disconnects, frps holds proxy registrations for ~90s (heartbeat timeout). Restarting within this window causes "proxy already exists" / "port already used" errors. Fix: wait 90s, OR change proxy name AND remote port, OR restart frps.

- **Security group blocking**: Cloud providers block incoming ports by default. If frpc says "start proxy success" but curl times out, the port is blocked at the security group level (ECS → Security Group → Inbound Rules).

- **Version mismatch**: frps and frpc must be same version.
- **GitHub downloads in China**: Use `ghproxy.net` mirror prefix.

## Verification

```bash
curl -sI http://<VPS_IP>:<remotePort>/
tail -f ~/.hermes/logs/frpc.log
```
