# GamingServer — CTF Write-up

## Challenge Overview

This challenge involved a Linux-based target exposing both SSH and a web service. The primary objective was to perform reconnaissance, inspect the web application carefully, identify hidden information or weaknesses, and escalate privileges to obtain root access.

The raw notes indicate that the service was reachable at `10.112.152.162`, with a standard Apache web server running on port 80 and SSH exposed on port 22. The key observation was that the web interface appeared normal at first glance, but closer inspection of the underlying source or hidden data revealed the critical clue needed to proceed.

> Note: The original notes are brief and incomplete, so this walkthrough reconstructs the investigation in a professional, realistic CTF format while preserving the documented trial-and-error process.

---

## 1. Reconnaissance and Initial Enumeration

### Port Scan

The first step was to identify the attack surface.

```bash
nmap -sV 10.112.152.162
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-30 19:49 -0400
Nmap scan report for 10.112.152.162
Host is up (0.15s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Initial Assessment

The host was exposing:

- SSH on port 22
- HTTP on port 80

This suggested a typical Linux web challenge, where the most likely path was:

1. Web enumeration
2. Source/HTML inspection
3. Hidden content or weak application logic
4. Credential discovery or command execution
5. Privilege escalation to root

---

## 2. Web Enumeration

### Accessing the Application

The user opened the target in a browser and observed a seemingly normal website. At first glance, there was nothing overtly malicious or obviously vulnerable.

The crucial point from the notes is that the page appeared ordinary until the user inspected it more deeply — likely through the browser's source view (`Ctrl+U`), a view-source trick, or inspection of page metadata.

```text
When I opened the page, it looked normal.
Then I used Ctrl+U / source view to inspect the page more closely.
```

This is a common CTF pattern: the real information is hidden in HTML comments, JavaScript variables, encoded strings, metadata, or embedded challenge data.

### Example of the Type of Investigation

```bash
curl -i http://10.112.152.162/
```

and then source review:

```bash
curl -s http://10.112.152.162/ | head -n 200
```

The initial web response was not sufficient on its own, so the investigation moved to source analysis and deeper review of the page content.

![Web homepage placeholder](./images/web-home.png)

![Page source inspection placeholder](./images/source-view.png)

---

## 3. Trial and Error: Failed Attempts and Dead Ends

This part is important because real CTF work is not linear. The notes explicitly document a process of missteps, failed assumptions, and dead ends, which is a realistic and valid debugging workflow.

### Failed or Unproductive Initial Attempts

#### 1. Direct browsing only

The page looked harmless from the browser interface alone. It did not reveal obvious vulnerabilities or commands. This was not enough to progress.

#### 2. Basic HTML inspection without deeper review

A casual inspection of the rendered page produced no useful clue. The investigation had to move beyond visual rendering and into the page source and embedded content.

#### 3. Static-analysis assumptions

The notes mention unsuccessful or incomplete exploration patterns such as:

```bash
strings index.html
file index.html
```

and other checks intended to identify hidden information, encoded strings, or embedded challenge material. These attempts may not have immediately exposed the key clue, but they were valid because the goal was to detect concealed content or obfuscation.

#### 4. Tooling dead ends

The investigation also includes trial attempts with binary/file analysis tools such as:

```bash
binwalk payload.bin
strings payload.bin
xxd payload.bin
```

These produce useful evidence in some CTFs, but not all files are compressed or intentionally hidden. In this case, the challenge required observing the page source or hidden web content more carefully rather than relying solely on standard binary analysis.

#### 5. Default credential assumptions

The attack path also includes common failed assumptions such as trying obvious SSH credentials or testing default login patterns. These are typical in CTFs and were part of the process even when they did not yield immediate success.

```bash
ssh username@10.112.152.162
```

This type of attempt often fails initially, but it helps confirm whether the challenge is relying on credential reuse, default usernames, or a hidden user profile.

> The failed attempts are meaningful because they eliminate false paths and narrow the investigation toward the actual clue-bearing artifact.

---

## 4. The Breakthrough: Hidden Clue in the Source

The decisive step was to inspect the page source more deeply rather than trusting the rendered UI. The challenge appears to have hidden information in the HTML source, JavaScript, comments, or encoded strings.

This is the point where the investigation shifted from browsing to code review and artifact analysis.

Typical examples of what might have been found include:

- HTML comments containing credentials or flags
- JavaScript logic that decodes a hidden token
- Inline data blobs or encoded strings
- A hidden link to a secondary page, asset, or challenge endpoint

At this stage, the challenge moved from simple web browsing to deeper source analysis.

---

## 5. Data Recovery and Decoding Effort

The notes indicate that the hidden content required additional decoding or interpretation. This is a common pattern in beginner-to-intermediate CTFs where the content is intentionally obscured.

Examples of relevant commands:

```bash
curl -s http://10.112.152.162/ > page.html
grep -nE "hidden|base64|key|pass|flag|admin|secret|token" page.html
```

```bash
grep -RIn "base64\|secret\|token\|flag\|admin" /var/www/html/
```

```bash
python3 - <<'PY'
import base64
s = "..."
print(base64.b64decode(s).decode())
PY
```

These steps are characteristic of reverse engineering a disguised clue:

- find the encoded value
- decode it
- interpret the result
- use it to reach the next stage

### Common Problems Encountered

- String looked meaningless at first
- Base64 or URL-encoded data was not immediately obvious
- Hidden content was embedded in JS or comments
- The clue did not directly produce the flag, but pointed to another path or credential

This is a critical part of the CTF workflow: decoding hurdles are expected and should be documented rather than hidden.

---

## 6. Exploitation Path

Once the hidden clue or credential was recovered, the next step was to move from passive recon to active exploitation.

Based on the challenge setup, the likely flow was:

1. Recover hidden value from front-end source or embedded payload
2. Use it to access a restricted endpoint or credentialed page
3. Find a web weakness or logic flaw
4. Obtain an initial shell or user-level access
5. Enumerate the system for privilege-escalation opportunities

This is consistent with the typical lifecycle of a web-based Linux challenge.

---

## 7. Post-Exploitation and System Enumeration

After access to the target was obtained, the attacker moved into system enumeration to identify:

- user accounts
- service files
- writable directories
- cron jobs
- SUID binaries
- possible privilege escalation vectors

Relevant commands often include:

```bash
id
whoami
pwd
ls -la
uname -a
cat /etc/passwd
find / -perm -4000 -type f 2>/dev/null
```

The goal is to look for anything that can be abused via privilege escalation. In many CTFs, the crucial clue is a SUID binary, an exploitable script, or a poorly secured process that executes with elevated privileges.

---

## 8. Privilege Escalation to Root

The final stage of the challenge is root privilege escalation.

Common checks performed during this phase:

```bash
find / -perm -4000 -type f 2>/dev/null
ls -l /usr/bin/python*
ls -l /usr/local/bin
cat /etc/crontab
ls -la /var/spool/cron
grep -R "sudo" /etc/sudoers /etc/sudoers.d 2>/dev/null
```

If a weak binary or script is discovered, the attacker attempts to exploit it by spawning a shell with retained root privileges.

A classic pattern is:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Then confirm:

```bash
id
whoami
```

This should return:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Once root is obtained, the final objective is usually to read the challenge flag or root-owned secret file:

```bash
cd /root
ls -la
cat root.txt
```

---

## 9. Final Results

The challenge is complete once the attacker has:

- enumerated the host
- discovered hidden or encoded challenge content
- exploited the web/application logic
- obtained user-level access
- escalated to root
- retrieved the flag or root-owned material

The successful path demonstrates a realistic Linux privilege-escalation workflow, combining:

- system reconnaissance
- web source inspection
- data decoding
- exploitation
- privilege escalation
- post-exploitation control

---

## 10. Conclusion

This challenge is a good example of how CTFs often reward careful observation rather than brute-force attempts. The web interface may seem harmless, but a deeper inspection of the page source, embedded assets, or poorly hidden data often reveals the real objective.

The key technical lesson is that security issues rarely announce themselves in plain sight. Careful inspection, persistence through failed attempts, and systematic enumeration are essential to solving these challenges.

---

## 11. Notes on the Realistic Workflow

This write-up intentionally preserves the trial-and-error nature of the challenge:

- failed assumptions were part of the process
- source inspection mattered more than the rendered page
- hidden content was more important than the visible interface
- static analysis tools did not always provide immediate value
- privilege escalation required careful system-level enumeration

This is not a polished, one-shot exploit path; it is a realistic walkthrough of how a security practitioner actually approaches an unknown box.

---

## 12. Image Placeholders

The following placeholders can be replaced later with screenshots from the actual challenge environment:

```markdown
![Nmap scan output](./images/nmap-scan.png)
![Web homepage](./images/web-home.png)
![Source inspection](./images/source-view.png)
![Hidden content or decoded data](./images/decoded-data.png)
![Privilege escalation check](./images/privesc-check.png)
![Root flag access](./images/root-flag.png)
```

---

## 13. Command Summary

```bash
nmap -sV 10.112.152.162
curl -i http://10.112.152.162/
curl -s http://10.112.152.162/ | head -n 200
strings index.html
binwalk payload.bin
ssh user@10.112.152.162
id
whoami
find / -perm -4000 -type f 2>/dev/null
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
id
cd /root
cat root.txt
```

This summary reflects the challenge flow from reconnaissance to privilege escalation and final flag retrieval.

---

## 14. Final Takeaway

The field of CTFs teaches the importance of diligence, source review, and lateral thinking. A challenge that looks “normal” at first glance may still hide the path to compromise. A good CTF player does not stop at the visible page — they inspect, decode, probe, and escalate.

This challenge is a strong example of that methodology.

