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

> **Lab scope:** The commands and observations below relate to the authorized CTF environment described in the notes. Credentials and private-key contents have been omitted from this repository write-up.

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

### Unsafe API Input Handling

In the authorized lab, testing the `ip` parameter showed that user-controlled input was passed to a system command without safe handling. The response exposed command output and confirmed execution in the API’s low-privileged `www` context.

> **Finding:** This is command injection caused by invoking a system command with untrusted input. The specific working payload is omitted from this write-up. In a real application, avoid shell execution for user input and validate any required values against a strict allowlist.

![Image Placeholder: Redacted API request and response](./assets/api-command-injection.png)

---

## 5. Application and Database Enumeration

The API process ran from a Node.js application directory. Directory enumeration showed the application files and a SQLite database:

```text
index.js
node_modules/
package.json
package-lock.json
start.sh
utech.db.sqlite
```

I used a basic string inspection of the SQLite file to identify its schema and contents. This was an initial discovery step rather than a structured database review. The schema showed a `users` table with login, password, and type fields:

```sql
CREATE TABLE users (
    login Varchar,
    password Varchar,
    type Int
)
```

The records contained password hashes. I identified them as MD5 hashes and used an offline lookup/cracking attempt to recover two passwords. The recovered usernames and passwords are not repeated here.

> **Password storage note:** Unsalted MD5 is not suitable for password storage. Use a dedicated password-hashing algorithm such as Argon2id, scrypt, or bcrypt, with unique salts and appropriate work factors.

![Image Placeholder: Redacted database schema and findings](./assets/sqlite-findings.png)

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

After obtaining root access, I inspected `/root`. The directory contained `private.txt`, a `.ssh` directory, and the root SSH key files:

```text
.ssh/
  authorized_keys
  id_rsa
  id_rsa.pub
```

The challenge question asked for the first nine characters of the private key. The notes include a key fragment, but private-key material is intentionally not reproduced in this public write-up. Do not commit private keys or fragments to a repository; rotate any key that has been exposed.

![Image Placeholder: Root directory listing with sensitive content redacted](./assets/root-directory.png)

---

## 9. Trial-and-Error Notes

The following unsuccessful attempts and troubleshooting steps were part of the investigation:

1. **Anonymous FTP:** The server rejected the anonymous login with `530 Login incorrect`; no FTP files were available.
2. **SQL injection on the partners login form:** SQLmap did not find an injection or dump records. The database was discovered later through the API foothold, not through the login form.
3. **SSH as `admin`:** Login was denied even though an account/hash had been found in the database. Testing another recovered account led to SSH access.
4. **Database inspection:** `strings` provided a quick view of schema and record text, but it was not a complete or structured database analysis. A SQLite-aware tool would be the more reliable way to inspect a copy of the database.
5. **Tool attempts not evidenced in the notes:** The supplied notes do not show a `binwalk` run or a separate decoding failure, so no result is attributed to either.

---

## 10. Attack Chain Summary

1. Scanned the target and identified FTP, SSH, and two HTTP services.
2. Tried anonymous FTP; authentication was rejected.
3. Enumerated the Apache site and found `robots.txt` and the sitemap.
4. Inspected `/partners.html`; SQLmap did not find SQL injection.
5. Reviewed `/js/api.js` and discovered the API on port `8081`.
6. Identified unsafe handling of the API’s `ip` parameter and obtained a low-privilege foothold in the lab.
7. Inspected the application directory and found `utech.db.sqlite`.
8. Recovered credentials from hashes; the `admin` SSH attempt failed, while SSH access as `r00t` succeeded.
9. Found that `r00t` belonged to the Docker group and used the unsafe Docker privilege boundary to reach root.
10. Located the root SSH key files; the key contents are omitted from this write-up.

> **Final escalation path:** API command injection → application database credential discovery → SSH as `r00t` → Docker group access → root.

---

## 11. Key Takeaways

- Inspect JavaScript and browser network history when a web application communicates with a separate API.
- SQL injection testing may fail even when sensitive database information is accessible through a different weakness.
- Treat credentials recovered from an application database as sensitive, and verify account scope only in an authorized environment.
- Untrusted input must never be passed directly to a shell command.
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
