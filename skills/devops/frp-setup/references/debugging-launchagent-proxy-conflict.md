# frp LaunchAgent Proxy Conflict — Full Debugging Transcript

## Scenario
Adding a new proxy (`cdp-agent-demo → localhost:8800 → VPS:8888`) to an existing
Hermes frpc setup on macOS (Apple Silicon). frpc managed by LaunchAgent with
KeepAlive=true.

## Error Timeline

### 1. Initial attempt: "proxy already exists"
```
[W] [cdp-agent-demo] start error: proxy [cdp-agent-demo] already exists
```
**Cause**: Old frpc session from a killed process still registered on frps.
**Fix attempted**: Renamed to `cdp-agent-demo-v2`.
**Result**: Still failed — old session also holds the remote port.

### 2. Renamed proxies: "port already used"
```
[W] [hermes-webui-8080-v2] start error: port already used
```
**Cause**: Old session still holds ports 8080, 80, 8888 on frps.
**Key insight**: Both proxy name AND remote port must be unique. Renaming alone
insufficient.

### 3. New ports (9090, 9999): "start proxy success" but unreachable
```
[I] [cdp-9999] start proxy success
[I] [webui-9090] start proxy success
```
But external `curl` timed out. **Root cause**: Alibaba Cloud security group
only had ports 80 and 8080 open. Ports 8888, 9090, 9999 all blocked.

### 4. Switch to known-open port 8080: SUCCESS
Mapped `cdp-demo → localhost:8800 → VPS:8080`. Port 8080 was already open in
the security group (used by Hermes WebUI). Instant success.

## Root Cause: LaunchAgent Auto-Restart

The **real** debugging blocker was that `com.hermes.frpc` LaunchAgent kept
respawning frpc, making every `pkill` / `kill` ineffective:

```bash
$ ps aux | grep frpc
xu  1913  ... frpc -c ~/.hermes/frpc.toml

$ kill 1913
$ ps aux | grep frpc
xu  1921  ... frpc -c ~/.hermes/frpc.toml   # IMMEDIATELY respawned!
```

**Discovery**: `launchctl list | grep frp` revealed `com.hermes.frpc` with
KeepAlive=true.

## Correct Workflow

```bash
# 1. Find the LaunchAgent
launchctl list | grep frp

# 2. Unload it (stops auto-restart)
launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist

# 3. Verify fully stopped
ps aux | grep frpc | grep -v grep  # should output nothing

# 4. Edit config, wait 100s for frps to release old sessions
vim ~/.hermes/frpc.toml
sleep 100

# 5. Reload
launchctl load ~/Library/LaunchAgents/com.hermes.frpc.plist

# 6. Check logs
tail -f ~/.hermes/logs/frpc.log  # look for "start proxy success"

# 7. Test
curl http://<VPS_IP>:<PORT>/health
```

## Key Takeaways

1. **Always check for LaunchAgent** before debugging frp on macOS with Hermes
2. **Security group is the #1 cause** of "proxy success but unreachable"
3. **Use ports already open** in security group if you can't modify it
4. **Wait 90-100s** after killing frpc for frps heartbeat timeout
5. **Both proxy name AND remote port** must be unique to avoid conflicts
