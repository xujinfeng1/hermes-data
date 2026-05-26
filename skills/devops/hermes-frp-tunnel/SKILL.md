---
name: hermes-frp-tunnel
description: Set up frp tunnel to expose Hermes WebUI to the internet via an Alibaba Cloud ECS.
---

# Hermes WebUI via frp Tunnel

Public access to Hermes dashboard through frp (fast reverse proxy) with an Alibaba Cloud ECS as the server.

## Architecture

```
Browser → http://xujinfeng.top (DNS → ECS public IP) → frps:80 → frpc → Mac:9119
```

## Prerequisites

- Alibaba Cloud ECS (Ubuntu 22.04) with public IP
- Domain DNS pointing to ECS
- Hermes dashboard running with `--insecure --tui`
- CORS fix applied to `hermes_cli/web_server.py` (see below)

## ECS Setup (frps server)

```bash
# Download frp
cd /tmp
curl -fsSL -o frp.tar.gz "https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_linux_amd64.tar.gz"
tar xzf frp.tar.gz
cp frp_0.68.1_linux_amd64/frps /usr/local/bin/
mkdir -p /etc/frp

# Config: /etc/frp/frps.toml
cat > /etc/frp/frps.toml << 'EOF'
bindPort = 7000
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "hermes2026"
EOF

# Systemd service: /etc/systemd/system/frps.service
cat > /etc/systemd/system/frps.service << 'EOF'
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
EOF

systemctl daemon-reload
systemctl enable --now frps
```

### Alibaba Cloud Security Group

Open inbound ports: **7000** (frp), **7500** (dashboard), **80** (HTTP), **8080** (alt).

## Mac Setup (frpc client)

```bash
# Download frp for macOS
curl -fsSL -o /tmp/frp.tar.gz "https://ghproxy.net/https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_darwin_arm64.tar.gz"
tar xzf /tmp/frp.tar.gz
cp frp_0.68.1_darwin_arm64/frpc ~/.local/bin/
chmod +x ~/.local/bin/frpc

# Config: ~/.hermes/frpc.toml
cat > ~/.hermes/frpc.toml << 'EOF'
serverAddr = "ECS_PUBLIC_IP"
serverPort = 7000

[[proxies]]
name = "hermes-webui-80"
type = "tcp"
localIP = "127.0.0.1"
localPort = 9119
remotePort = 80

[[proxies]]
name = "hermes-webui-8080"
type = "tcp"
localIP = "127.0.0.1"
localPort = 9119
remotePort = 8080
EOF

# Auto-start via launchd: ~/Library/LaunchAgents/com.hermes.frpc.plist
cat > ~/Library/LaunchAgents/com.hermes.frpc.plist << 'EOF'
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
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.hermes.frpc.plist
```

## CORS Fix

The dashboard's CORS middleware rejects non-localhost origins. When running with `--insecure`, patch `hermes_cli/web_server.py` `start_server()`:

```python
if host not in _LOCALHOST:
    _CORS_ALLOW_ORIGIN_REGEX = r"https?://.*"
    for mw in app.user_middleware:
        if mw.cls.__name__ == "CORSMiddleware":
            mw.kwargs["allow_origin_regex"] = _CORS_ALLOW_ORIGIN_REGEX
            break
```

## Troubleshooting

- **frpc can't connect**: check ECS security group has port 7000 open
- **HTTP timeout on port 80/8080**: check security group port, check frpc is running
- **CORS errors in browser**: verify the CORS patch is applied and dashboard was restarted
- **Chat tab can't type**: `pip install ptyprocess==0.7.0` — the PTY bridge needs this
