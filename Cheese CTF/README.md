# Cheese CTF Challenge Write-Up

## Overview

This walkthrough documents the Cheese CTF machine, from web enumeration through root-flag retrieval. The challenge path involved a local file inclusion (LFI) weakness, an SQL-injection login bypass, PHP filter-chain command execution, SSH access as `comte`, and a writable systemd timer that resulted in a SUID copy of `xxd`.

> **Authorized-lab note:** This is a retrospective of a CTF exercise. Replayable injection payloads, generated code-execution chains, reverse-shell commands, SSH key material, and exact privilege-escalation edits are intentionally not included. Do not test these techniques against systems without permission.

## 1. Initial Web Enumeration

The supplied notes did not include a port scan, so I started with the recorded web enumeration:

```bash
feroxbuster -u http://10.113.139.84/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

### Relevant results

```text
200 GET /login.php
200 GET /style.css
200 GET /
301 GET /images => /images/
200 GET /index.html
200 GET /login.css
200 GET /secret-script.php
200 GET /messages.html
200 GET /orders.html
200 GET /users.html
```

`secret-script.php` returned an empty response when visited directly. I noted it as a potentially interesting endpoint and checked how the pages used it.

> **Output:** The site exposed a login page, several content pages, and a PHP endpoint that accepted a `file` parameter.

## 2. File Inclusion Discovery

While reviewing `messages.html`, I found a link to `secret-script.php` that passed a `file` parameter. Testing the parameter confirmed that it could read local files; `/etc/passwd` content was returned.

The endpoint also referenced a message resource for the administrator. This established the LFI behavior and gave me a way to inspect application files.

> **Finding:** The application used a user-controlled file path without adequate validation. The exact traversal request is omitted.

> **Output:** The response began with standard `/etc/passwd` entries, including the root account.

## 3. Login Bypass and Source Review

I had no credentials for `login.php`, so I captured a login request in Burp Suite, saved it as `chess.txt`, and used FFUF with a login-bypass wordlist. The request was prepared with a `FUZZ` position. Several candidates returned redirects; a boolean-style SQL injection candidate produced a `302` response and reached `supersecretadminpanel.html`.

> The original notes record the exact bypass string. It is not reproduced here as a copy-pastable authentication-bypass payload.

I then used a PHP base64 stream filter through the file-reading endpoint to inspect `login.php`. After decoding the response, I found database configuration for the `users` database and an account named `comte`. The database password is omitted.

The database credentials did not work for SSH. I checked the SSH server configuration and found `PasswordAuthentication yes`, so password authentication was enabled; the failed login was due to the password not being valid for that SSH account, rather than password authentication being disabled.

> **Output:** Source review exposed application database credentials, but they were not valid SSH credentials.

## 4. Post-Exploitation Dead Ends

Before finding a working route to command execution, I tried standard LFI follow-up techniques:

- **Log poisoning:** I attempted to access the Apache access log, error log, and authentication log. These attempts did not produce a usable poisoning path.
- **PHP session poisoning:** I could not identify a valid PHP session ID from the responses, so I could not target a session file.
- **SSH credential reuse:** The database password failed over SSH.

These failed attempts were useful: they ruled out the obvious log/session approaches and showed that the exposed database credentials were separate from the SSH account credentials.

## 5. PHP Filter-Chain Command Execution

After the log- and session-poisoning attempts failed, I used `php_filter_chain_generator.py` to generate a PHP filter chain for the vulnerable file parameter. The chain was URL-encoded and used in the authorized lab to run a basic identity check.

The result confirmed command execution as the web-server account:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> **Redaction note:** The generator invocation, generated filter chain, encoded request, and command-execution payload are omitted because they provide a reusable path to remote code execution.

I obtained an interactive shell and attempted to read `/home/comte/user.txt`. As `www-data`, the read failed with `Permission denied`. I then used Python PTY handling to improve shell interactivity; the exact shell-upgrade command is omitted.

> **Output:** The initial foothold was `www-data`, and it did not have permission to read `comte`’s user flag.

## 6. SSH Access as `comte`

To move from the web account to `comte`, I generated an SSH key pair locally and added the public key to the account’s `authorized_keys` file through the shell access. I then connected using the corresponding private key.

> **Credential hygiene:** The public-key text and any private key are not included. Keep private keys out of source control and rotate them if exposed.

After connecting as `comte`, I had permission to inspect systemd unit files and continue local privilege enumeration.

## 7. MySQL Pivot Attempt

The notes supplied in the correction also describe using SSH port forwarding to reach a MySQL service bound to the target’s loopback interface on port `3306`. The database was queried and user hashes were extracted, but the MD5 hash could not be cracked.

This did not provide the successful root path, but it was a useful additional enumeration attempt. Do not expose the recovered hashes or database contents in a public write-up.

## 8. Writable Systemd Timer

As `comte`, I searched for writable files in the systemd unit directory:

```bash
find /etc/systemd/system -type f -writable 2>/dev/null
```

The search identified:

```text
/etc/systemd/system/exploit.timer
```

I reviewed the timer and its associated service. The service was configured to copy `/usr/bin/xxd` into `/opt` and set the SUID bit on the copy. Because the timer file was writable by `comte`, its schedule could be changed to trigger that privileged service.

After modifying the timer in the lab, I reloaded systemd’s unit configuration with:

```bash
sudo /bin/systemctl daemon-reload
```

The service then ran, and `/opt/xxd` appeared with elevated privileges. The exact timer schedule and complete unit contents are omitted to avoid providing a turnkey privilege-escalation recipe.

> **Finding:** A user-writable systemd timer can control when a privileged service runs. Systemd unit files should be root-owned and not writable by untrusted users. Review both timer and service permissions, and ensure privileged services do not create SUID copies of general-purpose utilities.

## 9. Flag Retrieval

The SUID copy of `xxd` was used in the lab to read the root flag. The recorded flags were:

```text
User flag: THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}
Root flag: THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

> **Output:** Both the user and root flags were retrieved, completing the challenge.

## 10. Trial-and-Error Summary

1. `secret-script.php` returned an empty response directly; its `file` parameter was the useful path.
2. The database password found in source did not work for SSH, even though `PasswordAuthentication` was enabled.
3. Apache access/error log and authentication-log poisoning attempts failed.
4. Session poisoning was not possible with the available information because I could not identify a PHP session ID.
5. The initial `www-data` shell could not read `/home/comte/user.txt` due to file permissions.
6. The SSH-forwarded MySQL attempt exposed hashes, but the MD5 value I tried could not be cracked.
7. The working escalation clue was the writable systemd timer and its privileged associated service.

## 11. Attack Chain Summary

1. Enumerated the web server and found `login.php`, `messages.html`, and `secret-script.php`.
2. Identified LFI through the `file` parameter and confirmed it by reading `/etc/passwd`.
3. Captured the login request and found an authentication-bypass condition with FFUF.
4. Read `login.php` through a PHP stream filter and found database credentials; SSH password reuse failed.
5. Tried log poisoning and session poisoning; neither worked.
6. Used a PHP filter-chain technique to obtain command execution as `www-data`.
7. Added an SSH public key for `comte` and logged in as that user.
8. Forwarded access to the internal MySQL service and extracted a hash that could not be cracked.
9. Found a writable systemd timer that triggered a privileged service creating a SUID `xxd` copy.
10. Reloaded systemd configuration, used the resulting binary to retrieve the root flag, and read the user flag.

> **Final escalation path:** LFI → PHP filter-chain command execution as `www-data` → SSH as `comte` → writable systemd timer → SUID `xxd` → root flag.

## Key Takeaways

- Test PHP endpoints with parameters when direct requests appear blank.
- Source-code disclosure can reveal database credentials, but those credentials may not be valid for system accounts.
- Log and session poisoning are possibilities to test, not guaranteed outcomes; record failures and move on when evidence rules them out.
- Writable systemd units are a critical privilege boundary issue, particularly when they control privileged services.
- Protect database hashes, passwords, and SSH private keys from being committed to a repository.
