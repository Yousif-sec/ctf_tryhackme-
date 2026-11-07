# Boiler CTF Challenge Write-Up

## Overview

This walkthrough documents the Boiler CTF machine, from initial service discovery to root access. The challenge combines anonymous FTP enumeration, a ROT13 clue, a Webmin rabbit hole, command injection in `sar2html`, credential discovery in a log file, SSH access, and a SUID `find` privilege-escalation issue.

The attack chain involved:

- Enumerating FTP, HTTP, Webmin, and SSH services
- Finding and decoding a clue in anonymous FTP
- Investigating Webmin without obtaining access
- Discovering `sar2html` and unsafe handling of its `plot` parameter
- Finding credentials in `log.txt` and in a backup script
- Moving from `basterd` to `stoner`
- Identifying a root-owned SUID `find` binary and obtaining root in the lab

> **Lab scope:** This is a write-up of an authorized CTF target. Replayable command-injection and privilege-escalation payloads, and plaintext passwords, are redacted. Replace `<target-ip>` only with the address assigned to your own lab instance.

---

## 1. Initial Reconnaissance

### Port Scanning

I started by scanning the target for open TCP services and their versions:

```bash
nmap -sS -sV <target-ip>
```

### Results

```text
Nmap 7.99 scan report for <target-ip>
Host is up (0.10s latency).
Not shown: 997 closed tcp ports (reset)

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
55007/tcp open  ssh     OpenSSH 7.2p2
Service Info: OS: Unix
```

The unusual SSH port was `55007`. I also noted the Webmin service on port `10000`, but began with FTP because it allowed an anonymous login.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. FTP Enumeration and ROT13 Clue

I connected to FTP and tried anonymous access. Unlike some targets, anonymous login succeeded:

```text
ftp <target-ip>
Name: anonymous
230 Login successful.
```

The directory contained a hidden-looking file named `.info.txt`. I downloaded and read it:

```text
ftp> get .info.txt
```

```text
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

The text was encoded with ROT13. Decoding it produced:

```text
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

The message did not provide credentials or a direct foothold, but it emphasized continuing with thorough enumeration.

![Image Placeholder: Anonymous FTP and decoded note](./assets/ftp-rot13-note.png)

---

## 3. Webmin Investigation — Dead End

Port `10000` hosted a Webmin login page. I tried default credentials, but the login failed.

I also tried several avenues against the login page:

1. SQLmap did not find a useful SQL injection.
2. I used FFUF with a login-bypass wordlist and a captured request:

   ```bash
   ffuf -request test2.txt \
     -request-proto https \
     -w /usr/share/wordlists/seclists/Fuzzing/login_bypass.txt
   ```

   This did not lead to useful access. The notes mention a vulnerability result but no successful session or actionable finding; Webmin remained a rabbit hole for this attempt.

I moved on to the Apache site on port `80`.

> **Troubleshooting note:** The Webmin service looked promising because of its version and login page, but the tests performed did not provide a foothold. Enumerating other exposed services was more productive.

![Image Placeholder: Webmin login and unsuccessful tests](./assets/webmin-dead-end.png)

---

## 4. Web Enumeration and `sar2html` Discovery

I enumerated the Apache site with Feroxbuster:

```bash
feroxbuster -u http://<target-ip>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html
```

The scan found a redirect to a test application under `/joomla/_test/`:

```text
301 GET /joomla/_test => /joomla/_test/
```

Further inspection identified the application as `sar2html`. The page used a `plot` parameter, which I investigated in the authorized lab. Testing showed unsafe command handling through this parameter; a basic identity check returned the web-service account, `www-data`.

> **Finding:** The `plot` parameter was vulnerable to command injection. The exact request and command payload are omitted, but the lab response showed execution as `www-data`.

![Image Placeholder: `sar2html` page and redacted test result](./assets/sar2html.png)

---

## 5. Post-Exploitation Enumeration and Log Discovery

With command execution as the web-service user established, I enumerated the application directory. Listing its contents revealed `log.txt`. Reading that file exposed SSH-related log entries, including a password accidentally recorded in a successful-login message.

The relevant log lines indicated:

- A password-authenticated SSH login for the account `basterd`
- A session-open message referring to `pentest`

The notes then used the account `basterd` for SSH access. The exposed password is omitted here.

> **Credential exposure:** Passwords should never be written to authentication logs. Treat the exposed value as compromised and rotate it.

![Image Placeholder: Redacted `log.txt` output](./assets/log-discovery.png)

---

## 6. SSH Access as `basterd`

I used the recovered credential to connect to SSH on the non-standard port:

```bash
ssh basterd@<target-ip> -p 55007
```

After logging in, I inspected the home directory:

```bash
ls -la
```

Relevant output:

```text
drwxr-x--- 3 basterd basterd 4096 Aug 22  2019 .
-rwxr-xr-x 1 stoner  basterd  699 Aug 21  2019 backup.sh
-rw------- 1 basterd basterd    0 Aug 22  2019 .bash_history
```

The `backup.sh` script was owned by `stoner` and readable by `basterd`, so I inspected it:

```bash
cat backup.sh
```

The script contained backup logic and, in a comment, another credential. That credential was associated with `stoner` in the challenge notes and is redacted here.

![Image Placeholder: `basterd` SSH session and redacted backup script](./assets/basterd-session.png)

---

## 7. Moving from `basterd` to `stoner`

Using the credential found in the backup script, I switched to the `stoner` account:

```bash
su stoner
```

This provided a second local user context. I then checked for SUID binaries:

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

The output included a `find` binary owned by root with the SUID bit set:

```text
-r-sr-xr-x 1 root root ... /usr/bin/find
```

This was the key privilege-escalation finding.

![Image Placeholder: SUID enumeration showing the `find` binary](./assets/suid-find.png)

---

## 8. Privilege Escalation to Root

The root-owned SUID `find` binary could be abused to run a shell with elevated effective privileges. In the lab, this resulted in a shell where `id` showed `euid=0(root)` while the real user remained `stoner`.

> **Redaction note:** The exact SUID shell-spawning command is omitted because it is directly reusable outside the challenge. The security issue is that an unnecessary SUID bit was set on a general-purpose utility. Remove the SUID bit or replace the binary with a properly restricted mechanism.

![Image Placeholder: Redacted root-shell verification](./assets/root-shell.png)

---

## 9. Flag Retrieval

After gaining root privileges, I read the challenge files. The user-level file in the notes was named `.secret`:

```text
You made it till here, well done.
```

The root flag file contained:

```text
It wasn't that hard, was it?
```

![Image Placeholder: Challenge completion](./assets/final-result.png)

---

## 10. Trial-and-Error Notes

The unsuccessful attempts and dead ends are part of the workflow:

1. **Anonymous FTP access worked, but did not immediately give a foothold.** The only file was `.info.txt`; decoding its ROT13 message gave an enumeration hint rather than credentials.
2. **Webmin default credentials failed.** The service had a login page, but the default username/password guess was incorrect.
3. **SQLmap against Webmin was unproductive.** No useful SQL injection was found.
4. **FFUF login-bypass testing did not produce useful access.** The notes mention a vulnerability result, but there was no successful login or follow-up evidence; it remained a dead end.
5. **The SSH log contained inconsistent account details.** One line recorded a login for `basterd`, while another session message referred to `pentest`. The working path in the notes was SSH as `basterd` on port `55007`.
6. **No `binwalk` or separate decoding-tool failure is recorded in the supplied notes.** The only documented decoding step was recognizing and decoding the FTP message with ROT13.

---

## 11. Attack Chain Summary

1. Scanned the target and identified FTP, HTTP, Webmin, and SSH on port `55007`.
2. Logged in to FTP anonymously and downloaded `.info.txt`.
3. Decoded the ROT13 note, which pointed to enumeration.
4. Tried Webmin default credentials, SQLmap, and FFUF login-bypass testing; these did not produce access.
5. Enumerated the Apache site and found `sar2html` under `/joomla/_test/`.
6. Identified command injection through the `plot` parameter and confirmed execution as `www-data`.
7. Listed the application files and read `log.txt`, finding an exposed SSH credential.
8. Logged in as `basterd` over SSH on port `55007`.
9. Read `backup.sh` and found a credential for `stoner`.
10. Switched to `stoner` and discovered a root-owned SUID `find` binary.
11. Abused the SUID misconfiguration to obtain an effective root shell and read the challenge files.

> **Final escalation path:** `sar2html` command injection → credential discovery in `log.txt` → SSH as `basterd` → `stoner` credential in `backup.sh` → SUID `find` → root.

---

## 12. Key Takeaways

- Anonymous FTP can expose useful clues even when it does not contain a direct foothold.
- ROT13 and other simple encodings may hide hints, so inspect downloaded files rather than dismissing them.
- Do not stop enumerating when a promising service such as Webmin becomes a dead end.
- Never log plaintext passwords.
- Review scripts and configuration files for accidentally embedded credentials.
- General-purpose binaries should not be given SUID permissions without a strict operational need.

---

![Image Placeholder: Boiler CTF exploitation path](./assets/attack-chain.png)
