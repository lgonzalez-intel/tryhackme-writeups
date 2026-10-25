# Pickle Rick — TryHackMe (Easy)

A Rick and Morty themed challenge: exploit a web server and find three "secret ingredients" (flags) hidden across the machine.

> This write-up explains the **method**, not the answers. No flags or passwords are included.

---

## 🎯 Objective

Rick has turned himself into a pickle and lost his password. The goal is to get into his web server and recover three ingredients for the potion that turns him back.

## 🔍 Recon

```
ping -c 3 <IP>          # host is up
nmap -sV -sC <IP>       # version detection + default scripts
```

Open ports:

| Port | Service | Notes |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2 (Ubuntu) |
| 80/tcp | HTTP | Apache 2.4.41 — page title "Rick is sup4r cool" |

The host is **Ubuntu (Linux)**, so Linux commands apply. As it's a web challenge, the focus is port 80.

## 🧭 Web enumeration

- **Page source (Ctrl+U):** an HTML comment leaks the **username**.
- **`/robots.txt`:** contains an odd string — saved as a possible password.
- **`/login.php`:** a login form.

## 🔑 Access

Logging in with the username from the source and the string from robots.txt gives access to a **Command Panel** — a web form that runs commands on the server.

```
ls      # lists files, including the first ingredient and a clue file
```

## 🥒 First ingredient — bypassing a command filter

`cat` is **disabled** ("Command disabled to make it hard"). The goal is just to read a file, and `cat` isn't the only way to do that in Linux:

```
less <file>     # also: more, head, tail, nl, grep
```

**Lesson:** a filter that blocks only the obvious command is bypassed with an equivalent one.

## 🥒 Second ingredient — exploring the file system

The clue file says to look around the file system.

```
ls /home                  # lists users
ls -l /home/rick          # first char '-' = file, 'd' = directory
less "/home/rick/<file>"  # quotes because the name has a space
```

## 🥒 Third ingredient — privilege escalation

`/root` isn't readable as the web user, but `sudo` works **without a password**:

```
sudo ls /root
sudo less /root/<file>
```

**This is the key weakness:** passwordless `sudo` lets the web user act as root — a textbook privilege escalation.

## 📚 What I learned

- The methodology: recon → web enumeration → access → privilege escalation.
- Reading files in Linux isn't only `cat` — `less`, `more`, `head`, `tail`, `nl`, `grep` all work.
- `ls -l`: the first character tells file (`-`) vs directory (`d`); the rest are permissions (rwx for owner/group/others).
- Full paths matter, and names with spaces need quotes.
- `sudo` runs as administrator; passwordless `sudo` is a serious misconfiguration.
- HTML/CSS/JS run in the browser; PHP runs on the server.

## 🛡️ Mitigation (blue team view)

- Don't leave credentials in page source or robots.txt.
- Never expose a command-execution panel to the web; validate and restrict input.
- Require a password for `sudo` and limit which commands a user can run.
- Apply least-privilege file permissions — avoid `rwxrwxrwx`.
- Keep services patched and disable unused ones.
