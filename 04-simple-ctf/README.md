# Simple CTF — TryHackMe (Easy)

A beginner box where every service version is patched against direct entry, so the way in is the **web application** running on top of them — a vulnerable CMS. From there, cracked credentials give an SSH foothold and a single misconfigured `sudo` rule leads to root.

> This write-up explains the **method**, not the answers. No flags or passwords are included.

---

## 🎯 Objective

From a single IP, find a foothold, read the user flag, and escalate to root for the root flag.

## 🔍 Recon

```
nmap -sC -sV -oN simplectf_nmap.txt <IP>
```

Open ports:

| Port | Service | Notes |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 (anonymous login allowed) |
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu) |
| 2222/tcp | SSH | OpenSSH 7.2p2 |

Two things stand out:

- **Only two services run under port 1000** (FTP and HTTP). SSH is deliberately moved to a **high, non-standard port (2222)** — a lazy default scan would miss it. Lesson: always enumerate thoroughly, and consider a full-port sweep (`nmap -p-`).
- Anonymous FTP is allowed — worth a look, but it held nothing useful here.

## 🧭 Enumeration — discarding the services

Each service version was checked for public exploits:

```
searchsploit openssh 7.2
searchsploit vsftpd 3.0.3
searchsploit apache 2.4.18
```

- **OpenSSH 7.2p2** → only username enumeration / DoS. Nothing that grants access from scratch.
- **vsftpd 3.0.3** → DoS only. (The famous vsftpd **2.3.4** backdoor is a classic trap — it does *not* apply to 3.0.3.)
- **Apache 2.4.18** → mostly DoS, *local* privilege escalation, or exploits for other versions/modules.

Everything was patched against direct entry → the real attack surface is the **web application**, not the server software.

## 🌐 Enumeration — the web

The site root only showed the default Apache page, so content was brute-forced:

```
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

`robots.txt` was also read by hand and pointed at `/openemr-5_0_1_3`.

> ⚠️ **Rabbit hole:** that path did not load — the app was not actually installed. A name in `robots.txt` doesn't guarantee the service exists. Verifying before launching exploits saved time.

The real application was at **`/simple`** → **CMS Made Simple**, version **2.2.8** (confirmed in `/simple/doc/CHANGELOG.txt`).

## 💥 Access — SQL Injection (CVE-2019-9053)

`searchsploit` returned *CMS Made Simple < 2.2.10 — SQL Injection* (`46635.py`); version 2.2.8 is in range.

```
searchsploit -m php/webapps/46635.py
python2 46635.py -u http://<IP>/simple --crack -w /usr/share/wordlists/rockyou.txt
```

The script is Python 2 and depends on `termcolor`, which no longer installs cleanly; the import was only cosmetic (coloured output) and was stubbed out so the exploit would run.

The injection leaks the admin **username**, **password hash** and **salt** character by character, and `--crack` recovers the plaintext password from `rockyou.txt`.

**Why it works:** the application builds its SQL queries by concatenating user input without separating data from code, so a crafted parameter is executed as part of the query. The correct fix is parameterised queries.

With the recovered credentials, log in over SSH — remembering the non-standard port:

```
ssh <user>@<IP> -p 2222
```

User flag is in the user's home directory.

## 🔓 Privilege escalation — sudo + vim (GTFOBins)

Check what the user can run as root:

```
sudo -l
# (root) NOPASSWD: /usr/bin/vim
```

`vim` can spawn a system shell from inside itself; run as root via `sudo`, that shell is also root (technique from **GTFOBins**, *Shell → Sudo*):

```
sudo vim -c ':!/bin/sh'
```

`whoami` → `root`. Root flag is in `/root`.

## 🎓 What I learned

- **Enumerate everything.** SSH on 2222 is invisible to a default scan — a full-port sweep catches it.
- **Patched services ≠ safe host.** When every service version is patched, look at the application layer.
- **Read `robots.txt` critically.** The `openemr` entry was a deliberate rabbit hole; verify a path responds before committing to it.
- **Match the CVE to the exact version.** `vsftpd 3.0.3` is not the backdoored `2.3.4`.
- **`sudo -l` + GTFOBins** is a fast, reliable privesc check whenever a binary runs as root.

## 🛡️ Mitigation (blue team)

- **Update the CMS.** CVE-2019-9053 is fixed in CMS Made Simple ≥ 2.2.10; keeping third-party apps patched closes the entry point.
- **Use parameterised queries / prepared statements** so user input can never be executed as SQL.
- **Remove the `sudo NOPASSWD` rule on `vim`** (and other interactive binaries). If a user needs elevated rights, scope them to the exact action, not a shell-capable editor.
- **Enforce strong passwords** — the whole chain only worked because the admin password was in `rockyou.txt`.
- **Detection:** SQL injection attempts appear in web logs as requests containing `'`, `UNION SELECT`, `OR 1=1`, etc. A service on a non-standard port deserves monitoring of *what* runs there, not just the port number.

---

**MITRE ATT&CK:** T1595 (Active Scanning) · T1190 (Exploit Public-Facing Application) · T1110.002 (Password Cracking) · T1078 (Valid Accounts) · T1548.003 (Sudo and Sudo Caching)
