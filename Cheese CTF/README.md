# Cheese CTF — CTF Write-Up

## Challenge Overview

This challenge involved a multi-stage compromise of a Linux machine exposing a PHP website and SSH. The attack chain included:

- Web content enumeration
- Local file inclusion (LFI) through a PHP endpoint
- Login-form testing and credential investigation
- Several unsuccessful code-execution approaches
- PHP filter-chain-based command execution as `www-data`
- Access to the `comte` account
- Abuse of a writable systemd timer and service to obtain root

Target details:

- Host: `10.113.139.84`
- Web service: HTTP on the target (port-scan notes were blank)
- User account: `comte`
- Initial shell: `www-data`
- Root flag: `THM{dca75486094810807faf4b7b0a929b11e5e0167c}`
- User flag: The supplied notes show a truncated value: `THM{9f2ce3df1beeecaf695b3a8560c......`

> These notes describe an authorized CTF lab. Secrets and copy-paste exploitation payloads are redacted; the investigation and failed attempts are retained.

---

## 1. Initial Reconnaissance

The original notes had a `[scan]` / `-ports-` heading but no port-scan command or output. I therefore do not have confirmed port results to reproduce here.

### Web content enumeration

I used Feroxbuster to enumerate common paths and PHP, HTML, and text files:

```bash
feroxbuster -u http://10.113.139.84/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

Observed results:

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

The `secret-script.php` endpoint returned an empty body in this scan. At that point, I did not yet know how it worked. The `messages.html` page became the useful lead.

![Initial web enumeration placeholder](images/initial-enumeration.png)

---

## 2. Discovering Local File Inclusion

I opened the message page and followed a message link. It led to `secret-script.php` with a `file` parameter. I first tried a PHP filter resource reference while testing the message name, then tried directory traversal to read `/etc/passwd`.

The original requests were of this form; the exact file-inclusion payloads are omitted:

```text
http://10.113.139.84/secret-script.php?file=[PHP filter resource reference]
http://10.113.139.84/secret-script.php?file=[directory traversal to /etc/passwd]
```

The traversal request returned account-file contents, including entries such as:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

This confirmed that the `file` parameter could read local files. The endpoint had appeared “blind” based on the initial empty scanner response, but manual interaction revealed the inclusion behavior.

> **Finding:** The application accepted a user-controlled file value and exposed local file contents.

![LFI result placeholder](images/lfi-result.png)

---

## 3. Login Form and Authentication Testing

The site also exposed `http://10.113.139.84/login.php`. I did not have a known username or password, so I captured the login request with Burp Suite, saved it as `chess.txt`, and changed the relevant request value to `FUZZ` for testing.

The original fuzzing command was:

```bash
ffuf -request chess.txt \
  -request-proto http \
  -w /usr/share/wordlists/seclists/Fuzzing/login_bypass.txt \
  -fw 227
```

Several candidate inputs returned HTTP 302 with the same response size and word count. The raw notes recorded these candidates:

```text
[SQL-injection-style candidate redacted]  [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 86ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 81ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 83ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 84ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 89ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 92ms]
[additional candidate redacted]           [Status: 302, Size: 792, Words: 217, Lines: 26, Duration: 92ms]
```

I then tested a candidate with an arbitrary password. The notes say that the login flow also reached the admin-panel content through the file endpoint, at `secret-script.php?file=supersecretadminpanel.html`.

> A 302 response by itself can be a false positive. In this case, the next useful observation was that the admin-panel resource could be reached; the original notes do not include the full response or session details.

---

## 4. Reading the PHP Source and Trying the Database Password over SSH

I used a base64-encoding PHP filter to inspect `login.php`. The request form was:

```text
http://10.113.139.84/secret-script.php?file=[base64-encoding filter for login.php]
```

The decoded PHP source revealed these database settings:

```text
$servername = "localhost";
$user = "comte";
$password = "[redacted]";
$dbname = "users";
```

I tried the database password for SSH as `comte`, but it did **not** work. I then checked `/etc/ssh/sshd_config` to see whether password authentication was disabled. The configuration contained:

```text
PasswordAuthentication yes
```

That ruled out the assumption that SSH password authentication was globally disabled, but it did not make the database password valid for the operating-system account. This was a dead end: the password was not usable for SSH.

---

## 5. Code-Execution Attempts and Troubleshooting

After confirming LFI, I explored ways to turn file access into command execution.

### 5.1 Log poisoning attempt — unsuccessful

I tried the following log paths for a log-poisoning approach:

```text
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

The notes say these attempts “not work to log Poisoning.” No successful poisoning result was obtained.

### 5.2 PHP session poisoning — unavailable

I also considered PHP session poisoning, but I did not have a usable PHP session ID. This approach could not be completed.

### 5.3 PHP filter-chain approach — successful

The notes identify a PHP filter-chain generator, described there as applicable to PHP versions below 8. The generator was run with a PHP command-execution snippet, and the resulting chain was URL-encoded and sent through the vulnerable file parameter. The exact generator payload, generated chain, and command-bearing request are redacted here.

The command output confirmed execution as the web-server user:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> The important result was command execution as `www-data`; the earlier log-poisoning and session-poisoning routes did not work.

![Web-shell result placeholder](images/www-data-shell.png)

---

## 6. Shell Stabilization and `comte` Home Directory

The raw notes show a reverse shell being connected back to the testing machine, followed by a Python PTY upgrade. The exact callback command and payload are omitted. After upgrading the shell, I checked the identity:

```text
www-data@ip-10-113-139-84:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I looked at `/home/comte` and found `snap` and `user.txt`. Reading the flag as `www-data` failed with a permissions error:

```text
www-data@ip-10-113-139-84:/home/comte$ ls
snap  user.txt

www-data@ip-10-113-139-84:/home/comte$ cat user.txt
cat: user.txt: Permission denied
```

This confirmed that the initial shell did not have permission to read the user flag.

---

## 7. SSH Key Attempt and Access as `comte`

The notes then describe generating an RSA key pair on Kali. Key generation did not start cleanly because a key already existed at the default path:

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kali/.ssh/id_rsa):
/home/kali/.ssh/id_rsa already exists.
Overwrite (y/n)?
```

The existing public key was then used in the `comte` account's `authorized_keys` file. The supplied key text is visibly truncated, so the complete key cannot be reproduced. The notes show a successful SSH session as `comte` afterward; the exact key-installation command and SSH command are omitted.

Once connected as `comte`, I checked the home directory and read `user.txt`. The original notes contain only this truncated flag prefix:

```text
THM{9f2ce3df1beeecaf695b3a8560c......
```

I have kept it truncated rather than guessing the missing characters.

---

## 8. Systemd Timer Enumeration

As `comte`, I searched `/etc/systemd/system` for files writable by the current user:

```text
comte@ip-10-113-139-84:~$ find /etc/systemd/system -type f -writable 2>/dev/null
/etc/systemd/system/exploit.timer
```

The timer initially contained:

```ini
[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=

[Install]
WantedBy=timers.target
```

I then inspected the associated service:

```ini
[Unit]
Description=Exploit Service

[Service]
Type=oneshot
ExecStart=[privileged copy-and-permission-change command redacted]
```

The service was configured to copy `/usr/bin/xxd` to `/opt/xxd` and set the set-user-ID bit on the copy. This was the key privilege-escalation clue: an unprivileged user could edit the timer, while the paired service performed a privileged operation.

> **Finding:** A user-writable systemd timer could trigger a privileged service that created a privileged copy of a utility.

---

## 9. Final Privilege Escalation to Root

I edited `exploit.timer` so it would run the associated service on a recurring schedule, then reloaded the systemd manager configuration. The exact timer edit and command sequence are omitted, but the notes confirm that `/opt/xxd` appeared afterward.

The raw transcript showed the privileged utility being used to read `/root/root.txt`; the exact invocation is omitted. It printed the challenge banner and root flag:

```text
THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

This completed the root escalation.

![Root flag placeholder](images/root-flag.png)

---

## 10. Trial-and-Error Log and Dead Ends

This section retains the failed attempts and troubleshooting notes rather than presenting only the successful path.

### Failed SSH password login

- I found database credentials in the PHP source.
- I tried the database password for SSH as `comte`.
- Login failed; the database password was not the SSH password.
- I checked `sshd_config` and found `PasswordAuthentication yes`, so password login was enabled in the server configuration, but the attempted credential was still wrong.

### Log poisoning did not work

I tried these paths, but did not get log poisoning to work:

```text
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

### No PHP session ID for session poisoning

I could not try session poisoning successfully because I did not have a PHP session ID.

### Web-server account could not read `user.txt`

From the `www-data` shell, `user.txt` was visible in the `comte` home directory, but reading it returned `Permission denied`.

### Existing SSH key at the default path

When generating an SSH key, the default `/home/kali/.ssh/id_rsa` already existed. The terminal prompted whether to overwrite it. I retained the original key rather than assuming a fresh key had been generated.

### The PHP endpoint looked empty at first

Feroxbuster reported `secret-script.php` as a zero-byte response. That initial output did not reveal its behavior; following the message link and manually testing its `file` parameter exposed the LFI issue.

### User flag was incomplete in the notes

The recorded `user.txt` output is truncated. The missing characters are not available in the source notes, so the flag is intentionally left incomplete rather than fabricated.

### Port-scan output was absent

The notes started with `[scan]` and `-ports-`, but no scan command or results followed. No open ports or service versions are inferred in this write-up.

---

## 11. Final Attack Chain Summary

1. Enumerate website paths with Feroxbuster.
2. Follow the message link to `secret-script.php` and identify the `file` parameter.
3. Confirm LFI by retrieving `/etc/passwd`.
4. Test the login form with a Burp-captured request and `ffuf`; reach the admin-panel content.
5. Read PHP source and discover database credentials; trying the password over SSH fails.
6. Try Apache and authentication log paths for log poisoning; no success.
7. Consider PHP session poisoning; no session ID is available.
8. Use a PHP filter-chain approach to obtain command execution as `www-data`.
9. Find `user.txt` in `/home/comte`; reading it as `www-data` fails with permission denied.
10. Use SSH key access to connect as `comte` and retrieve the user flag (the value in the notes is truncated).
11. Find a writable systemd timer paired with a privileged service.
12. Trigger the service and use the resulting privileged utility to read the root flag.

> Final path: web enumeration → LFI → `www-data` → `comte` → systemd timer/service misconfiguration → root.

---

## 12. Defensive Takeaways

- Restrict file access to an allowlist of known application resources; do not pass arbitrary user-controlled paths to file-reading functions.
- Validate and canonicalize paths, and ensure the resolved path remains within the intended content directory.
- Use parameterized SQL queries and validate authentication through protected content and session behavior, not just HTTP status codes.
- Do not reuse database credentials for operating-system accounts.
- Ensure application users cannot modify systemd unit or timer files.
- Review privileged services for unsafe file operations and permission changes.
- Protect SSH authorized-key files and monitor unexpected changes to them.
- Keep complete, accurate evidence—including full scan results and flag values—in private lab notes. Do not infer missing data.

---

![Web application placeholder](https://via.placeholder.com/800x250?text=Cheese+CTF+Web+Application)

![Privilege escalation flow placeholder](https://via.placeholder.com/800x250?text=www-data+to+comte+to+root)
