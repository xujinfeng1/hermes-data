# CDP Agent Deployment

## Local LaunchAgent (macOS)

Path: `~/Library/LaunchAgents/com.cdp.agent.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.cdp.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/xu/AgentProject/agent-for-saas/venv/bin/python</string>
        <string>-m</string>
        <string>uvicorn</string>
        <string>app.main:app</string>
        <string>--host</string>
        <string>0.0.0.0</string>
        <string>--port</string>
        <string>8800</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/xu/AgentProject/agent-for-saas</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/xu/AgentProject/agent-for-saas/logs/agent.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/xu/AgentProject/agent-for-saas/logs/agent.log</string>
</dict>
</plist>
```

Commands:
```bash
launchctl load ~/Library/LaunchAgents/com.cdp.agent.plist
launchctl unload ~/Library/LaunchAgents/com.cdp.agent.plist
```

## frp Tunnel to ECS

Config: `~/.hermes/frpc.toml`
```toml
serverAddr = "8.163.113.208"
serverPort = 7000

[[proxies]]
name = "cdp-demo"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8800
remotePort = 8080
```

Managed by `com.hermes.frpc.plist` LaunchAgent (KeepAlive=true).

### frp Pitfalls

1. **LaunchAgent respawn**: Killing frpc does nothing. Use `launchctl unload/load`.
2. **Port held for 90s**: After disconnect, frps holds the port. Wait before restarting.
3. **Security group**: Port must be in ECS inbound rules. 8080 was open; 8888 was not.
4. **Unique proxy names**: frps tracks by name. Changing names alone doesn't help - must change ports OR wait 90s.

## External Access

```
URL: http://8.163.113.208:8080
Demo: http://8.163.113.208:8080/demo
Health: http://8.163.113.208:8080/health
```
