# 🚩 flag-hoarder

> *Because hoarding flags is the only acceptable kind of hoarding.*

A personal collection of CTF (Capture The Flag) writeups — my way of documenting what I broke, how I broke it, and what I learned along the way. Every challenge in here taught me something, whether it was a clean solve or an embarrassing rabbit hole that cost me three hours.

---

## What's this about?

I use this repo to track my progress through CTF challenges and keep detailed writeups of my solutions. Think of it as a public notebook — structured enough to be useful later, honest enough to include the parts where I had no idea what I was doing.

The writeups cover the full thought process, not just the final payload. If I went down a wrong path before finding the solution, that's in here too. Those detours are usually where the actual learning happens.

---

## Platforms

| Platform | Focus |
|---|---|
| [HackTheBox](https://hackthebox.com) | Machines, challenges, Prolabs |
| [TryHackMe](https://tryhackme.com) | Learning paths, rooms, events |

More platforms may be added as I explore. This repo will keep growing.

---

## Repository Structure

```
flag-hoarder/
│
├── HackTheBox/
│   ├── Machines/
│   │   ├── Easy/
│   │   ├── Medium/
│   │   └── Hard/
│   └── Challenges/
│       ├── Web/
│       ├── Pwn/
│       ├── Crypto/
│       ├── Forensics/
│       ├── Reversing/
│       └── Misc/
│
├── TryHackMe/
│   ├── Rooms/
│   └── Paths/
│
└── Resources/
    └── cheatsheets, notes, useful references
```

> Structure may evolve as the repo grows. I'll keep it logical.

---

## Topics Covered

Not every writeup touches all of these, but collectively this repo deals with:

- **Web exploitation** — SQLi, XSS, SSRF, IDOR, auth bypasses, deserialization
- **Privilege escalation** — Linux & Windows, misconfigs, kernel exploits, SUID/GUID abuse
- **Binary exploitation & Reversing** — buffer overflows, ROP chains, disassembly, patching
- **Cryptography** — classic ciphers, RSA flaws, hash cracking, custom implementations
- **Forensics** — disk/memory analysis, steganography, packet capture analysis
- **Active Directory** — Kerberoasting, Pass-the-Hash, BloodHound enumeration, lateral movement
- **Network analysis** — Wireshark, traffic inspection, protocol analysis
- **OSINT** — recon techniques, metadata, open-source intelligence gathering

---

## Tools I Use Regularly

This isn't exhaustive, but these show up a lot across writeups:

`nmap` · `gobuster` / `ffuf` · `Burp Suite` · `netcat` · `pwntools` · `GDB` / `peda` · `IDA Free` / `Ghidra` · `Volatility` · `Wireshark` · `CyberChef` · `hashcat` / `john` · `BloodHound` / `SharpHound` · `Impacket` · `Metasploit` · `sqlmap` · `LinPEAS` / `WinPEAS`

---

## Progress Tracker

*(Updated as I go)*

| Platform | Machines / Rooms Completed | Challenges Completed |
|---|---|---|
| HackTheBox | 🔄 In progress | 🔄 In progress |
| TryHackMe | 🔄 In progress | 🔄 In progress |

---

## Why writeups?

A few reasons:

1. **Retention** — Writing it down forces you to actually understand what happened, not just get lucky with a payload.
2. **Portfolio** — This repo is a live record of real problem-solving, which matters more to me than a certification list.
3. **Community** — CTF culture runs on knowledge sharing. Someone helped me at some point (a blog post, a hint, a Discord message), and this is a way to give that back.

---

## A note on spoilers

All writeups are for retired machines and released challenges. I don't post solutions to active challenges — that's against platform rules and defeats the purpose anyway.

---

## Get in Touch

Always happy to connect with other CTF players, security folks, or anyone who wants to talk about a writeup.

- GitHub: [@justincognito725](https://github.com/justincognito725)
- TryHackMe Link 1: https://tryhackme.com/p/flag.raider
- TryHackMe Link 2: https://tryhackme.com/p/justflagging
- LinkedIn: www.linkedin.com/in/harshilmakwana

---

<p align="center">
  <i>Built one flag at a time.</i>
</p>