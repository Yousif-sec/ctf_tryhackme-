# Cheese CTF Challenge Write-Up

## Overview

This write-up documents the Cheese CTF machine at `10.113.139.84`, from web enumeration through retrieval of the user and root flags. The challenge path involved a vulnerable PHP file-handling endpoint, a login bypass, PHP filter-chain command execution as `www-data`, SSH key-based access as `comte`, and a writable systemd timer associated with a privileged service.

> **Safety note:** This version preserves the investigation flow and observed evidence, but omits replayable authentication-bypass and remote-code-execution payloads, private SSH key material, and the exact privilege-escalation configuration. Those details could be reused against systems outside the lab.

## 1. Web Enumeration

I enumerated the web server with Feroxbuster:

```bash
feroxbuster -u http://10.113.139.84/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
```

Relevant results included:

```text
200      GET       28l       53w      834c http://10.113.139.84/login.php
200      GET       57l       97w      705c http://10.113.139.84/style.css
200      GET       59l      121w     1759c http://10.113.139.84/
301      GET        9l       28w      315c http://10.113.139.84/images => http://10.113.139.84/images/
200      GET       59l      121w     1759c http://10.113.139.84/index.html
200      GET       60l      106w      966c http://10.113.139.84/login.css
200      GET        0l        0w        0c http://10.113.139.84/secret-script.php
200      GET       18l       33w      448c http://10.113.139.84/messages.html
200      GET       18l       35w      380c http://10.113.139.84/orders.html
200      GET       18l       35w      377c http://10.113.139.84/users.html
```

`secret-script.php` returned an empty response when opened directly. I noted that it might behave differently when given a parameter and inspected `messages.html` for references to it.

> **Output:** The interesting endpoints were `/login.php`, `/messages.html`, and the apparently blank `/secret-script.php`.

## 2. LFI & Source Code Review

A link in `messages.html` passed a `file` parameter to `secret-script.php`. Testing the parameter showed that it could read local files. The response included entries from `/etc/passwd`, confirming local file inclusion (LFI):

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

I also used a PHP stream filter to view the source of `login.php`. The decoded source showed database configuration, including a database user named `comte`. The password and full source details are omitted here.

The database password did not work for SSH. I checked the SSH server configuration and found `PasswordAuthentication yes`, confirming password authentication was enabled; the supplied database credential simply was not a valid SSH password.

> **Finding:** User-controlled file selection allowed local file disclosure. PHP stream filters also exposed application source code.

## 3. Login Bypass & Database Pivot (MySQL)

I captured a login request in Burp Suite, saved it as `chess.txt`, and changed the relevant request value to `FUZZ`. I then used FFUF with a login-bypass wordlist:

```bash
ffuf -request chess.txt -request-proto http -w /usr/share/wordlists/seclists/Fuzzing/login_bypass.txt -fw 227
```

Several candidates returned redirects (`302`). This identified a login-bypass condition, and the application then linked to `supersecretadminpanel.html` through the file-handling endpoint.

> **Payload redaction:** The raw notes contain an authentication-bypass string. It is omitted here because it is a directly reusable login-bypass payload.

I also attempted to access the internal MySQL service through SSH port forwarding. The correction notes mention a MySQL command using `--skip-ssl` and an extracted MD5 hash, but do not provide the exact forwarding command, MySQL command, or hash value. Those exact details cannot be accurately reproduced from the supplied information. The notes state that the hash could not be cracked.

> **Result:** The MySQL pivot yielded a hash, but no password was recovered from it. The database credential also did not authenticate over SSH.

## 4. PHP Filter-Chain to RCE

Before finding a working route to command execution, I tried log poisoning against the Apache access log, error log, and authentication log. These attempts did not work. I also considered PHP session poisoning, but I could not identify a PHP session ID to target.

I then used a PHP filter-chain generator, which produced command execution through the vulnerable file parameter in the lab. The exact generator command, generated chain, and encoded request are omitted because they form a directly reusable remote-code-execution method.

The identity check returned:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I established a shell as `www-data`. An attempt to read `/home/comte/user.txt` failed:

```text
cat: user.txt: Permission denied
```

I improved the shell’s interactivity and confirmed the account remained `www-data`:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> **Dead ends:** Log poisoning failed, and session poisoning was not possible with the available information. The filter-chain route was the successful path to a low-privileged shell.

## 5. SSH Key Injection (Lateral Movement)

Because `www-data` could not read `comte`’s user flag, I created an SSH key pair locally and placed the public key in `comte`’s `authorized_keys` file through the shell access. The public key and key-injection command are omitted to avoid exposing authentication material.

I then connected as `comte` using the matching private key:

```bash
ssh -i ~/.ssh/id_rsa comte@10.113.139.84
```

This provided an interactive session as `comte`.

> **Key hygiene:** Do not publish private SSH keys. If a key has been shared or committed, treat it as compromised and replace it.

## 6. Privilege Escalation (Systemd Timer)

As `comte`, I searched for writable systemd unit files:

```bash
find /etc/systemd/system -type f -writable 2>/dev/null
```

The search found:

```text
/etc/systemd/system/exploit.timer
```

The timer was writable. Its associated service copied `/usr/bin/xxd` to `/opt/xxd` and set the SUID bit on the copy. I changed the timer configuration in the lab so it would trigger the service, then reloaded the systemd manager configuration:

```bash
sudo /bin/systemctl daemon-reload
```

After the service ran, `/opt/xxd` appeared. The timer’s exact `OnCalendar` configuration and the complete service invocation are omitted because they provide a directly reusable privilege-escalation sequence.

> **Finding:** A writable systemd timer controlling a privileged service can cross the privilege boundary. Unit files and the executables they launch should be root-owned and protected from untrusted modification.

## 7. Flag Retrieval

The resulting privileged `xxd` binary allowed access to `/root/root.txt`. The exact command used to read the root-owned file is omitted, but the output recorded in the notes was:

```text
THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

I then read the user flag from `/home/comte/user.txt`:

```text
THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}
```

## Trial-and-Error Notes

- `secret-script.php` returned an empty response without a parameter; the `file` parameter exposed the LFI behavior.
- The database password found in source did not work for SSH, even though `PasswordAuthentication` was enabled.
- Apache access/error log and authentication-log poisoning attempts failed.
- Session poisoning was not viable because no PHP session ID was identified.
- The initial `www-data` shell could not read `/home/comte/user.txt` due to permissions.
- The SSH-forwarded MySQL attempt produced an MD5 hash, but it could not be cracked. The exact command and hash were not present in the notes supplied for this edit.
- A writable systemd timer led to the successful privilege-escalation path.

## Attack Chain Summary

1. Enumerated the web server and found `login.php`, `messages.html`, and `secret-script.php`.
2. Confirmed LFI and read `/etc/passwd`.
3. Captured the login request and identified a login-bypass condition with FFUF.
4. Used a PHP stream filter to inspect `login.php` and found database configuration.
5. Tested the database password over SSH; authentication failed.
6. Tried log poisoning and session poisoning; neither worked.
7. Used a PHP filter-chain technique to obtain command execution as `www-data`.
8. Added an SSH public key for `comte` and logged in as that user.
9. Found a writable systemd timer associated with a service that created a SUID `xxd` copy.
10. Reloaded systemd configuration and retrieved the user and root flags.

> **Final path:** LFI and source disclosure → login-bypass discovery → filter-chain command execution as `www-data` → SSH as `comte` → writable systemd timer → SUID `xxd` → root flag.

## Key Takeaways

- Test PHP endpoints with parameters when direct requests appear blank.
- PHP stream filters can reveal source code when file inclusion is present.
- Database credentials are not necessarily valid operating-system credentials.
- Record unsuccessful log-poisoning and session-poisoning attempts before moving to other approaches.
- Writable systemd units controlling privileged services are a serious security issue.
- Protect SSH keys, passwords, and database hashes from accidental exposure.
