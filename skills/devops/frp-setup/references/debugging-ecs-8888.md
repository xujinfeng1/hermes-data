# frp Debugging Session Reference

## Scenario
Adding a new proxy to expose CDP Agent (localhost:8800) on a new remote port (8888)
via existing frp tunnel to Alibaba Cloud ECS (8.163.113.208).

## Timeline

### Phase 1: "proxy already exists" loop
```
Action: Add [[proxies]] block for cdp-agent-demo → remotePort 8888
Action: kill frpc, restart
Log:    proxy added: [cdp-agent-demo hermes-webui-80 hermes-webui-8080]
Log:    [cdp-agent-demo] start error: proxy [cdp-agent-demo] already exists
Result: Deadlock — all 3 proxies fail with "already exists"

Root cause: Old frpc session still registered on frps (heartbeat timeout = 90s)
Attempted fix: Rename proxies (cdp-agent-demo → cdp-agent-demo-v2)
Result: Same error — old parent frpc process was STILL RUNNING
```

### Phase 2: Discovering LaunchAgent auto-restart
```
Symptom:    killall -9 frpc → new frpc PID appears instantly
Diagnosis:  ps aux shows frpc running as root; respawning detected
Discovery:  launchctl list | grep frp → com.hermes.frpc (KeepAlive=true)
File:       ~/Library/LaunchAgents/com.hermes.frpc.plist

Fix: launchctl unload ~/Library/LaunchAgents/com.hermes.frpc.plist
     → frpc finally stays dead
```

### Phase 3: Waiting for frps session release
```
Action:     Unload LaunchAgent, wait 60s
Result:     New proxy names work — "start proxy success"
But:        curl http://8.163.113.208:8888/health → timeout

Diagnosis:  frpc log shows "start proxy success" with no errors
            → NOT a frp issue → security group blocking port 8888

Quick test: Port 8080 was known-working (Hermes WebUI).
            Port 8888 was never opened in security group.
```

### Phase 4: Workaround — use known-open port
```
Action:     Change remotePort from 8888 → 8080 (already in security group)
            Hermes WebUI on localhost:9119 was down anyway
            Updated frpc config to single proxy: cdp-demo → 8080
Result:     curl http://8.163.113.208:8080/health → {"status":"ok"}
```

### Final state
```
Config:     serverAddr = "8.163.113.208", serverPort = 7000
            [[proxies]] name = "cdp-demo", localPort = 8800, remotePort = 8080
Management: LaunchAgent com.hermes.frpc (auto-restart)
            LaunchAgent com.cdp.agent (uvicorn on 8800)
External:   http://8.163.113.208:8080/health ✅
```

## Key lessons

1. **Always check for LaunchAgent** before killing frpc on macOS Hermes installs
2. **"start proxy success" + timeout = security group**, not frp. Don't debug frp.
3. **"proxy already exists" = old session on frps**. Wait 90s or use unique names.
4. **Port isolation test**: test known-working port vs new port to narrow down cause
5. **Reuse known-open ports** as a quick workaround when you can't access the cloud console
