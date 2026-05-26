# Debugging frp "proxy already exists" / "port already used"

## Symptom
After restarting frpc, all proxies fail with:
```
[W] [cdp-agent-demo] start error: proxy [cdp-agent-demo] already exists
[W] [hermes-webui-8080] start error: port already used
```

## Root Cause
frps server holds proxy registrations for ~90 seconds after client disconnects
(heartbeat timeout). New frpc instances with the same proxy names or remote ports
are rejected until old sessions expire.

## Resolution Path (tried in order)

### 1. Wait (simplest but slow)
```bash
pkill -9 frpc
# Wait 100+ seconds
/Users/xu/.local/bin/frpc -c ~/.hermes/frpc.toml
```

### 2. Unique proxy names (works but may still hit port conflicts)
```toml
name = "cdp-agent-demo-v2"  # instead of "cdp-agent-demo"
```

### 3. Unique names + unique remote ports
```toml
name = "cdp-demo-454234"
remotePort = 9090  # completely fresh port
```
Note: fresh ports may be blocked by cloud security group.

### 4. Restart frps on ECS (requires SSH access)
```bash
ssh root@8.163.113.208 "systemctl restart frps"
```

## What Actually Worked
Killed all frpc, waited 100s for frps heartbeat timeout, then started with original config.
Also: Hermes manages frpc via LaunchAgent (`~/Library/LaunchAgents/com.hermes.frpc.plist`)
with `KeepAlive=true`, so `pkill` causes instant respawn. Must use `launchctl unload/load`.

## Cloud Security Group
Port 8888 was unreachable even with successful frp proxy. Root cause: Alibaba Cloud
security group only had ports 80 and 8080 open. Port 8888 needed to be added in
ECS → Security Group → Inbound Rules. Workaround: use port 8080 for the CDP agent.

## Key Commands
```bash
# Check frpc process
ps aux | grep frpc | grep -v grep

# Read full frpc log (LaunchAgent managed)
tail -f ~/.hermes/logs/frpc.log

# Proper restart via LaunchAgent
launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist
# ... wait ...
launchctl load ~/Library/LaunchAgents/com.hermes.frpc.plist

# Test from outside
curl -s --connect-timeout 5 http://8.163.113.208:8080/health
```
