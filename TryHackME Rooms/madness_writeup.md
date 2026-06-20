# CTF Write-up: Madness — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Madness |
| **Difficulty** | Easy |
| **Tags**       | `Steganography` `Hex Editing` `ROT13` `GNU Screen SUID` |

---

## Introduction

Madness earns its name. On paper it's rated Easy, but the puzzle work here — particularly the layered steganography chain — will make you question your life choices. The hacking techniques themselves aren't complex, but the way the clues are hidden is genuinely devious. Room creator Optional clearly had fun designing this one. Once everything clicks though, it's enormously satisfying.

---

## Step 1 — Reconnaissance (Nmap)

```bash
nmap -A <target-ip>
```

Two ports open:

- **Port 22** — SSH
- **Port 80** — HTTP

Port 80 served the Apache2 default page. Nothing unusual — or so it seemed.

---

## Step 2 — Source Code Inspection

Viewing the page source of the Apache default page revealed a comment hidden among the HTML: "They will never find me."

Also spotted a reference to an image file: `thm.jpg` loaded from the root of the web server. Tried opening it in the browser — it failed to display. Downloaded it instead:

```bash
wget http://<target-ip>/thm.jpg
```

Tried opening the downloaded file locally. Still wouldn't open. The `file` command revealed the first puzzle:

```bash
file thm.jpg
```

Despite the `.jpg` extension, the file's contents identified it as a PNG. The file had the wrong magic bytes — its header was PNG, not JPEG.

---

## Step 3 — Fixing the Image Magic Bytes

Magic bytes (file signatures) sit at the very beginning of a file and tell the operating system what type of file it really is. The file had PNG magic bytes (`89 50 4E 47`) but was named `.jpg`. To open it as a JPEG it needed a JPEG header (`FF D8 FF E0`).

Opened the file in a hex editor:

```bash
hexedit thm.jpg
```

Changed the first few bytes from the PNG signature to a valid JPEG signature. Saved and closed.

Now the file opened — and inside the image was the name of a hidden directory on the web server.

---

## Step 4 — Hidden Directory & the Secret Number

Navigated to the hidden path in the browser:

```
http://<target-ip>/th1s_1s_h1dd3n/
```

The page said I needed to guess a secret. Inspecting the source code revealed a comment: the secret was a number between 0 and 99, passed as a URL parameter called "secret".

```
http://<target-ip>/th1s_1s_h1dd3n/?secret=1
```

Wrong guess. Wrong for 1, wrong for 2 — I needed to automate this. Generated a wordlist of numbers 0 to 99:

```bash
for i in $(seq 0 99); do echo $i >> out.txt; done
```

Used ffuf to fuzz the parameter, filtering out the default response size (incorrect guesses all returned the same page size):

```bash
ffuf -w out.txt:result -u "http://<target-ip>/th1s_1s_h1dd3n/?secret=result" -fs 407,408
```

ffuf found the correct value: `73`

```
http://<target-ip>/th1s_1s_h1dd3n/?secret=73
```

The page returned a passphrase.

---

## Step 5 — Steganography (Steghide)

With a passphrase in hand but no obvious place to use it, I came back to the downloaded `thm.jpg` file. Steganography tools can hide data inside image files — worth checking.

```bash
steghide --extract -sf thm.jpg
```

Entered the passphrase when prompted. A text file was extracted containing a username: `wbxre`

That looked encoded. Given the CTF context, ROT13 was the first thing to try. Decoded at cachesleuth.com/multidecoder:

```
wbxre → joker
```

Username: `joker`. Still needed a password for SSH.

---

## Step 6 — Password from the THM Room Image

Looked back at the TryHackMe room page itself. The room had a banner image — easy to overlook as decoration. Downloaded it:

```bash
wget <room-image-url>
```

Ran stegseek on it (stegseek is a fast steghide cracker that also handles rockyou automatically):

```bash
stegseek <downloaded-image>
```

Extracted another hidden text file. This one contained the SSH password for the `joker` user.

---

## Step 7 — SSH Access & User Flag

```bash
ssh joker@<target-ip>
```

Entered the discovered password. We're in.

```bash
cat /home/joker/user.txt
```

> **FLAG:** `THM{d5781e53b130efe2f94f9b0354a5e4ea}`

---

## Step 8 — Privilege Escalation (GNU Screen SUID)

Checked sudo permissions — nothing useful. Searched for SUID binaries:

```bash
find / -perm /4000 -type f 2>/dev/null
```

One result stood out immediately:

```
/bin/screen-4.5.0
```

GNU Screen version 4.5.0 has a known local privilege escalation (ExploitDB 41154). It's a bash script exploit — I copied the code, saved it to a file on the target, made it executable:

```bash
chmod +x exploit.sh
./exploit.sh
```

The script ran through a few steps and dropped a root shell.

```bash
whoami
# root
```

---

## Step 9 — Root Flag

```bash
cat /root/root.txt
```

> **FLAG:** `THM{5ecd98aa66a6abb670184d7547c8124a}`

---

## Summary & Takeaways

A heavily puzzle-oriented chain:

1. Apache default page source hid a reference to `thm.jpg`.
2. `thm.jpg` had wrong magic bytes — fixed with a hex editor.
3. Fixed image revealed a hidden directory name.
4. ffuf found the secret number (73) needed to get a passphrase.
5. steghide extracted a ROT13-encoded username from `thm.jpg`.
6. stegseek on the TryHackMe room banner image extracted the password.
7. SSH as joker; SUID GNU Screen 4.5.0 exploit gave root.

The real challenge of Madness is patience — the clues are all there, you just have to look everywhere, including places you'd normally ignore.
