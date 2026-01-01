# Sshuttle Quick Reference

## What is Sshuttle?

**Sshuttle** = Transparent VPN over SSH (no proxychains needed)

**Key Features**:
- ✅ Auto-configures iptables
- ✅ No proxychains required
- ✅ Transparent routing
- ✅ VPN-like experience
- ❌ SSH only (no SOCKS/HTTP/TOR)

## Installation

```bash
# Debian/Ubuntu/Kali
sudo apt-get install sshuttle

# Or via pip
pip install sshuttle
```

## Basic Usage

### Syntax
```bash
sudo sshuttle -r user@pivot_host NETWORK/CIDR
```

### Simple Example
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23
```

### With Verbose
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

### With Password
```bash
# Interactive password prompt
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23

# Or use SSH keys (recommended)
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 --ssh-cmd "ssh -i /path/to/key"
```

## Common Options

| Option | Description | Example |
|--------|-------------|---------|
| `-r` | Remote host | `-r user@host` |
| `-v` | Verbose | `-v` or `-vv` |
| `-x` | Exclude subnet | `-x 172.16.5.5/32` |
| `-l` | Listen address | `-l 0.0.0.0:12300` |
| `--dns` | Forward DNS | `--dns` |
| `-N` | Auto-detect networks | `-N` |
| `--no-latency-control` | Disable latency | `--no-latency-control` |

## Advanced Usage

### Route Multiple Networks
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 10.10.10.0/24
```

### Exclude Specific IPs
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -x 172.16.5.5
```

### Auto-Detect Remote Networks
```bash
sudo sshuttle -r ubuntu@10.129.202.64 -N
```

### Route All Traffic (0.0.0.0/0)
```bash
sudo sshuttle -r ubuntu@10.129.202.64 0.0.0.0/0
```

### With DNS Forwarding
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 --dns
```

## Using Tools with Sshuttle

### No Proxychains Needed!

#### Nmap
```bash
# Start sshuttle
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23

# In another terminal - use nmap directly
nmap -sT -p3389 172.16.5.19 -Pn
```

#### RDP
```bash
xfreerdp /v:172.16.5.19 /u:admin /p:pass
```

#### SMB
```bash
smbclient //172.16.5.19/C$ -U admin
```

#### Any Tool
```bash
# ALL tools work transparently
curl http://172.16.5.19
ssh user@172.16.5.19
nxc smb 172.16.5.19
```

## Complete Workflow

### Step 1: Start Sshuttle
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

### Step 2: Open New Terminal
```bash
# Keep sshuttle running in terminal 1
# Open terminal 2 for attacks
```

### Step 3: Use Tools Directly
```bash
# No proxychains needed!
nmap -sV 172.16.5.19
xfreerdp /v:172.16.5.19 /u:admin
```

### Step 4: Stop Sshuttle
```bash
# In sshuttle terminal: CTRL+C
# Automatically cleans up iptables rules
```

## How Sshuttle Works

### What Happens
```
1. Sshuttle connects via SSH to pivot
2. Creates iptables NAT rules on attack host
3. Redirects traffic for specified networks
4. Forwards through SSH tunnel
5. Traffic appears to come from pivot
```

### iptables Rules Created
```bash
# View rules while sshuttle running
sudo iptables -t nat -L -n -v

# Example output:
# Chain sshuttle-12300
# REDIRECT tcp -- * * 0.0.0.0/0 172.16.5.0/23 redir ports 12300
```

## Sshuttle vs Proxychains

| Feature | Sshuttle | Proxychains |
|---------|----------|-------------|
| **Setup** | Single command | Config + prefix |
| **Transparency** | ✅ Full | ❌ Per-app |
| **iptables** | ✅ Auto | ❌ Manual |
| **DNS** | ✅ Optional | ✅ Yes |
| **Protocols** | SSH only | SOCKS/HTTP/TOR |
| **Nmap -sS** | ❌ No | ❌ No |
| **Ease of Use** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

## Advantages vs SSH + Proxychains

### Sshuttle ✅
```bash
# One command
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23

# Use tools normally
nmap 172.16.5.19
```

### SSH + Proxychains ❌
```bash
# Multiple steps
ssh -D 9050 ubuntu@pivot
nano /etc/proxychains.conf
proxychains nmap 172.16.5.19
```

## Common Use Cases

### Pentest Internal Network
```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23
```

### Access Multiple Subnets
```bash
sudo sshuttle -r ubuntu@pivot 10.10.10.0/24 172.16.0.0/16
```

### Full VPN Experience
```bash
sudo sshuttle -r ubuntu@pivot 0.0.0.0/0 --dns
```

## Troubleshooting

### Issue 1: Permission Denied
```bash
# Must run as root/sudo
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23
```

### Issue 2: iptables Errors
```bash
# Check iptables available
sudo iptables -L

# Install if missing
sudo apt install iptables
```

### Issue 3: Can't Connect
```bash
# Test SSH first
ssh ubuntu@10.129.202.64

# Use verbose mode
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 -vv
```

### Issue 4: Cleanup Failed
```bash
# Manual cleanup if needed
sudo iptables -t nat -F sshuttle-12300
sudo iptables -t nat -X sshuttle-12300
```

### Issue 5: Traffic Not Routing
```bash
# Check routing
ip route
sudo iptables -t nat -L -n -v

# Restart sshuttle
CTRL+C
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23
```

## Running in Background

### Using screen
```bash
screen -dmS sshuttle sudo sshuttle -r ubuntu@pivot 172.16.5.0/23
```

### Using tmux
```bash
tmux new -d -s sshuttle 'sudo sshuttle -r ubuntu@pivot 172.16.5.0/23'
```

### Using nohup
```bash
sudo nohup sshuttle -r ubuntu@pivot 172.16.5.0/23 &
```

## Authentication

### SSH Key
```bash
# Recommended
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 --ssh-cmd "ssh -i ~/.ssh/id_rsa"
```

### SSH Config
```bash
# Use ~/.ssh/config
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23
```

### Password
```bash
# Interactive prompt
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23
```

## Verification

### Check Sshuttle Running
```bash
ps aux | grep sshuttle
```

### Check iptables Rules
```bash
sudo iptables -t nat -L -n -v | grep sshuttle
```

### Test Connectivity
```bash
ping 172.16.5.19
curl http://172.16.5.19
```

## Limitations

❌ **Cannot do**:
- SYN scans (`nmap -sS`) - use `-sT` instead
- UDP scans - TCP only
- Raw sockets - layer 3
- Non-SSH pivoting

✅ **Can do**:
- TCP connect scans (`nmap -sT`)
- Any TCP-based tool
- DNS forwarding (with `--dns`)
- Multiple networks

## Quick Commands

```bash
# Install
sudo apt install sshuttle

# Basic usage
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23

# With verbose
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 -v

# Multiple networks
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 10.10.10.0/24

# Auto-detect
sudo sshuttle -r ubuntu@pivot -N

# With DNS
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 --dns

# Exclude subnet
sudo sshuttle -r ubuntu@pivot 172.16.5.0/23 -x 172.16.5.5

# Stop
CTRL+C
```

## Key Points

🔴 **Remember**:
- Must run with sudo
- SSH only (no SOCKS/HTTP)
- Automatically configures iptables
- No proxychains needed
- Must use `-sT` for nmap (no SYN scan)
- CTRL+C cleans up automatically

🎯 **Best For**:
- Quick pivoting over SSH
- Transparent tool usage
- When you have SSH access
- Simpler than proxychains
