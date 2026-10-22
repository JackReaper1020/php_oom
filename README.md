# php2shell — CVE-2026-109618 OOM Bailout Crash Scanner

A Python 3 triage scanner for detecting PHP out-of-memory (OOM) bailout vulnerabilities in web servers. Sends crafted allocation-stress payloads and classifies server behaviour — crash, handled, rejected, or WAF-limited — without performing any exploitation.

> **Authorized use only.** Run this tool only against targets you own or have explicit written permission to test.

---

## What it does

PHP's memory allocator can be forced into a bailout path when a single request triggers allocations beyond `memory_limit`. Depending on the PHP-FPM / mod_php configuration and the PHP version, this can cause a worker process to crash rather than return a clean error. This scanner probes for that behaviour using four built-in plugins:

| Plugin | Method | Default size |
|---|---|---|
| `nested` | Deeply nested array input keys | 5 000 params × depth 20 |
| `string` | Single large URL-encoded field | 35 MiB |
| `multipart` | RFC 1867 multipart with many large parts | 5 × 8 MiB |
| `escalate` | Bisect-style escalating body size | 1 → 128 MiB |

---

## Installation

```bash
git clone https://github.com/youruser/php2shell.git
cd php2shell
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## Requirements

- Python 3.10+
- See `requirements.txt`

---

## Usage

```bash
# Run all plugins against a target
python3 CVE-2026-109618.py https://target/upload.php

# Dry-run — generate payloads without sending anything
python3 CVE-2026-109618.py https://target/upload.php --dry-run

# Single plugin
python3 CVE-2026-109618.py https://target/upload.php --plugin multipart

# With Burp proxy, custom timeout, verbose output
python3 CVE-2026-109618.py https://target/upload.php \
    --proxy http://127.0.0.1:8080 \
    --timeout 20 \
    --retries 5 \
    --verbose \
    --log-file audit.log

# Disable TLS verification (self-signed certs)
python3 CVE-2026-109618.py https://target/upload.php --no-verify
```

### All flags

| Flag | Default | Description |
|---|---|---|
| `target_url` | — | Full URL of the PHP endpoint to test |
| `--plugin` | `all` | `nested`, `string`, `multipart`, `escalate`, or `all` |
| `--timeout` | `15.0` | Per-request timeout in seconds |
| `--retries` | `3` | Crash-confirmation retry attempts |
| `--proxy` | — | HTTP proxy (e.g. `http://127.0.0.1:8080`) |
| `--no-verify` | off | Disable TLS certificate verification |
| `--dry-run` | off | Build payloads but do not send them |
| `--log-file` | — | Write log output to a file |
| `-v / --verbose` | off | Enable DEBUG-level logging |

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | No crash observed |
| `1` | Crash signature observed on at least one plugin |
| `2` | Target unreachable / fatal error |
| `3` | Interrupted by user (Ctrl-C) |

---

## Example output

```
2026-10-22 14:03:11 INFO     [*] Starting PHP OOM-bailout audit against: https://target/upload.php
2026-10-22 14:03:11 INFO     [+] Baseline reachable – HTTP 200
2026-10-22 14:03:11 INFO     [*] Plugin: Deep-Nested Array (5000 params × depth 20)
2026-10-22 14:03:13 INFO         Verdict: PROCESSED (200)
2026-10-22 14:03:13 INFO     [*] Plugin: Large URL-Encoded Field (35 MiB)
2026-10-22 14:03:18 WARNING  [!] Connection reset mid-request – possible worker crash.
2026-10-22 14:03:21 CRITICAL [CRITICAL] Reset reproduced on 2/3 retries and endpoint still alive.
2026-10-22 14:03:21 INFO         Verdict: CRASH
======================================================================
[*] Summary:
    [-] Deep-Nested Array (5000 params × depth 20): PROCESSED (200)
    [!] Large URL-Encoded Field (35 MiB): CRASH

[!] Worker-crash signature observed on at least one plugin.
    Manual confirmation required: check PHP-FPM / Apache error logs
    for segfaults or child-process exits, and repeat with varied sizes.
```

---

## How crash detection works

1. A baseline GET confirms the target is reachable before any stress payloads are sent.
2. Each plugin sends its payload. A `ConnectionError` with a reset/broken-pipe signature is flagged as a suspected crash.
3. The plugin retries up to `--retries` times to confirm the reset is reproducible.
4. A lightweight GET checks whether the endpoint is still alive post-reset.
5. If `≥ 2` retries reproduced the reset **and** the endpoint is still up, the verdict is `CRASH`. A non-reproducible reset is classified as `WAF/LIMIT (suspected)`.

---

## Project structure

```
php2shell/
├── CVE-2026-109618.py   # Main scanner
├── requirements.txt
├── .venv/               # Local virtualenv (gitignored)
└── README.md
```

---

## .gitignore

Paste this into `.gitignore` before your first commit:

```
.venv/
__pycache__/
*.pyc
*.log
.env
```

---

## Disclaimer

This tool is intended for **authorized security testing and research only**. The author is not responsible for any misuse or damage caused by running this tool against systems without explicit permission. Always operate within the scope of your bug bounty programme or a signed penetration testing agreement.

---

## Author

James Mangrati — bug bounty / security research
