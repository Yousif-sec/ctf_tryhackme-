# CyberCrafted — CTF Write-Up

## Challenge Overview

This challenge involved a Linux host running SSH, Apache, and a Minecraft service. The route through the machine included web-host discovery, SQL injection in a store search page, access to an administrative panel, retrieval and cracking of an SSH key passphrase, and a sudo-permitted Screen session that provided root-level access.

The attack chain included:

- Initial service scanning
- Web and virtual-host enumeration
- A failed attempt to access the store site and unsuccessful login-bypass guesses
- SQL injection in the store search feature and discovery of database records
- Command execution through the admin panel to retrieve an SSH key
- Passphrase recovery and SSH access as `xxultimatecreeperxx`
- Minecraft server file and credential discovery
- Switching to `cybercrafted`
- Abuse of a sudo permission involving a Screen session to obtain root

Target details:

- Host: `10.114.164.198`
- Virtual hosts: `store.cybercrafted.thm` and `admin.cybercrafted.thm`
- Open services: SSH (`22/tcp`), HTTP (`80/tcp`), and Minecraft (`25565/tcp`)
- Initial SSH user: `xxultimatecreeperxx`
- Later user: `cybercrafted`

> This write-up is based on the supplied authorized CTF notes. The notes contain truncated user and root flags, so their missing characters are not guessed. Direct SQL-injection, command-injection, credential-cracking, and privilege-escalation payloads are intentionally not reproduced as copy-paste instructions.

---

## 1. Initial Reconnaissance

I started with an Nmap service scan:

```bash
nmap 10.114.164.198 -sV -sS
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-07 16:42 -0500
Nmap scan report for 10.114.164.198
Host is up (0.12s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.29 ((Ubuntu))
25565/tcp open minecraft  Minecraft 1.7.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The open Minecraft port was an unusual service alongside SSH and HTTP, and it later became relevant after obtaining a local account.

---

## 2. Web Enumeration and Virtual Hosts

I enumerated web paths with Feroxbuster:

```bash
feroxbuster -u http://10.114.164.198/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

The notes do not include the Feroxbuster results, so I cannot list specific discovered paths from that scan. I then visited the store virtual host:

```text
http://store.cybercrafted.thm/
```

At first, I did not have permission to open this page. I then tried the admin virtual host:

```text
http://admin.cybercrafted.thm/
```

This displayed a login page. I tried several login-bypass candidates, but they did not work. This was an early dead end, so I moved on to look for other accessible functionality.

> The raw notes say the store page could not be opened initially, but do not record the exact browser or DNS error. The virtual-host names were nevertheless identified and used later.

![Web virtual hosts placeholder](images/virtual-hosts.png)

---

## 3. SQL Injection in the Store Search

I found a search endpoint at:

```text
http://store.cybercrafted.thm/search.php
```

I investigated the search form for SQL injection using sqlmap. The original notes show an automated form test and database dump, but the exact dump command is omitted here.

The database results included these records:

| ID | Hash/value | User |
|---:|---|---|
| 1 | `88b949dd5cdfbecb9f2ecbbfa24e5974234e7c01` | `xXUltimateCreeperXx` |
| 4 | `THM{bbe315906038c3a62d9b195001f75008}` | `web_flag` |

The `web_flag` record contained a complete flag:

```text
THM{bbe315906038c3a62d9b195001f75008}
```

The discovered user also provided a lead for the next stage. I was able to access the login/panel flow and found an operating-system command feature in `panel.php`.

> **Finding:** The store search feature exposed database contents through SQL injection. The hash-like value associated with `xXUltimateCreeperXx` was retained as found; the notes do not document any separate cracking result for it.

![Store search and database result placeholder](images/store-sql-injection.png)

---

## 4. Admin Panel and SSH Key Discovery

The admin panel had an OS command function. I used it to read the private SSH key associated with the `xxultimatecreeperxx` account. The exact command-injection request is omitted, but the file retrieved was:

```text
/home/xxultimatecreeperxx/.ssh/id_rsa
```

I saved the key locally as `id` and attempted to recover its passphrase using John the Ripper's SSH-key workflow. The notes record that the passphrase was recovered:

```text
[SSH key passphrase redacted]
```

I then set restrictive permissions on the key and used it to connect over SSH as `xxultimatecreeperxx`. The connection succeeded.

```text
xxultimatecreeperxx@cybercrafted:~$ id
uid=1001(xxultimatecreeperxx) gid=1001(xxultimatecreeperxx) groups=1001(xxultimatecreeperxx),25565(minecraft)
```

The user belonged to the `minecraft` group, which suggested that the Minecraft installation might be accessible from this account.

---

## 5. Minecraft Server Enumeration

I inspected `/opt` and found a Minecraft directory:

```text
xxultimatecreeperxx@cybercrafted:/opt$ ls
minecraft

xxultimatecreeperxx@cybercrafted:/opt/minecraft$ ls
cybercrafted  minecraft_server_flag.txt  note.txt  WorldBackup
```

The server flag file contained a truncated value in the notes:

```text
THM{ba93767ae3db9..............}
```

I also inspected the LoginSystem plugin log. It contained a registration entry and a password for the `cybercrafted` account:

```text
cybercrafted registered. PW: [redacted]
```

I used the discovered password to switch to the `cybercrafted` user. The password itself is redacted here because it is a credential.

> **Finding:** The Minecraft plugin log exposed an account credential. This provided a lateral move from `xxultimatecreeperxx` to `cybercrafted`.

![Minecraft files placeholder](images/minecraft-files.png)

---

## 6. Privilege Enumeration as `cybercrafted`

As `cybercrafted`, I checked the available sudo permissions. The notes show that this user could run `/usr/bin/screen -r cybercrafted` as root without listing other allowed commands:

```text
Matching Defaults entries for cybercrafted on cybercrafted:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User cybercrafted may run the following commands on cybercrafted:
    (root) /usr/bin/screen -r cybercrafted
```

This was the final escalation clue. The command attached to an existing Screen session, and the session interaction allowed access to a root-privileged context. The exact key sequence and command transcript are omitted.

The notes recorded the resulting identity as:

```text
uid=0(root) gid=1002(cybercrafted) groups=1002(cybercrafted)
```

The effective user ID was root, even though the primary group shown was still `cybercrafted`.

---

## 7. Flag Retrieval

From the root context, I listed `/root` and found `root.txt`. The supplied root flag is truncated:

```text
THM{8bb1eda065ceefb5795a24...........}
```

I also searched for `user.txt` and found it at:

```text
/home/cybercrafted/user.txt
```

The notes show an interrupted initial attempt (`^C`) followed by another attempt to read it. The recorded flag is also truncated:

```text
THM{b4aa20aaf08f17447.................}
```

The complete user and root flag values were not present in the supplied notes, so the omitted characters are not reconstructed.

![Flag retrieval placeholder](images/flags.png)

---

## 8. Trial-and-Error Log and Dead Ends

This section keeps the unsuccessful steps and troubleshooting details from the notes.

### Store page initially inaccessible

- I tried `http://store.cybercrafted.thm/` and did not have permission to open it.
- The exact error was not included in the notes.
- I continued by visiting the admin virtual host and later found the store search endpoint.

### Login-bypass guesses failed

- The admin virtual host showed a login page.
- I tried login-bypass inputs, but they did not work.
- The notes do not list the individual failed guesses.
- I moved on and found the search page instead.

### SSH access came after key discovery

- The notes do not show a successful password-based SSH attempt.
- Access was obtained after retrieving the `xxultimatecreeperxx` private key, recovering its passphrase, and using the key for SSH.

### The key was protected by a passphrase

- Reading the private key was not sufficient by itself; it had a passphrase.
- I saved the key as `id`, processed it with the SSH-key password-recovery workflow, and recovered the passphrase.
- The passphrase and key contents are intentionally not included.

### User/root flag output was incomplete

- The Minecraft server flag, user flag, and root flag were captured only as truncated values in the supplied text.
- No missing characters have been guessed or filled in.

### User flag read was interrupted once

- The notes show an initial `cat` attempt interrupted with `^C`.
- I then ran the read attempt again from the root context and recorded a truncated value.

---

## 9. Final Attack Chain Summary

1. Scan the target and identify SSH, Apache HTTP, and Minecraft services.
2. Enumerate the site and identify the store and admin virtual hosts.
3. Try the admin login-bypass ideas; these attempts fail.
4. Find `search.php` on the store virtual host and identify SQL injection.
5. Recover the web flag and the `xXUltimateCreeperXx` account lead from database records.
6. Use the admin panel's OS command feature to retrieve the account's SSH private key.
7. Recover the key passphrase and connect as `xxultimatecreeperxx`.
8. Inspect the Minecraft installation and find the LoginSystem plugin log.
9. Use the logged credential to switch to `cybercrafted`.
10. Discover the sudo permission for the named Screen session and obtain a root-privileged context.
11. Locate the user and root flags; both are truncated in the available notes.

> Final path: web SQL injection → admin-panel command execution → SSH as `xxultimatecreeperxx` → Minecraft plugin credential → `cybercrafted` → sudo Screen session → root.

---

## 10. Defensive Takeaways

- Use parameterized queries for all search inputs and avoid exposing database contents through injectable queries.
- Restrict administrative panels and do not provide shell-command functionality through web forms.
- Protect private SSH keys, use key passphrases, and prevent web application accounts from reading users' private keys.
- Do not store plaintext passwords in plugin logs or other application files. Rotate credentials if they have been logged.
- Review group membership and file permissions for service accounts such as `minecraft`.
- Do not grant `sudo` access to interactive terminal multiplexers such as Screen unless the configuration is tightly constrained and justified.
- Preserve full evidence and flag values in private lab notes, but never invent missing output.

---

![CyberCrafted web interface placeholder](https://via.placeholder.com/800x250?text=CyberCrafted+Web+Interface)

![CyberCrafted attack path placeholder](https://via.placeholder.com/800x250?text=Web+to+SSH+to+Root)
