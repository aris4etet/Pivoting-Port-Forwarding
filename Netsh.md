# Netsh Port Forwarding - Quick Workflow

## What is Netsh?

**Netsh** = Windows built-in tool for network configuration & port forwarding

## Quick Setup

### Create Port Forward
```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
```

**Translation**: 
- Listen on `10.129.15.150:8080` (Windows host)
- Forward to `172.16.5.25:3389` (internal RDP)

### Verify
```cmd
netsh.exe interface portproxy show v4tov4
```

### Connect
```bash
# From attack host
xfreerdp /v:10.129.15.150:8080 /u:admin /p:pass
```

## Traffic Flow
```
Attack Host
    ↓ Connect to
Windows Pivot (10.129.15.150:8080)
    ↓ Forwards to
Internal Target (172.16.5.25:3389)
```

## Complete Example
```cmd
# 1. Create forward (on Windows pivot)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25

# 2. Check (on Windows pivot)
netsh interface portproxy show v4tov4

# 3. Connect (from attack host)
xfreerdp /v:10.129.15.150:8080 /u:admin /p:pass
```

## Common Forwards

```cmd
# RDP
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=PIVOT_IP connectport=3389 connectaddress=TARGET_IP

# SMB
netsh interface portproxy add v4tov4 listenport=4445 listenaddress=PIVOT_IP connectport=445 connectaddress=TARGET_IP

# HTTP
netsh interface portproxy add v4tov4 listenport=8000 listenaddress=PIVOT_IP connectport=80 connectaddress=TARGET_IP
```

## Management Commands

```cmd
# List all forwards
netsh interface portproxy show all

# Delete specific forward
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150

# Delete all forwards
netsh interface portproxy reset
```

## Firewall Rule (if needed)

```cmd
# Allow inbound on 8080
netsh advfirewall firewall add rule name="Port Forward 8080" protocol=TCP dir=in localport=8080 action=allow
```

## Quick Reference

| Task | Command |
|------|---------|
| **Add forward** | `netsh interface portproxy add v4tov4 listenport=LP listenaddress=LA connectport=CP connectaddress=CA` |
| **Show forwards** | `netsh interface portproxy show v4tov4` |
| **Delete forward** | `netsh interface portproxy delete v4tov4 listenport=LP listenaddress=LA` |
| **Delete all** | `netsh interface portproxy reset` |

## Key Points

🔴 **Remember**:
- Built-in Windows tool (no install)
- Requires admin rights
- Persistent (survives reboot)
- May need firewall rule

🎯 **Use When**:
- On compromised Windows host
- Need simple port forward
- Living off the land (LOTL)
