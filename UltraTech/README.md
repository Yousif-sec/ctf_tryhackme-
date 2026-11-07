# UltraTech CTF Challenge Write-Up

## Overview

This walkthrough documents the UltraTech challenge, starting with service enumeration and web reconnaissance and ending with root access through Docker group membership.

The attack chain involved:

- Enumerating FTP, SSH, and web services
- Finding the web sitemap and reviewing the partner login page
- Discovering an API endpoint referenced by the site’s JavaScript
- Identifying unsafe command handling in the API
- Finding password hashes in the application’s SQLite database
- Testing recovered credentials against SSH
- Using the `docker` group privilege boundary to obtain root in the lab

> **Lab scope:** This write-up describes the authorized CTF environment from the notes. Replayable command-injection requests, recovered passwords, and private-key contents are redacted.

---

## 1. Initial Reconnaissance

### Port Scanning

I started by checking the target for open TCP services and identifying their versions:

```bash
nmap -sS -sV <target-ip>
```

### Results

```text
Nmap scan report for <target-ip>
Host is up (0.23s latency).
Not shown: 997 closed tcp ports (reset)

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.5
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8081/tcp  open  http    Node.js Express framework
31331/tcp open  http    Apache
```

The services exposed two likely web entry points, along with FTP and SSH. I focused first on the Apache site on port `31331`.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. FTP Enumeration

I checked whether FTP allowed anonymous access:

```text
ftp <target-ip>
Name: anonymous
Password:
530 Login incorrect.
ftp: Login failed
```

Anonymous access was rejected, so FTP did not provide an initial foothold. I moved on to the web server.

![Image Placeholder: Anonymous FTP login failure](./assets/ftp-anonymous-failure.png)

---

## 3. Web Enumeration

### Directory Discovery

I used Feroxbuster to look for common files and directories on the Apache site:

```bash
feroxbuster -u http://<target-ip>:31331/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

The scan found `/robots.txt`:

```text
200 GET http://<target-ip>:31331/robots.txt
```

### `robots.txt` and Sitemap

The file contained:

```text
Allow: *
User-Agent: *
Sitemap: /utech_sitemap.txt
```

I opened the sitemap and found these paths:

```text
/
/index.html
/what.html
/partners.html
```

The `/partners.html` page contained a login form.

![Image Placeholder: robots.txt and sitemap](./assets/robots-sitemap.png)

### Failed SQL Injection Test

I tested the login page for SQL injection using SQLmap:

```bash
sqlmap -u 'http://<target-ip>:31331/partners.html' --forms --batch --dump
```

This did not identify a usable SQL injection or return database contents. The login form was a dead end, so I continued enumerating the site’s client-side files.

![Image Placeholder: Partners login page](./assets/partners-login.png)

---

## 4. JavaScript Review and API Discovery

The site’s JavaScript was located at `/js/api.js`. Its relevant logic showed that the browser called an API on port `8081` and sent the hostname to a `/ping` endpoint:

```javascript
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
        return `${window.location.hostname}:8081`;
    }

    function checkAPIStatus() {
        const req = new XMLHttpRequest();
        try {
            const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`;
            req.open('GET', url, true);
            // Remaining status-handling code omitted.
        } catch (e) {
            // Error handling omitted.
        }
    }
})();
```

I reviewed the browser’s HTTP history and confirmed a request to the API:

```http
GET /ping?ip=<target-host> HTTP/1.1
Host: <target-host>:8081
```

### Testing the API

In the authorized lab, I tested how the `/ping` endpoint handled the `ip` parameter. The notes show that an encoded line break followed by a basic command caused the API to return command output, confirming that the parameter was passed to a system command unsafely.

> **Redaction note:** The original request included a command-injection separator and commands. The exact replayable request is omitted here; the observed steps were to check the process identity and list the application directory.

The response identified the process as `www`. The directory listing showed the Node.js application, dependencies, startup script, and SQLite database. This confirmed a low-privileged foothold in the API’s execution context and provided the next lead: `utech.db.sqlite`.

> **Finding:** User-controlled input reached a system command without safe argument handling. In a real application, avoid shell execution for user input and validate required values against a strict allowlist.

![Image Placeholder: Redacted API request and response](./assets/api-command-injection.png)

---

## 5. Application and Database Enumeration

The API process ran from a Node.js application directory. The listing obtained during the API testing showed:

```text
index.js
node_modules/
package.json
package-lock.json
start.sh
utech.db.sqlite
```

I then used `strings` against the SQLite file to get a quick view of its contents. In the lab, this was done through the same `/ping` API behavior. The exact command-injection request is omitted, but the inspected file was `utech.db.sqlite`.

The string output included the SQLite signature, a `users` table schema, and account records:

```text
SQLite format 3
CREATE TABLE users (
    login Varchar,
    password Varchar,
    type Int
)

<account records and password hashes redacted>
```

The hashes were identified as MD5. I used an online hash lookup for quick testing; John the Ripper is another option for authorized offline password recovery. Two account passwords were recovered, but the hashes and plaintext passwords are not repeated here.

> **Password storage note:** Unsalted MD5 is not suitable for password storage. Use a dedicated password-hashing algorithm such as Argon2id, scrypt, or bcrypt, with unique salts and appropriate work factors.

> **Troubleshooting:** `strings` was useful for quickly spotting the schema and records, but it is not a structured way to query a database. A SQLite-aware tool would be more reliable for inspecting a local copy.

![Image Placeholder: Redacted SQLite string output](./assets/sqlite-findings.png)

---

## 6. SSH Credential Testing

I first tried the recovered `admin` account over SSH. Authentication failed:

```text
Permission denied
```

This was an important failed attempt: credentials found in the application database did not necessarily provide SSH access for every account.

I then tested the other recovered account, `r00t`, and was able to log in. The account was not root. Initial identity and group enumeration showed membership in the `docker` group:

```text
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

![Image Placeholder: Failed admin login and successful SSH session](./assets/ssh-access.png)

---

## 7. Privilege Escalation Through Docker

After logging in, I checked the available Docker images:

```text
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
bash         latest    <image-id>      7 years ago   15.8MB
```

The significant finding was not the image itself, but the user’s Docker group membership. In this challenge configuration, Docker access allowed a container to access the host filesystem with root privileges, crossing the container/host privilege boundary.

> **Privilege escalation finding:** Access to the Docker daemon is effectively root-equivalent on many systems. Restrict Docker group membership and protect access to the daemon socket. The exact host-mount invocation is omitted here.

The root context was confirmed in the lab:

```text
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

![Image Placeholder: Redacted proof of root in the lab](./assets/root-access.png)

---

## 8. Root Directory and SSH Key

After obtaining root access, I navigated to the mounted root user’s home directory and listed its contents:

```text
cd /mnt/root/root/
ls
private.txt  snap
```

I then checked the directory, including hidden files:

```text
ls -la
```

The listing included a `.ssh` directory. Inside it, I found the root SSH key files:

```text
authorized_keys
id_rsa
id_rsa.pub
```

The challenge question asked for the first nine characters of the private key. The notes include a key fragment, but private-key material is intentionally not reproduced in this repository. Do not commit private keys or fragments; rotate any key that has been exposed.

![Image Placeholder: Root directory listing with sensitive content redacted](./assets/root-directory.png)

---

## 9. Trial-and-Error Notes

The following unsuccessful attempts and troubleshooting steps were part of the investigation:

1. **Anonymous FTP:** The server rejected the anonymous login with `530 Login incorrect`; no FTP files were available.
2. **SQL injection on the partners login form:** SQLmap did not find an injection or dump records. The database was discovered later through the API foothold, not through the login form.
3. **SSH as `admin`:** Login was denied even though an account/hash had been found in the database. Testing another recovered account led to SSH access.
4. **Database inspection:** `strings` provided a quick view of schema and record text, but it was not a complete or structured database analysis. A SQLite-aware tool would be a more reliable way to query the database.
5. **Tool attempts not evidenced in the notes:** The supplied notes do not show a `binwalk` run or a separate decoding failure, so no result is attributed to either.

---

## 10. Attack Chain Summary

1. Scanned the target and identified FTP, SSH, and two HTTP services.
2. Tried anonymous FTP; authentication was rejected.
3. Enumerated the Apache site and found `robots.txt` and the sitemap.
4. Inspected `/partners.html`; SQLmap did not find SQL injection.
5. Reviewed `/js/api.js` and discovered the API on port `8081`.
6. Tested the API’s `ip` parameter and confirmed unsafe command handling.
7. Listed the application directory and found `utech.db.sqlite`.
8. Used `strings` to identify the SQLite schema and MD5 password hashes; recovered credentials were tested against SSH.
9. The SSH login as `admin` failed; SSH access as `r00t` succeeded.
10. Found that `r00t` belonged to the Docker group and used the unsafe Docker privilege boundary to reach root.
11. Navigated to `/mnt/root/root/` and located the root SSH key files; key contents are omitted.

> **Final escalation path:** API command injection → application/database discovery → SSH as `r00t` → Docker group access → root.

---

## 11. Key Takeaways

- Inspect JavaScript and browser network history when a web application communicates with a separate API.
- SQL injection testing may fail even when sensitive database information is accessible through a different weakness.
- Untrusted input must never be passed directly to a shell command.
- Unsalted MD5 hashes can be recovered quickly and should not be used for passwords.
- Docker daemon access should be treated as highly privileged and restricted accordingly.
- Never publish recovered passwords, password hashes, or private SSH key material.

---

## Useful Enumeration Commands

The following commands cover the non-secret discovery steps documented above. Replace `<target-ip>` only with the address assigned to your authorized lab:

```bash
nmap -sS -sV <target-ip>
feroxbuster -u http://<target-ip>:31331/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
curl http://<target-ip>:31331/robots.txt
curl http://<target-ip>:31331/utech_sitemap.txt
curl http://<target-ip>:31331/js/api.js
sqlmap -u 'http://<target-ip>:31331/partners.html' --forms --batch --dump
ssh admin@<target-ip>
ssh r00t@<target-ip>
```

---

![Image Placeholder: UltraTech challenge completion](./assets/final-result.png)
