# Cheese CTF — Web Exploitation and Privilege Escalation

> **Scope and safety:** This is a sanitized write-up for an authorized, isolated CTF lab. Exact injection strings, generated filter-chain payloads, reverse-shell commands, key-injection steps, and privilege-escalation commands are intentionally omitted. The narrative preserves the investigation and failed approaches without providing a copy-paste exploitation chain.

## Overview

The challenge exposed a small PHP website and an SSH service. The investigation moved from web enumeration to a file-inclusion issue, then to code execution as the web-server account. A writable systemd timer and its associated service provided the final privilege-escalation path.

**Target noted in the raw notes:** `10.113.139.84`  
**User:** `comte`  
**Root flag recorded:** `THM{dca75486094810807faf4b7b0a929b11e5e0167c}`  
**User flag:** The supplied notes contain only a truncated value, so it is not reconstructed here.

> Run scans and testing only against systems for which you have explicit authorization.

## 1. Initial reconnaissance and web enumeration

The original port-scan section contained no results, so no open-port findings can be reliably reproduced. Web content discovery identified a login page, several static pages, an image directory, and a PHP endpoint whose response was empty during enumeration.

```bash
feroxbuster -u http://10.113.139.84/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

Notable paths from the scan:

| Path | Result | Observation |
|---|---:|---|
| `/login.php` | 200 | Login form |
| `/style.css` | 200 | Site stylesheet |
| `/` and `/index.html` | 200 | Main page |
| `/images/` | 301 | Redirected to the directory URL |
| `/secret-script.php` | 200 | Empty response in the scanner output; later investigation showed it accepted a `file` parameter |
| `/messages.html` | 200 | Message listing; a message linked to the PHP endpoint |
| `/orders.html` | 200 | Orders page |
| `/users.html` | 200 | Users page |
| `/login.css` | 200 | Login-page stylesheet |

**Image placeholder:** `![Initial site and directory-enumeration results](images/initial-enumeration.png)`

## 2. File-inclusion investigation

Following a message link led to `secret-script.php` with a `file` parameter. This suggested that the server was loading a caller-selected resource. A PHP stream-wrapper reference was tried first while exploring the message name, followed by a traversal-style request for the system account file. The latter returned file contents, confirming a local file inclusion (LFI) weakness.

The response included standard account entries such as `root` and system service accounts. This confirmed that the parameter was not safely constrained to the intended message files.

> **Finding:** User-controlled file paths were passed to server-side file handling without an effective allowlist or canonical-path check.

**Image placeholder:** `![File-inclusion behavior](images/file-inclusion.png)`

## 3. Login testing and credential investigation

The login form did not provide credentials. A request captured with Burp Suite was adapted for wordlist-based testing with `ffuf`. Several candidates produced HTTP 302 responses with the same response size and word count. A boolean-condition-style candidate was then used to reach the admin-panel content through the file endpoint.

The exact login-bypass input is omitted. In a real assessment, a redirect alone is not sufficient proof of authentication: compare response bodies, session state, and access to a protected resource to distinguish a genuine bypass from a generic redirect.

### Database credentials: a dead end for SSH

Reading the login PHP source through a base64-encoding stream wrapper revealed database configuration values:

```text
Database user: comte
Database password: [redacted]
Database name: users
```

The database password was tried for SSH access, but authentication failed. The SSH daemon configuration was checked; it indicated that password authentication was enabled. This ruled out password authentication being globally disabled, but did not make the database password valid for the `comte` SSH account. The credentials were service-specific and should not have been assumed to be reusable.

## 4. Code-execution investigation: failed approaches and result

Several server-side code-execution avenues were considered before the successful approach:

1. **Log poisoning:** Attempts to use the Apache access and error logs did not produce code execution. The authentication log was also considered, but no successful poisoning path was established.
2. **PHP session poisoning:** This path could not be pursued because no usable PHP session identifier was available.
3. **PHP filter-chain approach:** A PHP filter-chain technique was eventually used to make the vulnerable file-loading behavior evaluate a command-execution payload. The generated payload and request syntax are intentionally omitted. The resulting command output confirmed execution in the context of `www-data`.

> **Result:** Initial execution was obtained as the low-privileged web-server account (`www-data`), not as `comte` or root.

**Troubleshooting note:** The supplied notes mention filter-chain generation but do not include outputs from `binwalk` or `strings`. No results for those tools are invented or attributed to this stage.

## 5. Post-exploitation and user access

A shell session was upgraded to a more usable interactive terminal. Local enumeration showed the `comte` home directory and a `user.txt` file, but the web-server account could not read it due to file permissions.

The notes then describe adding an SSH public key for `comte` and connecting with the corresponding private key. The key material and exact account-modification commands are omitted. After access as `comte` was obtained, the user flag was read; the supplied value is truncated, so it cannot be faithfully reproduced here.

> **Lesson:** A database password is not automatically an operating-system password. Treat service credentials as scoped, and document failed credential reuse rather than presenting it as a valid login.

## 6. Privilege escalation to root

As `comte`, writable systemd configuration was discovered under `/etc/systemd/system`. The relevant timer was paired with a service that performed a privileged file-copy operation and set the copied utility's set-user-ID bit. The timer configuration was changed so that the service would run on a recurring schedule, and systemd was instructed to reload its configuration. The resulting privileged utility was then used to access the root-owned flag.

The exact timer contents, commands, and set-user-ID invocation are omitted because they would provide a directly reusable privilege-escalation recipe.

**Root flag:**

```text
THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

**Image placeholder:** `![Root flag obtained in the authorized lab](images/root-flag.png)`

## 7. Attack-chain summary

1. Enumerated the website and discovered `secret-script.php`.
2. Followed a message link and identified an unsafe file parameter.
3. Confirmed local file inclusion by retrieving a system file.
4. Investigated the login flow; a database credential was not valid for SSH.
5. Tried log poisoning and considered session poisoning; neither produced a usable path.
6. Used a PHP filter-chain technique to obtain command execution as `www-data`.
7. Obtained access as `comte`; the web-server account alone could not read `user.txt`.
8. Identified a writable systemd timer associated with a privileged service and reached root.

## 8. Defensive takeaways

- Replace user-controlled file paths with a strict server-side allowlist of resource identifiers.
- Canonicalize paths and verify that resolved files remain inside the intended content directory.
- Avoid dynamic inclusion of user-supplied values; disable or tightly constrain risky stream-wrapper behavior where practical.
- Use parameterized SQL queries and test authentication behavior beyond status codes and redirects.
- Keep application, database, and operating-system credentials separate; rotate any exposed credentials.
- Ensure unprivileged users cannot modify systemd unit or timer files. Set ownership and permissions so only administrators can change them.
- Review privileged services for unnecessary file operations and set-user-ID behavior.
- Audit SSH keys and home-directory permissions, and monitor changes to authorized-key files.
- Preserve scan outputs, request/response evidence, and complete flag values in authorized lab notes so results are reproducible without guessing.
