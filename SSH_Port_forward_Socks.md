# SSH Port Forwarding & SOCKS Tunneling Quick Reference

## Port Forwarding Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Local (-L)** | Forward local port to remote | Access remote service locally |
| **Dynamic (-D)** | SOCKS proxy tunnel | Pivot to entire network |
| **Remote (-R)** | Forward remote port to local | Reverse tunnel |

## SSH Local Port Forwarding (-L)

### Basic Syntax
```bash
ssh -L LOCAL_PORT:TARGET_HOST:TARGET_PORT user@SSH_SERVER
```

### Single Port Forward
```bash
# Forward local port 1234 to remote MySQL (3306)
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64

# Access MySQL locally
mysql -u root -p -h 127.0.0.1 -P 1234
```

### Multiple Ports
```bash
# Forward MySQL and Apache
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

### Forward to Different Host
```bash
# Forward to another server in network
ssh -L 1234:172.16.5.19:3389 ubuntu@10.129.202.64
```

## Verify Port Forward

### Using netstat
```bash
netstat -antp | grep 1234

# Output:
tcp   0   0 127.0.0.1:1234   0.0.0.0:*   LISTEN   4034/ssh
```

### Using nmap
```bash
nmap -sV -p1234 localhost
```

### Using lsof
```bash
lsof -i :1234
```

## SSH Dynamic Port Forwarding (-D)

### Enable SOCKS Proxy
```bash
# Create SOCKS proxy on port 9050
ssh -D 9050 ubuntu@10.129.202.64

# Keep session alive
ssh -D 9050 -N ubuntu@10.129.202.64
```

**Flags**:
- `-D 9050` = Dynamic port forwarding (SOCKS proxy)
- `-N` = No command execution (just tunnel)
- `-f` = Background process

### With Background
```bash
# Run in background
ssh -D 9050 -N -f ubuntu@10.129.202.64

# Kill later
ps aux | grep "ssh -D"
kill PID
```

## Configure Proxychains

### Edit /etc/proxychains.conf
```bash
sudo nano /etc/proxychains.conf

# Add/modify at bottom:
socks4  127.0.0.1 9050

# Or for SOCKS5:
socks5  127.0.0.1 9050
```

### Verify Configuration
```bash
tail -4 /etc/proxychains.conf

# Output:
# defaults set to "tor"
socks4  127.0.0.1 9050
```

## Using Proxychains

### Basic Syntax
```bash
proxychains COMMAND
```

### Nmap through Proxy
```bash
# Ping scan (host discovery)
proxychains nmap -sn 172.16.5.1-200

# Full TCP connect scan (required for proxychains)
proxychains nmap -sT -Pn 172.16.5.19

# Service scan
proxychains nmap -sT -Pn -sV -p445,3389,135,139 172.16.5.19
```

**Important Nmap Options**:
- `-sT` = Full TCP connect (required, no SYN scan)
- `-Pn` = Skip ping (Windows blocks ICMP)
- `-sV` = Version detection

### Other Tools
```bash
# cURL
proxychains curl http://172.16.5.19

# Metasploit
proxychains msfconsole

# xfreerdp
proxychains xfreerdp /v:172.16.5.19 /u:user /p:pass

# SSH
proxychains ssh user@172.16.5.19

# NetExec
proxychains nxc smb 172.16.5.19 -u user -p pass
```

## Metasploit with Proxychains

### Start Metasploit
```bash
proxychains msfconsole
```

### Use Auxiliary Modules
```bash
# Search for RDP scanner
search rdp_scanner

# Use module
use auxiliary/scanner/rdp/rdp_scanner

# Set target
set RHOSTS 172.16.5.19

# Run
run
```

### Other Useful Modules
```bash
# SMB version
use auxiliary/scanner/smb/smb_version
set RHOSTS 172.16.5.0/24
run

# HTTP title
use auxiliary/scanner/http/title
set RHOSTS 172.16.5.19
run
```

## RDP through SOCKS Tunnel

```bash
# Basic RDP
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:'pass@123'

# With options
proxychains xfreerdp /v:172.16.5.19 \
                     /u:victor \
                     /p:'pass@123' \
                     /cert:ignore \
                     /dynamic-resolution

# Accept certificate when prompted
```

## Complete Attack Workflow

### 1. Compromise Initial Target
```bash
# SSH into pivot host
ssh ubuntu@10.129.202.64
```

### 2. Enumerate Networks
```bash
# Check interfaces
ifconfig
ip a

# Note internal networks
# Example: ens224: 172.16.5.129
```

### 3. Setup Dynamic Port Forward
```bash
# Exit from SSH session
exit

# Create SOCKS tunnel
ssh -D 9050 -N -f ubuntu@10.129.202.64
```

### 4. Configure Proxychains
```bash
# Edit config
sudo nano /etc/proxychains.conf

# Add:
socks4  127.0.0.1 9050
```

### 5. Scan Internal Network
```bash
# Host discovery
proxychains nmap -sn 172.16.5.1-50

# Port scan discovered hosts
proxychains nmap -sT -Pn -p- 172.16.5.19
```

### 6. Access Internal Services
```bash
# RDP
proxychains xfreerdp /v:172.16.5.19 /u:admin /p:pass

# SMB
proxychains smbclient //172.16.5.19/share -U user

# HTTP
proxychains curl http://172.16.5.19
```

## Network Topology Example

```
Attack Host (10.10.15.5)
    ↓
SSH Tunnel (-D 9050)
    ↓
Pivot Host (10.129.202.64 + 172.16.5.129)
    ↓
Internal Network (172.16.5.0/23)
    ↓
Target (172.16.5.19)
```

## Common Issues & Solutions

### Issue 1: Nmap Not Working
```bash
# Must use -sT (full TCP connect)
proxychains nmap -sT -Pn 172.16.5.19

# Don't use: -sS (SYN scan won't work)
```

### Issue 2: Proxychains Not Configured
```bash
# Check config
tail /etc/proxychains.conf

# Should have:
socks4  127.0.0.1 9050
```

### Issue 3: SSH Tunnel Died
```bash
# Check if running
ps aux | grep "ssh -D"

# Restart
ssh -D 9050 -N -f ubuntu@10.129.202.64
```

### Issue 4: DNS Resolution Fails
```bash
# Use IP addresses instead of hostnames
proxychains nmap -sT 172.16.5.19

# Or enable proxy_dns in proxychains.conf
```

## Port Forwarding Cheat Sheet

### Local Forward (-L)
```bash
# Syntax
ssh -L [local_port]:[target_host]:[target_port] user@pivot_host

# Example
ssh -L 3306:localhost:3306 user@10.129.202.64
```

### Dynamic Forward (-D)
```bash
# Syntax
ssh -D [local_port] user@pivot_host

# Example
ssh -D 9050 user@10.129.202.64
```

### Remote Forward (-R)
```bash
# Syntax
ssh -R [remote_port]:[local_host]:[local_port] user@pivot_host

# Example
ssh -R 8080:localhost:80 user@10.129.202.64
```

## Proxychains Configuration Options

### /etc/proxychains.conf
```bash
# Proxy type
socks4  127.0.0.1 9050  # SOCKS4
socks5  127.0.0.1 9050  # SOCKS5
http    127.0.0.1 8080  # HTTP

# Proxy DNS
proxy_dns           # Enable DNS proxying

# Chain type
#dynamic_chain      # Each proxy in order
#strict_chain       # All proxies must work
random_chain        # Random order

# Timeout
tcp_read_time_out 15000
tcp_connect_time_out 8000
```

## Quick Commands Reference

| Task | Command |
|------|---------|
| **Local forward** | `ssh -L 1234:localhost:3306 user@host` |
| **Dynamic forward** | `ssh -D 9050 user@host` |
| **Background tunnel** | `ssh -D 9050 -N -f user@host` |
| **Edit proxychains** | `sudo nano /etc/proxychains.conf` |
| **Nmap via proxy** | `proxychains nmap -sT -Pn target` |
| **RDP via proxy** | `proxychains xfreerdp /v:target /u:user` |
| **Metasploit via proxy** | `proxychains msfconsole` |
| **Check tunnel** | `ps aux \| grep "ssh -D"` |

## SOCKS vs HTTP Proxy

| Feature | SOCKS4 | SOCKS5 | HTTP |
|---------|--------|--------|------|
| **Authentication** | ❌ No | ✅ Yes | ✅ Yes |
| **UDP Support** | ❌ No | ✅ Yes | ❌ No |
| **DNS Proxying** | ❌ No | ✅ Yes | ❌ No |
| **Speed** | Fast | Fast | Medium |

## Key Points

🔴 **Remember**:
- Use `-sT` (full TCP) with proxychains + nmap
- Use `-Pn` to skip ping (Windows blocks ICMP)
- SOCKS tunnel = `-D` flag
- Local forward = `-L` flag
- Edit `/etc/proxychains.conf` before use

🎯 **Best Practices**:
- Use `-N -f` to background SSH tunnel
- Always verify with `netstat` or `ps`
- Use IP addresses (not hostnames) when possible
- Test connectivity before full scans

## Troubleshooting

```bash
# Check SSH tunnel
ps aux | grep "ssh -D"

# Check listening ports
netstat -tlnp | grep 9050

# Test proxy
proxychains curl http://ifconfig.me

# Kill hung tunnel
killall -9 ssh

# Restart
ssh -D 9050 -N -f user@pivot_host
```

