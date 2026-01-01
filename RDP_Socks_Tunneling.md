# SocksOverRDP - Quick Workflow

## What is SocksOverRDP?

**SocksOverRDP** = SOCKS proxy over RDP using Dynamic Virtual Channels (DVC)

**Use When**:
- ✅ Windows-only network
- ✅ SSH not available
- ✅ Have RDP access
- ✅ Need to pivot through RDP

## Quick Setup

### 1. Download Tools (Attack Host)
```bash
# SocksOverRDP
https://github.com/nccgroup/SocksOverRDP/releases

# Proxifier Portable
https://www.proxifier.com/download/
# Look for ProxifierPE.zip
```

### 2. Transfer to First Target (10.129.x.x)
```bash
# Via RDP shared folder
xfreerdp /v:TARGET /u:user /p:pass /drive:share,/path/to/files

# Or via SMB/HTTP
```

### 3. Load DLL on First Target
```cmd
# Navigate to SocksOverRDP folder
cd C:\Users\htb-student\Desktop\SocksOverRDP-x64

# Register DLL
regsvr32.exe SocksOverRDP-Plugin.dll

# Should show: "DllRegisterServer succeeded"
```

### 4. RDP to Second Target
```cmd
# From first target, RDP to internal host
mstsc.exe

# Connect to: 172.16.5.19
# Credentials: victor / pass@123

# Prompt shows: "SocksOverRDP plugin enabled, listening on 127.0.0.1:1080"
```

### 5. Start Server on Second Target
```cmd
# Transfer SocksOverRDP-Server.exe to 172.16.5.19
# Run as Administrator
SocksOverRDP-Server.exe
```

### 6. Verify Listener (First Target)
```cmd
# Check SOCKS listener
netstat -antb | findstr 1080

# Should show:
# TCP    127.0.0.1:1080    0.0.0.0:0    LISTENING
```

### 7. Configure Proxifier (First Target)
```
1. Extract ProxifierPE.zip
2. Run Proxifier.exe
3. Profile → Proxy Servers → Add
   - Address: 127.0.0.1
   - Port: 1080
   - Protocol: SOCKS Version 5
4. Profile → Proxification Rules
   - Applications: mstsc.exe (or Any)
   - Action: Proxy SOCKS5 127.0.0.1
```

### 8. Access Third Target
```cmd
# From first target, use mstsc.exe
# Proxifier routes through 127.0.0.1:1080
# Traffic flows: Target1 → RDP → Target2 → Target3

mstsc.exe
# Connect to: 172.16.6.155
```

## Network Flow

```
Attack Host
    ↓ RDP
First Target (10.129.x.x)
    ↓ Load SocksOverRDP-Plugin.dll
    ↓ RDP to
Second Target (172.16.5.19)
    ↓ Run SocksOverRDP-Server.exe
    ↓ SOCKS listener on 127.0.0.1:1080
    ↓ Proxifier routes traffic
Third Target (172.16.6.155)
```

## Complete Example

```cmd
# FIRST TARGET (10.129.x.x)
# 1. Register plugin
regsvr32.exe SocksOverRDP-Plugin.dll

# 2. RDP to second target
mstsc.exe
# Connect: 172.16.5.19
# User: victor
# Pass: pass@123

# SECOND TARGET (172.16.5.19)
# 3. Run server (as Admin)
SocksOverRDP-Server.exe

# FIRST TARGET (verify)
# 4. Check listener
netstat -antb | findstr 1080

# 5. Configure Proxifier
# GUI: Add SOCKS5 127.0.0.1:1080

# 6. Use mstsc.exe to connect to third target
mstsc.exe
# Connect: 172.16.6.155
# Traffic auto-routed through Proxifier
```

## Proxifier Configuration

### Add Proxy Server
```
Profile → Proxy Servers → Add
    Address: 127.0.0.1
    Port: 1080
    Protocol: SOCKS Version 5
```

### Add Proxification Rule
```
Profile → Proxification Rules → Add
    Name: RDP via SocksOverRDP
    Applications: mstsc.exe
    Target hosts: Any
    Action: Proxy SOCKS5 127.0.0.1
```

### Test
```
Profile → Proxification Rules → Check
```

## RDP Performance Tip

```
# If RDP slow:
mstsc.exe → Show Options → Experience tab
Set Performance to: Modem

# Reduces bandwidth usage
```

## Key Files

```
SocksOverRDP-Plugin.dll  # Load on first target
SocksOverRDP-Server.exe  # Run on second target
Proxifier.exe            # Configure on first target
```

## Troubleshooting

### Issue 1: DLL Registration Failed
```cmd
# Run as Administrator
regsvr32.exe SocksOverRDP-Plugin.dll

# Unregister first if needed
regsvr32.exe /u SocksOverRDP-Plugin.dll
regsvr32.exe SocksOverRDP-Plugin.dll
```

### Issue 2: No Listener on 1080
```cmd
# Check listener
netstat -antb | findstr 1080

# Restart SocksOverRDP-Server.exe on second target
# Re-register DLL on first target
```

### Issue 3: Proxifier Not Working
```
1. Check proxy server added: 127.0.0.1:1080
2. Check rule created for mstsc.exe
3. Enable Proxifier (Profile → Enable)
4. Check logs (Profile → View → Proxification → Logs)
```

### Issue 4: Connection Refused
```cmd
# Verify server running on second target
tasklist | findstr SocksOverRDP-Server

# Check firewall
netsh advfirewall show allprofiles
```

## Key Points

🔴 **Remember**:
- Plugin on first target (regsvr32)
- Server on second target (as Admin)
- Proxifier on first target
- Listener is 127.0.0.1:1080
- Must use RDP (mstsc.exe)

🎯 **Setup Order**:
1. Register DLL (first target)
2. RDP to second target
3. Run server (second target)
4. Configure Proxifier (first target)
5. Use mstsc.exe (auto-proxied)

## Quick Reference

| Component | Location | Purpose |
|-----------|----------|---------|
| **SocksOverRDP-Plugin.dll** | First target | Creates SOCKS client |
| **SocksOverRDP-Server.exe** | Second target | SOCKS server |
| **Proxifier** | First target | Routes traffic |
| **Listener** | 127.0.0.1:1080 | SOCKS proxy |

Windows-only pivoting! 🪟🔄
