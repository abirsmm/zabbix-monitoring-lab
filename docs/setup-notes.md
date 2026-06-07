# 🔧 Setup Notes & Troubleshooting

Detailed notes from building the Zabbix 7.0 monitoring lab from scratch on VirtualBox.

---

## Table of Contents

- [Environment Details](#environment-details)
- [Step-by-Step Build Log](#step-by-step-build-log)
- [Issues Encountered & Fixes](#issues-encountered--fixes)
- [Firewall Rules](#firewall-rules)
- [Database Reference](#database-reference)
- [Useful Zabbix Commands](#useful-zabbix-commands)
- [Item Keys Reference](#item-keys-reference)
- [Trigger Expressions Reference](#trigger-expressions-reference)
- [Email Alert Configuration](#email-alert-configuration)

---

## Environment Details

| Setting | Value |
|---|---|
| Host OS | Windows 10/11 |
| Host RAM | 16 GB |
| Hypervisor | VirtualBox |
| Network Mode | Bridged Adapter |
| Ubuntu Version | 22.04 LTS |
| Zabbix Version | 7.0 LTS |
| MySQL Version | 8.0 |
| Apache Version | 2.4 |
| Zabbix Agent (Windows) | 7.4.11 |
| Zabbix Server IP | 192.168.18.4 |
| Windows Client IP | 192.168.18.9 |
| Zabbix Web UI | http://192.168.18.4/zabbix |
| Default Login | Admin / zabbix |

---

## Step-by-Step Build Log

### Phase 1: Ubuntu VM Setup

- Installed Ubuntu Server 22.04 LTS inside VirtualBox
- Initially set to NAT network (got IP 10.0.2.15 — not reachable from host browser)
- Fixed by switching VirtualBox Adapter 1 to **Bridged Adapter**
- After reboot got IP **192.168.18.4** on local network

### Phase 2: Zabbix Installation

```bash
# Add Zabbix 7.0 repo for Ubuntu 22.04
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
sudo apt update

# Install server + frontend + agent
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y
```

### Phase 3: MySQL Database Setup

```bash
sudo apt install mysql-server -y
sudo mysql
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Zabbix@123';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
SET GLOBAL log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
QUIT;
```

```bash
# Import schema (takes 1-2 minutes, no output is normal)
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

### Phase 4: Configure Zabbix Server

```bash
# Set DB password in config
sudo nano /etc/zabbix/zabbix_server.conf
# Find line: # DBPassword=
# Change to: DBPassword=Zabbix@123

# Start and enable all services
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

### Phase 5: Web UI Setup Wizard

- Opened browser on Windows host: `http://192.168.18.4/zabbix`
- Completed setup wizard with DB credentials
- Logged in with `Admin` / `zabbix`
- Changed default password immediately after login

### Phase 6: Windows Agent Installation

- Downloaded Zabbix Agent 7.4.11 MSI from zabbix.com/download_agents
- Installed with:
  - Host name: `Windows-Client`
  - Zabbix server IP: `192.168.18.4`
  - Port: `10050`
- Added Windows Firewall inbound rule for TCP 10050

```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -Action Allow
```

### Phase 7: Adding Hosts in Zabbix UI

**Ubuntu-Server:**
- Template: `Linux by Zabbix agent`
- Group: `Linux servers`
- Interface: `127.0.0.1:10050`

**Windows-Client:**
- Template: `Windows by Zabbix agent`
- Group: `Windows servers`
- Interface: `192.168.18.9:10050`

### Phase 8: Custom Items Created

Some items were not included in default templates and had to be created manually:

| Item | Host | Type | Key |
|---|---|---|---|
| Disk space used percent | Ubuntu-Server | Zabbix agent | `vfs.fs.size[/,pused]` |
| ICMP ping | Ubuntu-Server | Simple check | `icmpping` |
| ICMP ping | Windows-Client | Simple check | `icmpping` |

### Phase 9: Triggers Created

| Trigger | Expression | Severity |
|---|---|---|
| Disk > 80% (Ubuntu) | `last(/Ubuntu-Server/vfs.fs.size[/,pused])>80` | Warning |
| Host unreachable (Ubuntu) | `max(/Ubuntu-Server/icmpping,3m)=0` | High |
| Host unreachable (Windows) | `max(/Windows-Client/icmpping,3m)=0` | High |

### Phase 10: Email Alerts

- Gmail App Password generated (requires 2FA enabled)
- SMTP configured in Zabbix: `smtp.gmail.com:587` STARTTLS
- Media type assigned to Admin user
- Trigger action created: notify on Warning+ severity
- Recovery operation added: notify when problem resolves
- Test email confirmed delivered to inbox

---

## Issues Encountered & Fixes

### Issue 1: Ubuntu IP was 10.0.2.15 (NAT)

**Problem:** Browser on Windows host could not reach `http://10.0.2.15/zabbix`

**Cause:** VirtualBox default network is NAT — internal to VM only

**Fix:**
1. Shut down Ubuntu VM
2. VirtualBox → Settings → Network → Adapter 1
3. Change "Attached to" from `NAT` to `Bridged Adapter`
4. Select your physical network card under Name
5. Start VM → run `ip a` → new IP was `192.168.18.4`

---

### Issue 2: Zabbix login blocked ("account temporarily blocked")

**Problem:** After a few failed login attempts, account was locked

**Fix:** Reset via MySQL:
```bash
sudo mysql -uzabbix -p
```
```sql
USE zabbix;
UPDATE users SET attempt_failed=0, attempt_clock=0 WHERE alias='Admin';
QUIT;
```
Then logged in with: `Admin` / `zabbix` (capital A, lowercase password)

---

### Issue 3: Trigger error — incorrect item key

**Problem:**
```
Cannot add trigger
* Incorrect item key "vfs.fs.size[/,pused]" provided for trigger expression on "Ubuntu-Server"
```

**Cause:** The Linux template did not include a filesystem usage item by default

**Fix:** Manually created the item first:
- Configuration → Hosts → Ubuntu-Server → Items → Create item
- Key: `vfs.fs.size[/,pused]`, Type: Zabbix agent, Info type: Numeric (float)
- Waited 1 min for data to collect, then created the trigger successfully

---

### Issue 4: icmpping trigger error

**Problem:**
```
Cannot add trigger
* Incorrect item key "icmpping" provided for trigger expression
```

**Cause:** ICMP ping item was not created yet for either host

**Fix:** Created Simple check items manually on both hosts:
- Type: `Simple check`
- Key: `icmpping`
- Type of information: `Numeric (unsigned)`

Note: Simple check items run from the Zabbix server directly — no agent needed on target

---

### Issue 5: Gmail SMTP authentication failed

**Problem:** Regular Gmail password rejected by Zabbix SMTP

**Fix:**
1. Enabled 2-Step Verification on Google account
2. Generated App Password at myaccount.google.com → Security → App passwords
3. Used the 16-character App Password in Zabbix media type settings

---

## Firewall Rules

### Ubuntu Server (ufw)

```bash
# Allow Zabbix agent port
sudo ufw allow 10050/tcp

# Allow Apache web UI
sudo ufw allow 80/tcp

# Allow SNMP (if needed later)
sudo ufw allow 161/udp
```

### Windows Client (PowerShell as Admin)

```powershell
# Allow Zabbix agent inbound
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -Action Allow
```

---

## Database Reference

```bash
# Connect to Zabbix database
sudo mysql -uzabbix -p
USE zabbix;

# Check user table
SELECT userid, alias, attempt_failed FROM users;

# Reset locked account
UPDATE users SET attempt_failed=0, attempt_clock=0 WHERE alias='Admin';

# Exit
QUIT;
```

---

## Useful Zabbix Commands

```bash
# Check Zabbix server status
sudo systemctl status zabbix-server

# Check Zabbix server logs
sudo tail -f /var/log/zabbix/zabbix_server.log

# Check agent logs
sudo tail -f /var/log/zabbix/zabbix_agentd.log

# Restart all Zabbix services
sudo systemctl restart zabbix-server zabbix-agent apache2

# Test agent connectivity from server
zabbix_get -s 127.0.0.1 -p 10050 -k system.uptime
zabbix_get -s 192.168.18.9 -p 10050 -k system.uptime
```

---

## Item Keys Reference

| Key | Description | Type |
|---|---|---|
| `system.cpu.util` | CPU utilization % | Zabbix agent |
| `vm.memory.size[available]` | Available RAM in bytes | Zabbix agent |
| `vm.memory.size[total]` | Total RAM in bytes | Zabbix agent |
| `vfs.fs.size[/,pused]` | Disk used % (Linux root) | Zabbix agent |
| `vfs.fs.size[C:,pused]` | Disk used % (Windows C:) | Zabbix agent |
| `icmpping` | ICMP ping (1=up, 0=down) | Simple check |
| `icmppingloss` | ICMP packet loss % | Simple check |
| `icmppingsec` | ICMP response time (sec) | Simple check |
| `system.uptime` | System uptime in seconds | Zabbix agent |
| `net.if.in[eth0]` | Network in bytes/sec | Zabbix agent |
| `net.if.out[eth0]` | Network out bytes/sec | Zabbix agent |

---

## Trigger Expressions Reference

```
# Disk space above 80% - Ubuntu
last(/Ubuntu-Server/vfs.fs.size[/,pused])>80

# Disk space above 80% - Windows
last(/Windows-Client/vfs.fs.size[C:,pused])>80

# Host unreachable for 3 minutes - Ubuntu
max(/Ubuntu-Server/icmpping,3m)=0

# Host unreachable for 3 minutes - Windows
max(/Windows-Client/icmpping,3m)=0

# CPU above 90% for 5 minutes (future use)
avg(/Ubuntu-Server/system.cpu.util,5m)>90

# Low memory below 10% (future use)
last(/Ubuntu-Server/vm.memory.size[pavailable])<10
```

---

## Email Alert Configuration

| Setting | Value |
|---|---|
| SMTP Server | smtp.gmail.com |
| SMTP Port | 587 |
| Connection Security | STARTTLS |
| Authentication | Username and password |
| Username | your.email@gmail.com |
| Password | 16-character App Password |
| From | your.email@gmail.com |

**Trigger Action Settings:**
- Name: `Email alert on problem`
- Condition: Trigger severity >= Warning
- Operation: Send message to Admin via Email
- Recovery: Send recovery message when resolved

---

*Last updated during home lab build — Zabbix 7.0 on Ubuntu 22.04 LTS*