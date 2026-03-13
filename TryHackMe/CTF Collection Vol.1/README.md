# CTF Write-Up: [CTF Collection Vol.1](https://tryhackme.com/room/ctfcollectionvol1)

> **Platform:** TryHackMe | **Difficulty:** Easy

---

## 🚩 Flags

<details>
<summary>Task 02</summary>

`THM{ju57_d3c0d3_7h3_b453}`

</details>

<details>
<summary>Task 03</summary>

`THM{3x1f_0r_3x17}`

</details>

<details>
<summary>Task 04</summary>

`THM{500n3r_0r_l473r_17_15_0ur_7urn}`

</details>

<details>
<summary>Task 05</summary>

`THM{wh173_fl46}`

</details>

<details>
<summary>Task 06</summary>

`THM{qr_m4k3_l1f3_345y}`

</details>

<details>
<summary>Task 07</summary>

`THM{345y_f1nd_345y_60}`

</details>

<details>
<summary>Task 08</summary>

`THM{17_h45_l3553r_l3773r5}`

</details>

<details>
<summary>Task 09</summary>

`THM{hail_the_caesar}`

</details>

<details>
<summary>Task 10</summary>

`THM{4lw4y5_ch3ck_7h3_c0m3mn7}`

</details>

<details>
<summary>Task 11</summary>

`THM{y35_w3_c4n}`

</details>

<details>
<summary>Task 12</summary>

`THM{50c141_4cc0un7_15_p4r7_0f_051n7}`

</details>

<details>
<summary>Task 13</summary>

`THM{0h_my_h34d}`

</details>

<details>
<summary>Task 14</summary>

`THM{3xclu51v3_0r}`

</details>

<details>
<summary>Task 15</summary>

`THM{y0u_w4lk_m3_0u7}`

</details>

<details>
<summary>Task 16</summary>

`THM{7h3r3_15_h0p3_1n_7h3_d4rkn355}`

</details>

<details>
<summary>Task 17</summary>

`THM{SOUNDINGQR}`

</details>

<details>
<summary>Task 18</summary>

`THM{ch3ck_th3_h4ckb4ck}`

</details>

<details>
<summary>Task 19</summary>

`TRYHACKME{YOU_FOUND_THE_KEY}`

</details>

<details>
<summary>Task 20</summary>

`THM{17_ju57_4n_0rd1n4ry_b4535}`

</details>

<details>
<summary>Task 21</summary>

`THM{d0_n07_574lk_m3}`

</details>

---

## Task 01 — Welcome

No flag required. A simple acknowledgement task to kick off the room.

---

## Task 02 — Base64 Decode

**Challenge:** Decode `VEhNe2p1NTdfZDNjMGQzXzdoM19iNDUzfQ==`

The trailing `==` padding is a classic Base64 tell. Decoded directly in CyberChef using **From Base64**.

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true,false)&input=VEhNe2p1NTdfZDNjMGQzXzdoM19iNDUzfQ==)

> 🚩 `THM{ju57_d3c0d3_7h3_b453}`

---

## Task 03 — Metadata

**Challenge:** "Meta! meta! meta! meta..." — hinting at image metadata.

`exiftool` was run against the provided image:

```bash
exiftool Find_me_1577975566801.jpg
```

The flag was embedded directly in the **Owner Name** field:

```
Owner Name : THM{3x1f_0r_3x17}
```

> 🚩 `THM{3x1f_0r_3x17}`

---

## Task 04 — Steganography

**Challenge:** "Something is hiding. That's all you need to know."

`steghide` was used to extract hidden data from the provided image with no passphrase:

```bash
steghide extract -sf Extinction_1577976250757.jpg
# Enter passphrase: (blank)
cat Final_message.txt
```

```
It's going to be over soon. Sleep my child.

THM{500n3r_0r_l473r_17_15_0ur_7urn}
```

> 🚩 `THM{500n3r_0r_l473r_17_15_0ur_7urn}`

---

## Task 05 — White Text

**Challenge:** "Huh, where is the flag?"

The flag was hidden in the task description using **white-coloured text** — invisible in light mode. Selecting all text or switching to dark mode revealed it.

> 🚩 `THM{wh173_fl46}`

---

## Task 06 — QR Code

**Challenge:** Task file contained a QR code.

Scanned using Google Lens. The QR code decoded directly to the flag.

> 🚩 `THM{qr_m4k3_l1f3_345y}`

---

## Task 07 — Strings on a Binary

**Challenge:** Task file was a compiled ELF binary with a `.hello` extension.

`cat` produced unreadable output. `strings` extracted all printable text:

```bash
strings hello_1577977122465.hello | grep THM
THM{345y_f1nd_345y_60}
```

> 🚩 `THM{345y_f1nd_345y_60}`

---

## Task 08 — Base58 Decode

**Challenge:** Decode `3agrSy1CewF9v8ukcSkPSYm3oKUoByUpKG4L`

The character set (no `0`, `O`, `I`, `l`) identified this as **Base58** — the encoding used by Bitcoin addresses. Decoded in CyberChef using **From Base58**.

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=From_Base58('123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz',true)&input=M2FnclN5MUNld0Y5djh1a2NTa1BTWW0zb0tVb0J5VXBLRTRM)

> 🚩 `THM{17_h45_l3553r_l3773r5}`

---

## Task 09 — Caesar Cipher (ROT7)

**Challenge:** Decode `MAF{atbe_max_vtxltk}`

ROT13 was the obvious guess but produced garbage. Trying different rotation values in CyberChef with **ROT13** set to **7** yielded the flag.

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=ROT13(true,true,false,7)&input=TUFGE2F0YmVfbWF4X3Z0eGx0a30)

> 🚩 `THM{hail_the_caesar}`

---

## Task 10 — Page Source

**Challenge:** "No downloadable file, no ciphered or encoded text."

Pressing **F12** to open developer tools and searching for `THM` in the page source revealed the flag hidden in a `<span>` element:

```html
<span data-testid="glossary-term" class="glossary-term">THM</span>{4lw4y5_ch3ck_7h3_c0m3mn7}
```

> 🚩 `THM{4lw4y5_ch3ck_7h3_c0m3mn7}`

---

## Task 11 — Corrupted PNG Header

**Challenge:** "I accidentally messed up with this PNG file. Can you help me fix it?"

`exiftool` and `file` both confirmed the image was corrupt:

```bash
exiftool spoil_1577979329740.png
# Error: File format error

file spoil_1577979329740.png
# spoil_1577979329740.png: data
```

Opening in **GHex** revealed the file started with `23 33 44 5F` (`#3D_`) instead of the correct PNG magic bytes. A valid PNG header must begin with:

```
89 50 4E 47 0D 0A 1A 0A
```

The first four bytes were corrected in GHex:

| Offset | Wrong | Correct |
|--------|-------|---------|
| 0x00 | `23` | `89` |
| 0x01 | `33` | `50` |
| 0x02 | `44` | `4E` |
| 0x03 | `5F` | `47` |

After saving, the file was confirmed valid:

```bash
file spoil_1577979329740.png
# spoil_1577979329740.png: PNG image data, 800 x 800, 8-bit/color RGBA, non-interlaced
```

Opening the repaired image revealed the flag.

> 🚩 `THM{y35_w3_c4n}`

---

## Task 12 — OSINT

**Challenge:** "Some hidden flag inside TryHackMe social account."

OSINT led to a Reddit post at `r/tryhackme` announcing a new room. The original post and account have since been deleted — had a good laugh trying to find it.

> 🚩 `THM{50c141_4cc0un7_15_p4r7_0f_051n7}`

---

## Task 13 — Brainfuck

**Challenge:** A wall of `+`, `>`, `<`, `-`, `.` characters.

Recognised immediately as **Brainfuck** — an esoteric programming language that uses only 8 characters. The code was run at [copy.sh/brainfuck](https://copy.sh/brainfuck/):

```
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>++++++++++++++.------------.+++++.>+++++++++++++++++++++++.<<++++++++++++++++++.>>-------------------.---------.++++++++++++++.++++++++++++.<++++++++++++++++++.+++++++++.<+++.+.>----.>++++.
```

Output: `THM{0h_my_h34d}`

> 🚩 `THM{0h_my_h34d}`

---

## Task 14 — XOR

**Challenge:** XOR the following two hex strings:

```
S1: 44585d6b2368737c65252166234f20626d
S2: 1010101010101010101010101010101010
```

S2 appeared to be binary at first glance, but it was actually a **repeating hex key** (`0x10`). Both strings were XOR'd in CyberChef using **From Hex → XOR** with key `10` (Hex):

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=From_Hex('Auto')XOR(%7B'option':'Hex','string':'10'%7D,'Standard',false)&input=NDQ1ODVkNmIyMzY4NzM3YzY1MjUyMTY2MjM0ZjIwNjI2ZA)

> 🚩 `THM{3xclu51v3_0r}`

---

## Task 15 — Binwalk File Extraction

**Challenge:** "Please exfiltrate my file :)"

`binwalk` was used to analyse the provided image for embedded files:

```bash
binwalk -e hell_1578018688127.jpg
```

```
DECIMAL       HEXADECIMAL     DESCRIPTION
265845        0x40E75         Zip archive data ... name: hello_there.txt
```

A ZIP archive was embedded inside the JPG. The extracted text file contained the flag:

```bash
cat _hell_1578018688127.jpg.extracted/hello_there.txt
# Thank you for extracting me, you are the best!
# THM{y0u_w4lk_m3_0u7}
```

> 🚩 `THM{y0u_w4lk_m3_0u7}`

---

## Task 16 — LSB Steganography

**Challenge:** "There is something lurking in the dark."

The image was opened in **Stegsolve** and the colour planes were cycled through using the arrow buttons. The flag appeared clearly on **Blue plane 1**:

```bash
java -jar Stegsolve.jar
```

> 🚩 `THM{7h3r3_15_h0p3_1n_7h3_d4rkn355}`

---

## Task 17 — QR Code + Audio

**Challenge:** "How good is your listening skill?" — flag formatted as `THM{Listened Flag}` in ALL CAPS.

The task file contained a QR code which decoded to a SoundCloud link:
`https://soundcloud.com/user-86667759/thm-ctf-vol1`

Listening to the audio track revealed the spoken flag.

> 🚩 `THM{SOUNDINGQR}`

---

## Task 18 — Wayback Machine

**Challenge:** Find a past version of `https://www.embeddedhacker.com/` from 2 January 2020.

The Wayback Machine was used to navigate to the archived snapshot:

```
https://web.archive.org/web/20200102131252/https://www.embeddedhacker.com/
```

Scrolling through the archived page revealed the flag embedded in a post.

> 🚩 `THM{ch3ck_th3_h4ckb4ck}`

---

## Task 19 — Vigenère Cipher

**Challenge:** Decode `MYKAHODTQ{RVG_YVGGK_FAL_WXF}` — the flag format is `TRYHACKME{FLAG IN ALL CAPS}`.

The known plaintext `TRYHACKME` against the ciphertext `MYKAHODTQ` was enough to deduce the key. Decoding in CyberChef with **Vigenère Decode**, key `THM`:

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=Vigen%C3%A8re_Decode('THM')&input=TVlLQUhPRFRRe1JWR19ZVkdHS19GQUxfV1hGfQo)

> 🚩 `TRYHACKME{YOU_FOUND_THE_KEY}`

---

## Task 20 — Decimal to Hex to ASCII

**Challenge:** Decode the following number:

```
581695969015253365094191591547859387620042736036246486373595515576333693
```

The number was too large for CyberChef to handle directly. It was first converted to hex using RapidTables:

🔗 [RapidTables Decimal to Hex](https://www.rapidtables.com/convert/number/decimal-to-hex.html?x=581695969015253365094191591547859387620042736036246486373595515576333693)

```
54484D7B31375F6A7535375F346E5F307264316E3472795F62343533357D
```

The resulting hex string was then decoded to ASCII in CyberChef using **From Hex**:

🔗 [CyberChef Recipe](https://cyberchef.io/#recipe=From_Hex('Auto')&input=NTQ0ODRkN2IzMTM3NWY2YTc1MzUzNzVmMzQ2ZTVmMzA3MjY0MzE2ZTM0NzI3OTVmNjIzNDM1MzMzNTdk)

> 🚩 `THM{17_ju57_4n_0rd1n4ry_b4535}`

---

## Task 21 — Network Traffic Analysis

**Challenge:** "I just hacked my neighbor's WiFi and try to capture some packet."

The `.pcapng` file was opened in **Wireshark** and filtered by HTTP traffic:

```bash
wireshark flag_1578026731881.pcapng
```

```
http
```

Packet 1827 showed an HTTP `200 OK` response to a `GET /flag.txt` request. Following the TCP stream revealed the flag in plain text in the response body.

> 🚩 `THM{d0_n07_574lk_m3}`

---


## Techniques Summary

| Task | Technique | Tool |
|------|-----------|------|
| 02 | Base64 decode | CyberChef |
| 03 | Image metadata extraction | exiftool |
| 04 | Steganography extraction (no passphrase) | steghide |
| 05 | Hidden white text | Browser / dark mode |
| 06 | QR code scan | Google Lens |
| 07 | Printable string extraction from ELF binary | strings |
| 08 | Base58 decode | CyberChef |
| 09 | Caesar cipher — ROT7 | CyberChef |
| 10 | Page source inspection | Browser DevTools (F12) |
| 11 | Corrupted PNG magic bytes repair | GHex |
| 12 | Social media OSINT | Reddit / r/tryhackme |
| 13 | Brainfuck code execution | copy.sh/brainfuck |
| 14 | XOR with repeating hex key | CyberChef |
| 15 | Embedded file extraction from JPG | binwalk |
| 16 | LSB bit-plane steganography — Blue plane 1 | Stegsolve |
| 17 | QR code scan + audio listening | Google Lens / SoundCloud |
| 18 | Web archive OSINT | Wayback Machine |
| 19 | Vigenère cipher decode (key: THM) | CyberChef |
| 20 | Decimal → Hex → ASCII | RapidTables + CyberChef |
| 21 | HTTP stream analysis from PCAPNG | Wireshark |

---

*CTF Collection Vol.1 — TryHackMe | Completed ✅*
