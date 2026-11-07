# Cheese CTF Challenge Write-Up

## Overview

This walkthrough documents the Cheese CTF machine, from web enumeration to root. The path involved a blind PHP file-inclusion issue, a login bypass, PHP filter-chain code execution, SSH access as `comte`, and a writable systemd timer that led to a root-owned SUID `xxd` binary.

The main steps were:

- Enumerating the web application and discovering `secret-script.php`
- Using its `file` parameter to read local files
- Testing the login form and finding a bypass condition
- Recovering database configuration and confirming those credentials did not work over SSH
- Ruling out log poisoning and session poisoning
- Using a PHP filter-chain technique for code execution as `www-data`
- Installing an SSH public key for access as `comte`
- Finding a writable systemd timer and using the resulting SUID `xxd` to read the root flag

> **Lab scope and secrets:** This is documentation for an authorized TryHackMe-style CTF environment. The database password, SSH public key, and replayable code-execution payload are redacted. Do not reuse or publish challenge credentials or private keys.

---

## 1. Initial Reconnaissance

The supplied notes did not include the port-scan command or its output, so I began with the recorded web enumeration results rather than inventing scan results.

> **Add the Nmap output here if you have it:** `./assets/nmap-scan.png`

---

## 2. Web Enumeration

I used Feroxbuster to enumerate the web server:

```bash
feroxbuster -u http://<target-ip>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

Relevant results included a login page, several content pages, and `secret-script.php`:

```text
200 GET /login.php
200 GET /style.css
200 GET /
301 GET /images/ => /images/
200 GET /index.html
200 GET /login.css
200 GET /secret-script.php
200 GET /messages.html
200 GET /orders.html
200 GET /users.html
```

The `secret-script.php` page returned an empty response when visited directly, so I treated it as a potentially blind file-handling endpoint and continued reviewing the site pages.

![Image Placeholder: Feroxbuster results](./assets/web-enumeration.png)

---

## 3. File Inclusion Discovery

The messages page exposed a link that passed a `file` parameter to `secret-script.php`. I inspected how that parameter behaved. A file-wrapper test did not immediately give useful output, but a path traversal attempt was successful and returned the beginning of `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

This confirmed that the `file` parameter could be used to read local files. The exact traversal request is omitted from this write-up.

![Image Placeholder: Redacted file-inclusion response](./assets/file-inclusion.png)

---

## 4. Login Testing and Database Configuration

The site also exposed `/login.php`. I did not have a known username or password, so I captured a login request in Burp Suite, saved it as `chess.txt`, and used FFUF with a login-bypass wordlist. The request filename in my original notes was `chess.txt`; I changed the relevant request value to the `FUZZ` marker before running the test.

```bash
ffuf -request chess.txt \
  -request-proto http \
  -w /usr/share/wordlists/seclists/Fuzzing/login_bypass.txt \
  -fw 227
```

Several candidate inputs returned redirects. One boolean-style SQL injection candidate appeared to bypass the login, and the authenticated path linked to `supersecretadminpanel.html` through the file endpoint.

> **Finding:** The login form did not properly handle user-supplied input. The specific bypass string and replayable request are intentionally not included.

I then used the file-reading functionality with a PHP stream filter to inspect `login.php`. The source revealed local database settings, including a database username and password. Those database credentials did **not** work for SSH; this was a failed credential-reuse attempt, not an SSH foothold.

![Image Placeholder: Redacted login bypass and PHP configuration](./assets/login-and-config.png)

---

## 5. Failed Post-Exploitation Paths

Before finding the working execution route, I tried several approaches using the file-inclusion behavior:

### Log poisoning attempts

I tried reading the common Apache and authentication log paths:

```text
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/auth.log
```

These attempts did not produce a usable log-poisoning path.

### PHP session poisoning

I also considered PHP session poisoning. I could not identify a PHP session ID from the available responses, so I did not have a known session file to target. This route was therefore abandoned.

### SSH with the database credentials

The credentials recovered from `login.php` were tested over SSH but did not authenticate. This established that the database account was not the `comte` SSH credential.

> These dead ends helped narrow the investigation: the file-inclusion primitive was useful, but the available log and session artifacts did not provide code execution.

---

## 6. PHP Filter-Chain Code Execution

After the unsuccessful log- and session-poisoning attempts, I used a PHP filter-chain generator compatible with the target’s PHP setup. The generated chain was supplied through the vulnerable file parameter. In the authorized lab, this resulted in command execution as the web server account.

The first identity check confirmed:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> **Redaction note:** The generated filter chain, encoded request, and command payload are omitted because they are directly reusable for remote code execution. The key result was a shell running as `www-data`.

I established an interactive shell and verified the identity again. From `/home/comte`, the `user.txt` file was visible but not readable as `www-data`:

```text
cat: user.txt: Permission denied
```

This permission failure confirmed that the web account was still unprivileged and that another access step was required.

![Image Placeholder: Redacted `www-data` shell and identity check](./assets/www-data-shell.png)

---

## 7. SSH Access as `comte`

I generated an SSH key pair locally and used the public key for the `comte` account. The actual key material is deliberately omitted. After the public key was placed in the account’s `authorized_keys`, I connected with the corresponding private key:

```bash
ssh -i ~/.ssh/id_rsa comte@<target-ip>
```

This provided an interactive shell as `comte`. The private key must remain private; do not commit it to the repository.

![Image Placeholder: SSH session as `comte` with key material redacted](./assets/comte-ssh.png)

---

## 8. Writable Systemd Timer Discovery

As `comte`, I searched for writable files under the systemd unit directory:

```bash
find /etc/systemd/system -type f -writable 2>/dev/null
```

The search found a writable timer:

```text
/etc/systemd/system/exploit.timer
```

I inspected the timer and its associated service. The timer initially had no useful schedule configured. The service was configured to copy `/usr/bin/xxd` to `/opt/xxd` and set the SUID bit on the copy. Because the timer file was writable and the service ran with elevated privileges, this was the privilege-escalation path.

> **Finding:** A user-writable systemd unit can be dangerous when it triggers a privileged service. Unit files and the executables they launch should be owned by root and not writable by untrusted users.

I updated the timer schedule in the lab and reloaded systemd’s unit configuration. After the timer ran, `/opt/xxd` was present with the elevated permission needed for the next step.

> **Redaction note:** The exact timer activation sequence is not reproduced as a turnkey privilege-escalation recipe. The important misconfiguration was a writable timer invoking a root-level service that created a SUID copy of `xxd`.

![Image Placeholder: Writable timer and redacted service configuration](./assets/systemd-timer.png)

---

## 9. Reading the Flags

The SUID `xxd` copy allowed me to read the root-owned flag through its file-processing behavior. The root flag recorded in the notes was:

```text
THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

I then returned to `/home/comte` and read `user.txt`:

```text
THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}
```

![Image Placeholder: Redacted flag retrieval and completion](./assets/flags.png)

---

## 10. Trial-and-Error and Troubleshooting Notes

The unsuccessful attempts and troubleshooting steps are included for completeness:

1. **`secret-script.php` looked blank when opened directly.** The useful behavior was reached through its `file` parameter.
2. **The database password did not work over SSH.** Database credentials and operating-system credentials were not interchangeable here.
3. **Log poisoning did not work.** Attempts to use Apache access/error logs and the authentication log did not provide a usable path.
4. **Session poisoning was not viable with the available information.** I could not identify a PHP session ID to locate or poison a session file.
5. **`www-data` could not read `/home/comte/user.txt`.** The `Permission denied` result showed that the initial shell did not have sufficient privileges.
6. **The working escalation clue was a writable systemd timer.** Inspection of its paired service showed that the timer could trigger the creation of a SUID `xxd` copy.
7. **No `binwalk` or `strings` attempt was included in these notes.** The PHP filter-chain technique was the documented route to command execution.

---

## 11. Attack Chain Summary

1. Enumerated the web server and found `login.php`, `messages.html`, and `secret-script.php`.
2. Investigated the `file` parameter and confirmed local file disclosure by reading `/etc/passwd`.
3. Captured the login request and used FFUF to identify a login bypass.
4. Read the PHP configuration and discovered database credentials; testing them over SSH failed.
5. Tried Apache/authentication log poisoning and PHP session poisoning; neither produced a foothold.
6. Used a PHP filter-chain approach to obtain command execution as `www-data`.
7. Confirmed `www-data` could not read `comte`’s `user.txt`.
8. Added an SSH public key for `comte` and logged in using the matching private key.
9. Found a writable systemd timer associated with a privileged service that created a SUID copy of `xxd`.
10. Used the resulting privileged utility to read the root flag and then read the user flag as `comte`.

> **Final escalation path:** PHP file inclusion → filter-chain command execution as `www-data` → SSH access as `comte` → writable systemd timer → SUID `xxd` → root flag.

---

## Key Takeaways

- A blank PHP endpoint may still expose meaningful behavior through query parameters.
- PHP stream filters can expose source code and, in vulnerable configurations, may contribute to code execution.
- Database credentials should not be assumed to work for SSH or other services.
- Writable systemd unit files can create a direct privilege boundary failure if they control privileged services.
- Avoid publishing passwords, private SSH keys, or reusable exploit payloads in a write-up.

---

![Image Placeholder: Cheese CTF attack-chain summary](./assets/attack-chain.png)
