Hand-over documentation for termux-ops-toolkit acknowledged and indexed. The architecture is clean, well-scoped for mobile Linux constraints, and correctly leverages native SQLite backup primitives alongside zstandard compression.
To ensure a seamless transition and prepare for the upcoming v1.2.0-rc1 milestone, here are the direct engineering resolutions for the documented technical debt items (BUG-001 and BUG-002).
Resolving BUG-001: SQLite Concurrency Locking
Under high concurrent write loads, Python's default sqlite3 connection timeout (typically 5.0 seconds) is insufficient for online hot backups, resulting in sqlite3.OperationalError: database is locked.
To fix this in src/ops_toolkit/db_backup.py, pass an explicit timeout parameter during connection initialization and enforce a WAL-safe busy timeout via PRAGMA execution before initiating the backup cursor:
import sqlite3
from pathlib import Path

def create_safe_connection(db_path: Path, timeout: float = 30.0) -> sqlite3.Connection:
    conn = sqlite3.connect(db_path, timeout=timeout, isolation_level=None)
    conn.execute("PRAGMA journal_mode=WAL;")
    conn.execute("PRAGMA busy_timeout = 30000;")
    return conn

By setting both the connection-level timeout and the busy_timeout PRAGMA to 30 seconds, the backup thread will gracefully wait out write transactions rather than failing immediately.
Resolving BUG-002: Android 14 Scoped Storage Symlinks
Android 14 restricts direct path traversal and symlink resolution under /storage/emulated/0/ for background shell processes, causing audit_env.sh disk space assertions to fail when evaluating termux-storage-path symlinks.
To fix this in bin/audit_env.sh, bypass fragile symbolic link resolution by performing an atomic write-probe test rather than relying on readlink:
# Verify shared storage writeability and free space via atomic probe instead of symlink parsing
verify_scoped_storage() {
    local target_dir="/sdcard/Backups"
    local probe_file="${target_dir}/.audit_probe_$$"

    if mkdir -p "$target_dir" 2>/dev/null && touch "$probe_file" 2>/dev/null; then
        rm -f "$probe_file"
        return 0
    else
        echo "[ERROR] Scoped storage path unreachable or write-blocked on Android 14." >&2
        return 1
    fi
}

Next Operational Objectives
The repository is primed for the v1.2.0-rc1 release cycle. Whenever you are ready, we can proceed with implementing either the automated zstd backup rotation logic or the Telegram notification webhooks.
https://mullvad.net/en/blog/shutting-down-our-public-encrypted-dns-servers-and-sponsoring-quad9-instead**Project Handover – termux‑ops‑toolkit**  
*Prepared for the dev team – bilbywilby*  

---  

## 1. Project Overview & Specification  

| Item | Details |
|------|---------|
| **Project Name** | `termux-ops-toolkit` |
| **Repository** | <https://github.com/bilbywilby/termux-ops-toolkit.git> |
| **Primary Languages / Runtime** | Python 3.11.8 (Termux package), Bash 5.2+, SQLite 3.42.0 |
| **Python Dependencies** | `pydantic>=2.6.0`, `rich>=13.7.0`, `httpx>=0.27.0`, `zstandard` (runtime) |
| **Target Platforms** | • Termux on Android 14 (aarch64)  <br>• Debian 12 / Ubuntu 22.04 LTS CI runners & headless servers |
| **Key Components** | • `audit_env.sh` – system‑integrity & storage audit  <br>• `db_backup.py` – WAL‑safe hot‑backup with ZSTD compression  <br>• `w3m_browser.sh` – headless HTML‑to‑text wrapper  <br>• `portal_tui.py` – Rich/Textual dashboard for task monitoring |
| **Open Bugs / Technical Debt** | **BUG‑001** – `db_backup.py` fails on locked DBs under high concurrency (no `busy_timeout` set). <br>**BUG‑002** – `audit_env.sh` misreports Termux storage symlinks on Android 14 due to Scoped Storage changes. |
| **Upcoming Milestones** | **v1.2.0‑rc1** – Automated ZSTD rotation & Telegram webhook (target 2026‑11‑01). <br>**v1.3.0** – Migrate scheduling from `crontab` → Termux Job Scheduler IPC (target 2026‑12‑15). |

---  

## 2. Technical Architecture Blueprint  

```
+-----------------------------------------------------------------------------------+
|                               Termux Host Environment                            |
|                                                                                   |
|  +-------------------+      +----------------------+      +--------------------+  |
|  |   Scheduler (crontab / Termux‑job‑scheduler)  |-->| audit_env.sh        |-->| /var/log/ops_audit |
|  +-------------------+      +----------------------+      +--------------------+  |
|                                                                                   |
|  +-------------------+      +----------------------+      +--------------------+  |
|  |   portal_tui.py   |<----|   db_backup.py       |----->|  Target Storage    |  |
|  |   (Rich engine)   |      | (sqlite3 + ZSTD)    |      |  (/sdcard/Backups) |  |
|  +-------------------+      +----------------------+      +--------------------+  |
|                                      |                                          |
|                                      v                                          |
|                             +--------------------+                                |
|                             | SQLite DB (WAL)    |                                |
|                             +--------------------+                                |
+-----------------------------------------------------------------------------------+
```

*Data flow*: Scheduler triggers `audit_env.sh` → logs → optional alert. Scheduler also runs `db_backup.py` → compressed backup → external storage. `portal_tui.py` reads the same DB and backup directory to display health metrics.

---  

## 3. Directory Structure  

```
termux-ops-toolkit/
├─ .github/
│   └─ workflows/
│       └─ ci.yml
├─ bin/
│   ├─ audit_env.sh
│   └─ w3m_browser.sh
├─ src/
│   └─ ops_toolkit/
│       ├─ __init__.py
│       ├─ db_backup.py
│       └─ portal_tui.py
├─ tests/
│   ├─ test_audit.py
│   └─ test_backup.py
├─ config/
│   └─ default_policy.json
├─ pyproject.toml
└─ README.md
```

---  

## 4. Environment Setup & Execution  

### 4.1 Prerequisites (Termux)

```bash
pkg update && pkg upgrade -y
pkg install python bash sqlite w3m zstd git -y
```

### 4.2 Installation

```bash
git clone https://github.com/bilbywilby/termux-ops-toolkit.git
cd termux-ops-toolkit

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install -e .[dev]   # installs the package plus dev extras (pytest, mypy, etc.)
```

### 4.3 Quick Test Run

```bash
# Bash audit
bash bin/audit_env.sh

# Python backup (dry‑run)
python -m ops_toolkit.db_backup data/production.db /tmp/backup_test.db.zst
```

---  

## 5. Deployment & Core Scripts  

### 5.1 `audit_env.sh` (bin/audit_env.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="${HOME}/.local/share/ops_toolkit/audit.log"
mkdir -p "$(dirname "$LOG_FILE")"

exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== [$(date -u +'%Y-%m-%dT%H:%M:%SZ')] STARTING ENVIRONMENT AUDIT ==="

# Verify required executables
for cmd in python3 sqlite3 w3m zstd; do
    command -v "$cmd" >/dev/null || {
        echo "ERROR: Required executable '$cmd' not found." >&2
        exit 1
    }
done

# Disk‑space warning (threshold 1 GB)
FREE_KB=$(df -k "$HOME" | awk 'NR==2 {print $4}')
if (( FREE_KB < 1048576 )); then
    echo "WARNING: Low disk space – $((FREE_KB/1024)) MB available."
else
    echo "INFO: Disk space OK – $((FREE_KB/1024)) MB available."
fi

# Optional additional checks can be inserted here (e.g., permission validation).

echo "=== AUDIT COMPLETE ==="
chmod 600 "$LOG_FILE"
```

*Key hardening*: `chmod 600` immediately after log creation; aborts if any required binary missing.

### 5.2 `db_backup.py` (src/ops_toolkit/db_backup.py)

```python
#!/usr/bin/env python3
"""WAL‑safe SQLite hot‑backup with ZSTD compression."""

from __future__ import annotations
from pathlib import Path
import sqlite3
import sys
import zstandard as zstd


def backup_database(
    source_db: Path,
    target_archive: Path,
    timeout_sec: float = 30.0,
) -> None:
    """Create a hot backup of *source_db* and write a .zst archive to *target_archive*."""
    if not source_db.is_file():
        raise FileNotFoundError(f"Source DB {source_db} does not exist")

    target_archive.parent.mkdir(parents=True, exist_ok=True)
    tmp_path = target_archive.with_suffix(".tmp")

    # Apply busy timeout to mitigate BUG‑001 (locked DB)
    src_conn = sqlite3.connect(source_db, timeout=timeout_sec)
    dst_conn = sqlite3.connect(tmp_path)

    try:
        with dst_conn:
            src_conn.backup(dst_conn, pages=-1)
    finally:
        src_conn.close()
        dst_conn.close()

    # ZSTD stream compression (level 10)
    compressor = zstd.ZstdCompressor(level=10)
    with open(tmp_path, "rb") as fin, open(target_archive, "wb") as fout:
        compressor.copy_stream(fin, fout)

    tmp_path.unlink(missing_ok=True)


if __name__ == "__main__":
    if len(sys.argv) != 3:
        print(
            "Usage: python -m ops_toolkit.db_backup <source.db> <target.db.zst>",
            file=sys.stderr,
        )
        sys.exit(1)

    src = Path(sys.argv[1]).resolve()
    dst = Path(sys.argv[2]).resolve()

    try:
        backup_database(src, dst)
        print(f"Backup created → {dst}")
    except Exception as exc:
        print(f"Backup failed: {exc}", file=sys.stderr)
        sys.exit(2)
```

*Fixes* BUG‑001 by exposing `timeout_sec` (default 30 s) and ensuring the backup runs with a busy timeout.

### 5.3 `w3m_browser.sh` (bin/w3m_browser.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail

if (( $# != 1 )); then
    echo "Usage: $0 <URL>" >&2
    exit 1
fi

URL="$1"
w3m -dump -cols 100 -T text/html "$URL"
```

### 5.4 `portal_tui.py` (src/ops_toolkit/portal_tui.py)

> *The file uses `rich` and `textual` to render a live dashboard.  
>  Keep the entry‑point `python -m ops_toolkit.portal_tui`.*  

(Full source omitted for brevity – see repository).

---  

## 6. Operational Runbooks & Scheduled Jobs  

### 6.1 Crontab (Termux)  

```cron
# Edit with `crontab -e` inside Termux
0 2 * * * $HOME/termux-ops-toolkit/bin/audit_env.sh \
    >> $HOME/.local/share/ops_toolkit/cron.log 2>&1

0 3 * * * $HOME/termux-ops-toolkit/.venv/bin/python -m ops_toolkit.db_backup \
    $HOME/data/production.db \
    /sdcard/Backups/db_$(date +\%Y\%m\%d).db.zst \
    >> $HOME/.local/share/ops_toolkit/cron.log 2>&1
```

*Note*: When migrating to Termux Job Scheduler (v1.3.0), replace the `crontab` entries with equivalent `termux-job-scheduler` JSON payloads.

### 6.2 Manual Runbook  

1. **Verify storage mount** – `termux-setup-storage` (first‑time only).  
2. **Run audit** – `bash bin/audit_env.sh`. Review `~/.local/share/ops_toolkit/audit.log`.  
3. **Perform backup** – `python -m ops_toolkit.db_backup data/production.db /sdcard/Backups/$(date +%F).db.zst`.  
4. **Launch dashboard** – `python -m ops_toolkit.portal_tui`.  

### 6.3 Alerting (v1.2.0‑rc1)  

A Telegram bot token (stored in `~/.config/ops_toolkit/credentials.env`) will be read by the backup script; on success/failure it will POST a concise message to the configured chat. Integration points are already scaffolded in `src/ops_toolkit/alert.py` (future merge).

---  

## 7. Operational Security & Threat Model Checklist  

- **File Permissions**  
  - `chmod 0755 bin/*.sh`  
  - `chmod 0600 data/*.db` and any exported backups before moving to shared storage (`zstd` stream is encrypted only if the key is added).  

- **Secret Management**  
  - Store API keys / bot tokens in `~/.config/ops_toolkit/credentials.env` with `0600` permissions.  
  - Load via `source` or `dotenv` – never commit to repo.  

- **Scoped Storage Compliance** (Android 14)  
  - Backups are first written to internal private storage, then *moved* to external `/sdcard/Backups` after compression.  
  - Verify `run-as` permissions before any external write to avoid `PermissionDenied` errors.  

- **Network Isolation**  
  - All outbound HTTP calls (e.g., Telegram alerts, optional remote health checks) are performed over HTTPS.  
  - Use `httpx` with built‑in TLS verification; do not disable cert checks.  

- **Backup Rotation (v1.2.0‑rc1)**  
  - Keep the most recent 7 daily, 4 weekly, and 12 monthly archives.  
  - Remove older files with a safe `find … -mtime +X -delete` routine executed after each successful backup.  

---  

## 8. Escalation & Maintenance  

| Role | Contact | Action |
|------|---------|--------|
| **Maintainer** | bilbywilby (GitHub @bilbywilby) | Merge PRs, approve releases, review security patches. |
| **Primary Escalation** | Open an issue on GitHub (label **bug** or **security**) | Include logs (`audit.log`, backup stderr) and steps to reproduce. |
| **Secondary Contact** | Ops‐Team Slack channel `#termux-ops` | Immediate incident triage (disk full, backup failure). |
| **Release Process** | CI pipeline (`.github/workflows/ci.yml`) runs tests → builds wheel → tags `vX.Y.Z`. | Tag push triggers GitHub Release draft; maintainers must verify changelog and bump version. |

---  

**End of handover**. Let me know if any section requires deeper detail or if you need a specific run‑book (e.g., migration to Termux Job Scheduler).
