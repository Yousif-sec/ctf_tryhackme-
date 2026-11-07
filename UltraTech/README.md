# UltraTech — CTF Walkthrough

> **Scope and safety:** This write-up documents an authorized, isolated CTF lab. Run security testing only against systems you own or have explicit permission to assess. Host-specific exploit payloads, recovered passwords, and private-key material are intentionally redacted; they are secrets, not suitable for a repository write-up.

## Challenge overview

The challenge exposes FTP, SSH, and two web services. The path to root involves web-service enumeration, discovery of a vulnerable API behavior, local credential discovery, SSH access as a low-privileged user, and an unsafe Docker privilege boundary.

**Outcome:** root access was obtained in the lab. The supplied notes do not establish a separately verified flag value.

> **Credential hygiene:** The source notes contain recovered account credentials and a fragment of a root SSH private key. Those values are not reproduced here. Treat any exposed credentials or key material as compromised and rotate them wherever applicable.

## 1. Initial scan and service enumeration

An initial TCP service/version scan identified three listening services:

| Port | Service | Observation |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 on Ubuntu |
| 8081 | HTTP | Node.js / Express API |
| 31331 | HTTP | Apache-hosted site |

The scan reported the host as up and described the environment as Linux/Unix.

For an authorized lab, record service versions and investigate each exposed service independently. Keep the target value scoped to the lab assignment rather than copying a transient address into a reusable guide.

```text
Example workflow (replace the placeholder only in your authorized lab):
nmap -sS -sV <authorized-lab-host>
```

> **Image placeholder:** `[Insert screenshot of the initial service scan, with unrelated or sensitive details redacted.]`

## 2. FTP check — anonymous access failed

The FTP service was tested for anonymous access. The server requested a password and then rejected the login with `530 Login incorrect`.

This was a dead end: anonymous FTP access was not available. No files were obtained through this attempt, so the investigation moved on to the web services.

> **Image placeholder:** `[Insert screenshot of the rejected anonymous FTP login.]`

## 3. Web enumeration

Directory/content enumeration of the Apache site found `/robots.txt`. Its contents pointed to a sitemap at `/utech_sitemap.txt`. The sitemap listed `/`, `/index.html`, `/what.html`, and `/partners.html`.

The `/partners.html` page appeared to contain a login form. A form-focused SQL injection check was attempted, but it did not identify an injectable parameter or return database contents. This was another dead end; the login form alone did not reveal a way forward.

> **Image placeholder:** `[Insert redacted screenshots of robots.txt, the sitemap, and the partners page.]`

## 4. JavaScript review and API behavior

Further inspection of the site’s JavaScript revealed that the browser called an API on port `8081`, including a `/ping` route with an `ip` query parameter. A request to that route was observed in the browser’s network history.

Testing in the authorized lab showed that the API handled the supplied value unsafely and allowed server-side command execution. The observed response included the API process identity (`www`) and basic host information. This established the initial foothold in the challenge.

> **Finding:** User-controlled input reached a system command without safe argument handling. This is a command-injection vulnerability. Validate inputs against a strict allowlist and avoid invoking a shell; use safe process APIs with fixed arguments instead.
>
> **Redaction note:** The original notes include a working command-injection request. Its payload and replayable request are intentionally omitted.

> **Image placeholder:** `[Insert a screenshot showing the vulnerable request and a redacted response. Do not include reusable payloads, secrets, or unrelated host data.]`

## 5. Local enumeration and database discovery

With the API’s low-privileged execution context established, the challenge notes describe inspecting the application directory. It contained the Node.js application, its startup script, dependencies, and a SQLite database named `utech.db.sqlite`.

A first-pass text/string inspection of the SQLite file exposed schema-related text and account records. This was useful for identifying the database’s purpose, but string output is not a reliable substitute for properly parsing a database. The notes do not record a `binwalk` attempt or a separate decoding failure, so those steps are not invented here.

The discovered account values were MD5 password hashes. An offline lookup/cracking attempt recovered credentials for two accounts. The recovered secrets are deliberately not reproduced in this write-up.

> **Important:** Unsalted MD5 is unsuitable for password storage. Use a modern password-hashing scheme such as Argon2id, scrypt, or bcrypt, with unique salts and appropriate parameters.

> **Image placeholder:** `[Insert a redacted view of the database schema and findings. Remove usernames if required by your disclosure policy, and always remove hashes and plaintext passwords.]`

## 6. SSH access — one login failed, another succeeded

The recovered credentials were tested against SSH. The first attempted account, `admin`, was denied access. That failed login showed that a valid web/database credential did not necessarily correspond to an SSH-enabled account.

A second account from the database, `r00t`, did permit SSH access in the lab. The resulting shell was not root, but account and group enumeration showed membership in the `docker` group.

> **Image placeholder:** `[Insert a screenshot of the failed admin SSH attempt and the successful low-privilege session, with passwords and identifying details removed.]`

## 7. Privilege escalation through Docker access

The user’s Docker group membership was the key privilege-escalation clue. The host had a local `bash` image. In this challenge configuration, Docker access allowed a container to be started with the host filesystem exposed, and the container ran with root privileges. This crossed the host/container privilege boundary and provided access to root-owned files.

> **Finding:** Membership in the Docker group is effectively privileged on many host configurations. Treat Docker socket access and equivalent group membership as root-level access. Restrict membership, protect the daemon socket, and use rootless/container isolation options where appropriate.
>
> **Redaction note:** The host-mount command from the notes is intentionally not reproduced because it is a directly reusable privilege-escalation instruction.

> **Image placeholder:** `[Insert a redacted screenshot showing the privilege boundary and proof of root in the isolated lab. Do not include private-key contents.]`

## 8. Root SSH key and final evidence

After reaching root in the lab, the notes report locating the root user’s `.ssh` directory and reading `id_rsa`. The transcript only provides a short fragment of the private key and does not provide a complete, independently verifiable key or flag value.

Private SSH keys must never be published. If key material has been exposed, consider it compromised, replace it, and remove it from repository history as well as the current file.

## 9. Troubleshooting and dead ends (kept for completeness)

| Attempt | Result | What it established |
| --- | --- | --- |
| Anonymous FTP login | Rejected with `530 Login incorrect` | Anonymous access was unavailable. |
| SQL injection testing of the partners login form | No injection was identified | The login form was not the route used in this walkthrough. |
| SSH login as `admin` | Permission denied | The database credential did not grant SSH access for this account. |
| First-pass SQLite string inspection | Revealed schema and record text, but not a structured database view | Useful for discovery; use a proper SQLite reader in a controlled environment for reliable analysis. |
| `binwalk` or separate decoding attempts | Not present in the supplied notes | No outcome can be stated without evidence. |

## 10. Remediation summary

1. Replace shell-based command execution with safe APIs and strict input validation.
2. Restrict or remove access to the vulnerable API route; authenticate and authorize sensitive operations.
3. Rotate any credentials that were stored in the database or exposed during testing.
4. Replace unsalted MD5 password hashes with a dedicated modern password-hashing algorithm.
5. Limit SSH access to explicitly authorized accounts and use key-based authentication where suitable.
6. Remove unnecessary users from the Docker group and secure the Docker daemon/socket.
7. Protect private keys with restrictive permissions, rotate exposed keys, and prevent secrets from entering source control.

## Summary of the attack path

```text
Service discovery
  → web content enumeration
  → JavaScript reveals API route
  → unsafe API input handling yields a low-privilege foothold
  → local application/database discovery
  → credential reuse testing (first SSH login fails; second succeeds)
  → Docker group membership enables root access in the lab
```

**End state:** root access was achieved in the challenge environment. Sensitive credentials and private-key material have been excluded from this document.
