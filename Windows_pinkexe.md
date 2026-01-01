# Plink (PuTTY Link) Quick Reference

## What is Plink?

**Plink** = Command-line SSH client for Windows (part of PuTTY package)

**Use Cases**:
- Create SSH tunnels from Windows
- Dynamic port forwarding (SOCKS proxy)
- Pivot through compromised Windows hosts
- "Living off the land" on Windows systems

## Why Use Plink?

✅ **Advantages**:
- Pre-installed on many Windows systems (with PuTTY)
- No admin rights needed
- Living off the land (LOTL)
- Avoids uploading custom tools
- Less likely to trigger alerts

## Installation

### Download Plink
```powershell
# Download from PuTTY website
https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html

# Or use existing PuTTY installation
C:\Program Files\PuTTY\plink.exe
```

### Check if Installed
```cmd
# Check common locations
where plink
dir "C:\Program Files\PuTTY\plink.exe"
dir "C:\Program Files (x86)\PuTTY\plink.exe"

# Search for plink
dir /s C:\plink.exe
```

## Basic Plink Syntax
```cmd
plink [options] [user@]host [command]
```

## Dynamic Port Forwarding (SOCKS Proxy)

### Basic SOCKS Proxy
```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

**Explanation**:
- `-ssh` = Use SSH protocol
- `-D 9050` = Dynamic port forward on port 9050
- `ubuntu@10.129.15.50` = SSH server to connect to

### With Password
```cmd
plink -ssh -D 9050 -pw "password" ubuntu@10.129.15.50
```

### With Key File
```cmd
plink -ssh -D 9050 -i C:\path\to\key.ppk ubuntu@10.129.15.50
```

### Background Mode
```cmd
start /B plink -ssh -D 9050 -N ubuntu@10.129.15.50
```

**Flags**:
- `-N` = No command execution (just tunnel)
- `start /B` = Run in background

## Local Port Forwarding

### Forward Single Port
```cmd
plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.15.50
```

**Explanation**:
- `-L` = Local port forward
- `3389` = Local port to listen on
- `172.16.5.19:3389` = Remote host:port to forward to

### Multiple Port Forwards
```cmd
plink -ssh -L 3389:172.16.5.19:3389 -L 445:172.16.5.19:445 ubuntu@10.129.15.50
```

## Remote Port Forwarding

### Reverse Port Forward
```cmd
plink -ssh -R 8080:127.0.0.1:80 ubuntu@10.129.15.50
```

**Explanation**:
- `-R` = Remote port forward
- `8080` = Remote port to listen on
- `127.0.0.1:80` = Local host:port to forward to

## Common Plink Options

| Option | Description | Example |
|--------|-------------|---------|
| `-ssh` | Use SSH protocol | `plink -ssh user@host` |
| `-D PORT` | Dynamic forward (SOCKS) | `plink -D 9050 user@host` |
| `-L` | Local port forward | `plink -L 3389:target:3389 user@host` |
| `-R` | Remote port forward | `plink -R 8080:127.0.0.1:80 user@host` |
| `-N` | No command execution | `plink -N -D 9050 user@host` |
| `-pw` | Password | `plink -pw pass user@host` |
| `-i` | Key file (.ppk) | `plink -i key.ppk user@host` |
| `-P` | Port (SSH server) | `plink -P 2222 user@host` |
| `-v` | Verbose | `plink -v user@host` |
| `-batch` | Non-interactive | `plink -batch user@host` |

## Using Proxifier with Plink

### Step 1: Start Plink SOCKS Proxy
```cmd
plink -ssh -D 9050 -N ubuntu@10.129.15.50
```

### Step 2: Configure Proxifier

**Download**: https://www.proxifier.com/

**Configuration**:
1. Open Proxifier
2. Profile → Proxy Servers
3. Add new proxy:
   - Address: 127.0.0.1
   - Port: 9050
   - Protocol: SOCKS Version 5
4. Profile → Proxification Rules
5. Add applications (e.g., mstsc.exe for RDP)

### Step 3: Use Proxified Applications
```cmd
# RDP will now route through Plink tunnel
mstsc.exe
```

## Connecting to Internal RDP via Plink

### Complete Workflow
```cmd
# Step 1: Start Plink SOCKS proxy
plink -ssh -D 9050 -N ubuntu@10.129.15.50

# Step 2: Configure Proxifier
# (GUI configuration as above)

# Step 3: Start RDP to internal target
mstsc /v:172.16.5.19
```

## Authentication Methods

### Password Authentication
```cmd
plink -ssh -D 9050 -pw "MyPassword123!" ubuntu@10.129.15.50
```

### Key File Authentication
```cmd
# Use .ppk file (PuTTY format)
plink -ssh -D 9050 -i C:\keys\mykey.ppk ubuntu@10.129.15.50
```

### Convert OpenSSH Key to PPK
```powershell
# Use PuTTYgen
puttygen.exe id_rsa -o mykey.ppk

# Or from command line
puttygen id_rsa -O private -o mykey.ppk
```

### Interactive Password Prompt
```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
# Will prompt for password
```

## Common Use Cases

### Scenario 1: Access Internal RDP
```cmd
# Start tunnel
plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.15.50

# Connect to RDP via localhost
mstsc /v:localhost
```

### Scenario 2: SOCKS Proxy for Multiple Tools
```cmd
# Start SOCKS proxy
plink -ssh -D 9050 ubuntu@10.129.15.50

# Configure Proxifier
# Route: mstsc.exe, browser, etc.
```

### Scenario 3: Access Internal SMB Share
```cmd
# Forward SMB port
plink -ssh -L 445:172.16.5.19:445 ubuntu@10.129.15.50

# Access share
net use \\127.0.0.1\C$ /user:admin password
```

### Scenario 4: Reverse Shell Relay
```cmd
# Reverse port forward
plink -ssh -R 4444:127.0.0.1:4444 ubuntu@10.129.15.50

# Now targets can connect to Ubuntu:4444
# Traffic forwarded to Windows:4444
```

## Plink vs SSH (Linux)

| Feature | Plink (Windows) | SSH (Linux) |
|---------|-----------------|-------------|
| **Dynamic forward** | `plink -D 9050` | `ssh -D 9050` |
| **Local forward** | `plink -L` | `ssh -L` |
| **Remote forward** | `plink -R` | `ssh -R` |
| **Password** | `-pw password` | Interactive or keys |
| **Key file** | `-i key.ppk` | `-i key` |
| **No command** | `-N` | `-N` |

## Troubleshooting

### Issue 1: "Host Key Not Cached"
```cmd
# Accept host key automatically
echo y | plink -ssh ubuntu@10.129.15.50 exit

# Or use -batch mode (accepts automatically)
plink -batch -ssh -D 9050 ubuntu@10.129.15.50
```

### Issue 2: Authentication Failed
```cmd
# Verify credentials
plink -ssh -v ubuntu@10.129.15.50

# Use password explicitly
plink -ssh -pw "password" ubuntu@10.129.15.50

# Check key file
plink -ssh -i C:\path\to\key.ppk ubuntu@10.129.15.50
```

### Issue 3: Port Already in Use
```cmd
# Check what's using port
netstat -ano | findstr :9050

# Use different port
plink -ssh -D 9051 ubuntu@10.129.15.50
```

### Issue 4: Connection Drops
```cmd
# Add keepalive
plink -ssh -D 9050 ubuntu@10.129.15.50 -o ServerAliveInterval=60
```

## Living Off The Land (LOTL)

### Find Plink on System
```cmd
# Search for plink
dir /s C:\plink.exe
dir /s "C:\Program Files\PuTTY\plink.exe"

# Search file shares
dir \\file-server\share\tools\plink.exe
```

### Use Without Installation
```cmd
# Copy from file share
copy \\file-server\share\tools\plink.exe C:\Windows\Temp\

# Run from current directory
.\plink.exe -ssh -D 9050 ubuntu@10.129.15.50
```

### Avoid Detection
```cmd
# Use legitimate paths
copy plink.exe "C:\Program Files\PuTTY\plink.exe"

# Run from temp
copy plink.exe %TEMP%\plink.exe
%TEMP%\plink.exe -ssh -D 9050 ubuntu@10.129.15.50
```

## Alternative: Use Native Windows SSH

### Windows 10/11 (1809+)
```powershell
# Native SSH client
ssh -D 9050 -N ubuntu@10.129.15.50

# Local forward
ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.15.50

# Remote forward
ssh -R 8080:127.0.0.1:80 ubuntu@10.129.15.50
```

## Proxifier Configuration

### Add SOCKS Proxy
```
Profile → Proxy Servers → Add
  Address: 127.0.0.1
  Port: 9050
  Protocol: SOCKS Version 5
  
Profile → Proxification Rules
  Name: RDP via Plink
  Applications: mstsc.exe
  Target hosts: Any
  Action: Proxy SOCKS5 127.0.0.1
```

### Test Configuration
```
Profile → Proxification Rules → Check...
Shows which apps will be proxified
```

## Verification Commands

### Check Plink Running
```cmd
tasklist | findstr plink
netstat -ano | findstr :9050
```

### Test SOCKS Proxy
```powershell
# With curl (if available)
curl --socks5 127.0.0.1:9050 http://172.16.5.19

# With PowerShell (via Proxifier)
Test-NetConnection -ComputerName 172.16.5.19 -Port 3389
```

### Check Port Forward
```cmd
netstat -ano | findstr :3389
telnet localhost 3389
```

## Complete Attack Example
```cmd
# SCENARIO: Access internal RDP server

# Step 1: Check if plink exists
where plink
# Found: C:\Program Files\PuTTY\plink.exe

# Step 2: Accept host key
echo y | plink -ssh ubuntu@10.129.15.50 exit

# Step 3: Start SOCKS proxy
plink -ssh -D 9050 -N -pw "password" ubuntu@10.129.15.50

# Step 4: Configure Proxifier
# (GUI - add SOCKS5 127.0.0.1:9050)

# Step 5: Add rule for mstsc.exe
# Profile → Proxification Rules → Add
# Application: mstsc.exe
# Action: Proxy via SOCKS5

# Step 6: Connect to internal RDP
mstsc /v:172.16.5.19

# SUCCESS: RDP traffic routed through Plink tunnel
```

## Key Points

🔴 **Remember**:
- Plink = PuTTY command-line SSH
- Use `-D` for SOCKS proxy
- Use `-L` for local port forward
- Use `-R` for remote port forward
- Combine with Proxifier for GUI apps
- Great for LOTL on Windows

🎯 **Best Practices**:
- Accept host key first with `echo y | plink ...`
- Use `-batch` for non-interactive
- Use `-N` when only tunneling
- Check for existing PuTTY installation
- Consider native SSH on Win10+

## Quick Commands
```cmd
# SOCKS proxy
plink -ssh -D 9050 -N ubuntu@10.129.15.50

# Local forward
plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.15.50

# Remote forward
plink -ssh -R 8080:127.0.0.1:80 ubuntu@10.129.15.50

# With password
plink -ssh -D 9050 -pw "pass" ubuntu@10.129.15.50

# With key
plink -ssh -D 9050 -i key.ppk ubuntu@10.129.15.50

# Accept host key
echo y | plink -ssh ubuntu@10.129.15.50 exit
