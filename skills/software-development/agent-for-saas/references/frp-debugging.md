# frp Deployment Debugging

## Problem: proxy already exists / port already used

When frpc is killed and restarted too quickly, the frps server still holds
the old proxy registrations. Default heartbeat timeout is ~90 seconds.

### Symptom
```
[W] [cdp-agent-demo] start error: proxy [cdp-agent-demo] already exists
[W] [cdp-agent-demo] start error: port already used
```

### Root Cause
- frpc process killed (SIGKILL/SIGTERM)
- frps hasn't received heartbeat timeout yet
- Old proxy name + port still registered on frps
- New frpc can't claim them

### Solutions (in order of preference)

1. **Wait 90 seconds** between kill and restart
2. **Use unique proxy names** — append timestamp: `cdp-demo-{timestamp}`
3. **Use different remote ports** — but security group may block them
4. **Restart frps on VPS** (requires SSH): `systemctl restart frps`

### LaunchAgent Pattern

Hermes auto-creates `~/Library/LaunchAgents/com.hermes.frpc.plist` with:
- `KeepAlive=true` → auto-restarts frpc on crash
- `RunAtLoad=true` → starts at login

To change config:
```bash
# 1. Stop
launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist

# 2. Check no frpc running
ps aux | grep frpc | grep -v grep

# 3. Edit config
vim ~/.hermes/frpc.toml

# 4. Wait 90s for frps timeout

# 5. Restart
launchctl load ~/Library/LaunchAgents/com.hermes.frpc.plist

# 6. Verify
curl http://8.163.113.208:8080/health
```

### Security Group

Alibaba Cloud ECS inbound rules block ports by default.
Known open ports: 80, 8080, 7000 (frp control), 7500 (frp admin).
To add a port: ECS console → Security Group → Inbound Rules → Add.

## Specific session trace

Session started 2026-05-22 with frpc already running (PID 1913).
Multiple kill/restart cycles created orphaned proxies on frps.
Eventually resolved by:
1. Killing ALL frpc processes including LaunchAgent-spawned ones
2. Using unique proxy names to bypass stale registrations
3. Mapping CDP agent (localhost:8800) to remote port 8080 (known open)
