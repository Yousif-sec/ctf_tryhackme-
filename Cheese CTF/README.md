# Wonderland CTF Challenge Write-Up

## Overview

This write-up records the enumeration and initial access steps documented for the Wonderland machine. The notes identify SSH and HTTP, a hidden directory trail, a steganographic hint, and credentials for the `alice` account.

The supplied notes stop after identifying the SSH credentials and attempting to log in. They do **not** include the post-exploitation or root-escalation steps, so those sections are marked as incomplete rather than filled with unverified steps.

> **Lab scope:** Perform these steps only against an authorized CTF target. The target IP below is from the provided lab notes. The recovered password is redacted.

---

## 1. Initial Reconnaissance

I began with a service/version scan:

```bash
nmap -sV <target-ip>
```

### Results

```text
Nmap scan report for <target-ip>
Host is up (0.22s latency).
Not shown: 990 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Golang net/http server
```

The scan identified SSH on port `22` and an HTTP service on port `80`. I moved on to web enumeration.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

I used Feroxbuster to discover directories and common file types:

```bash
feroxbuster -u http://<target-ip>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt
```

The scan revealed a nested directory structure beginning with `/r`:

```text
/r/
/r/a/
/r/a/b/
/r/a/b/b/
```

The repeated directory names suggested that the route was a clue rather than an accidental directory listing. I noted the path and continued looking for the hint that would explain how to follow it.

![Image Placeholder: Feroxbuster directory results](./assets/web-enumeration.png)

---

## 3. Image Analysis and Steganography

I downloaded the image referenced by the site:

```bash
wget http://<target-ip>/img/white_rabbit_1.jpg
```

Then I attempted to extract embedded content with Steghide:

```bash
steghide extract -sf white_rabbit_1.jpg
```

Steghide prompted for a passphrase and successfully wrote the extracted file `hint.txt`. I read the hint:

```bash
cat hint.txt
```

```text
follow the r a b b i t
```

This clarified that the directory names found during enumeration should be followed in sequence: `r` → `a` → `b` → `b` → `i` → `t`.

> **Troubleshooting:** Steghide required a passphrase. The notes do not record a failed passphrase attempt or any `binwalk`/`strings` investigation, so no additional tool failures are inferred here.

![Image Placeholder: Extracted steganography hint](./assets/steganography-hint.png)

---

## 4. Following the Rabbit Trail

Using the hint, I continued through the discovered web directories in order:

```text
/r/a/b/b/i/t
```

The trail led to a page containing a clue for the user `alice` and a password. The password in the notes is truncated and is not reproduced here.

> **Credential note:** Do not publish recovered passwords in a repository. If a password has been exposed, consider it compromised and rotate it anywhere it may have been reused.

![Image Placeholder: Rabbit trail and redacted Alice credential clue](./assets/rabbit-trail.png)

---

## 5. SSH Login Attempt

The notes indicate that the discovered Alice credentials were used to try SSH access:

```bash
ssh alice@<target-ip>
```

The raw notes end at this point. They do not include the SSH result, a shell transcript, post-exploitation enumeration, or the steps used to obtain root. Therefore, I cannot accurately document whether this login succeeded or how privilege escalation was completed from the information provided so far.

![Image Placeholder: SSH session after confirming the result](./assets/ssh-access.png)

---

## 6. Post-Exploitation and Root Escalation

**Not documented in the supplied notes.** No user-shell output, privilege enumeration, root escalation commands, or root flag output were provided. Add the verified commands and terminal output here to complete the walkthrough; do not infer them from the challenge name alone.

![Image Placeholder: Add verified post-exploitation and root-shell evidence](./assets/root-access.png)

---

## 7. Trial-and-Error and Troubleshooting Notes

The notes provided document these observations:

1. The initial scan showed two open services, SSH and HTTP; it did not identify additional open ports.
2. Directory enumeration exposed the nested `r/a/b/b` path, but the complete `r/a/b/b/i/t` sequence became clear only after extracting the image hint.
3. Steghide prompted for a passphrase and then successfully extracted `hint.txt`.
4. No failed SSH login, `binwalk` attempt, `strings` output, or separate decoding failure was included in the notes. These are not added as events because there is no evidence they occurred.
5. The SSH login and privilege-escalation outcome remain unconfirmed in the supplied transcript.

---

## 8. Attack Chain Summary

1. Scanned the target and identified SSH and HTTP.
2. Enumerated the web server and found the nested `/r/a/b/b` directory path.
3. Downloaded `white_rabbit_1.jpg`.
4. Extracted `hint.txt` with Steghide.
5. Followed the hint through the `r` → `a` → `b` → `b` → `i` → `t` directory sequence.
6. Found a clue containing the username `alice` and a password (redacted).
7. Attempted SSH as `alice`; the supplied notes do not record the result or later escalation steps.

> **Current documented endpoint:** Discovery of Alice’s credentials and an SSH attempt. Add verified session and privilege-escalation evidence before describing this as a completed root compromise.

---

## Key Takeaways

- Directory names and media files can work together to provide an enumeration trail.
- Steganography tools may prompt for a passphrase before extracting embedded data.
- Keep a clear distinction between confirmed results and steps that have not yet been documented.
- Do not commit credentials or secrets to a public or shared repository.

---

![Image Placeholder: Wonderland challenge completion](./assets/final-result.png)
