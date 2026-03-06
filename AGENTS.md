# AGENTS.md

**Version:** 2.0
**Date:** 2026-03-06
**Purpose:** Technical reference for deploy_hdhomerun development

---

## Project Overview

**deploy_hdhomerun** is a Bash shell script that provisions and auto-updates the [HDHomeRun Record](https://www.silicondust.com/) software on Linux devices. It handles first-time installation and weekly update checking via cron, with full support for ARM64 (aarch64) and x86_64 platforms.

Tested on:
- **SteamOS** (x86_64, immutable root via overlayfs)
- **Netgear RN102** (ARM64 NAS)
- **Synology DS418** (ARM64 NAS)

- **Language:** Bash 4+
- **Architecture:** Single executable script + gitignored site config
- **Target platforms:** ARM64 and x86_64 Linux with systemd and cronie
- **Upstream source:** http://download.silicondust.com/hdhomerun/hdhomerun_record_linux

---

## Quick Setup

```bash
# Clone or download the repository
git clone https://github.com/andrewwyatt/deploy_hdhomerun.git
cd deploy_hdhomerun

# Create and edit your site configuration (never committed)
cp hdhomerun.local.example hdhomerun.local
vi hdhomerun.local

# Run as root to install
sudo ./deploy_hdhomerun

# Dry-run to preview what would happen (no changes made)
sudo ./deploy_hdhomerun -n -v
```

---

## Configuration

Site-specific settings live in `hdhomerun.local` (gitignored, never committed).
The file `hdhomerun.local.example` is the committed template with all options documented.

**Key settings:**

| Variable | Purpose | Default |
|----------|---------|---------|
| `BASEPATH` | Root for all HDHomeRun data | `$HOME/hdhomerun` |
| `APPPATH` | Application install directory | `$BASEPATH/app` |
| `SHOWPATH` | Recording storage path | `$BASEPATH/shows` |
| `SHOW_SYMLINK_TARGET` | If set, SHOWPATH becomes a symlink here (for NFS mounts) | *(unset)* |
| `SYSTEMD_UNIT_DIR` | Where to install the .service file | `/etc/systemd/system` |
| `UPDATE_CRON` | Cron schedule for weekly updates | `00 00 * * 00` |
| `DEPLOY_DIR` | Deploy script location (for cron job) | script directory |
| `UPDATE_LOG` | Log file for cron-driven updates | `/var/log/hdhomerun_deploy.log` |

**NFS / network storage:**
Set `SHOW_SYMLINK_TARGET` to the NFS mount path (e.g. `/television`). The script will:
1. Create `SHOWPATH` as a symlink pointing there
2. Add `remote-fs.target` and the systemd mount unit as `After=` dependencies

**Immutable-root systems (SteamOS):**
Set `SYSTEMD_UNIT_DIR="/etc/systemd/system"` - the overlay makes this writable even though
`/usr/lib/systemd/system` is not.

---

## Architecture

```
deploy_hdhomerun (main script)
    |
    +-- Reads: hdhomerun.local (required, gitignored)
    |
    +-- Detects architecture (aarch64 -> arm64, x86_64 -> x64)
    |
    +-- Ensures APPPATH directory exists
    |
    +-- Manages SHOWPATH (real directory OR symlink to SHOW_SYMLINK_TARGET)
    |
    +-- Downloads: hdhomerun_record_linux from Silicondust (to mktemp dir)
    |       Extracts: hdhomerun_record_{arm64|x64}
    |       Cleanup: temp dir removed on exit via trap
    |
    +-- Branch: Update or Fresh Install?
    |
    |   [Update path - binary already installed]
    |   +-- sha256sum comparison (installed vs downloaded)
    |   +-- If different: stop service, replace binary, start service
    |   +-- If same: "No update necessary"
    |
    |   [Fresh install path - binary not present]
    |   +-- Create $APPPATH/hdhomerun_record/bin/
    |   +-- Install architecture binary
    |   +-- Create wrapper script: hdhomerun_record (uses exec + "$@")
    |   +-- Write hdhomerun.conf (RecordPath only; StorageID added by app)
    |   +-- Build systemd After= line (adds NFS mount unit if SHOW_SYMLINK_TARGET set)
    |   +-- Install unit to $SYSTEMD_UNIT_DIR/hdhomerun.service
    |   +-- systemctl daemon-reload, enable, start
    |   +-- Install weekly cron job: /etc/cron.d/hdhomerun
    |
    +-- Final: systemctl enable hdhomerun --now (idempotent)
```

---

## Directory Structure

| Path | Purpose |
|------|---------|
| `deploy_hdhomerun` | Main deployment/update script (executable) |
| `hdhomerun.local.example` | Template for site config (committed) |
| `hdhomerun.local` | Site config for this machine (gitignored, NOT committed) |
| `variables` | Backward-compat shim (retained for external references) |
| `README.md` | User-facing documentation |
| `LICENSE` | Apache 2.0 license |

**Runtime paths (set in `hdhomerun.local`):**

| Path | Purpose |
|------|---------|
| `$APPPATH/hdhomerun_record/bin/hdhomerun_record_${ARCH}` | Installed binary |
| `$APPPATH/hdhomerun_record/bin/hdhomerun_record` | Arch-agnostic wrapper |
| `$APPPATH/hdhomerun_record/hdhomerun.conf` | HDHomeRun config (RecordPath + StorageID) |
| `$SYSTEMD_UNIT_DIR/hdhomerun.service` | systemd unit file |
| `/etc/cron.d/hdhomerun` | Weekly update cron job |
| `/var/log/hdhomerun_deploy.log` | Cron job log output |

---

## Code Style

**Bash Conventions:**

- `set -euo pipefail` at the top - fail fast, no silent errors
- Use `#!/bin/bash` shebang (not `/bin/sh`)
- Use `[[ ... ]]` for string tests and regex matching
- Use `[ ... ]` for POSIX file tests (`-d`, `-f`, `-L`, etc.)
- Quote all variable expansions: `"${VAR}"` not `$VAR`
- Use `$()` for command substitution (never backticks)
- Use `exec` in wrapper scripts: `exec "${BINARY}" "$@"` (not `${BINARY} $*`)
- Error check every critical operation with `$?` or `||`
- Use heredoc (`<<EOF`) for multi-line file creation
- Descriptive section comments with `###` delimiters
- Exit with non-zero on fatal errors via `die()`

**Helper function pattern:**

```bash
log()     { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
verbose() { [[ ${VERBOSE} -eq 1 ]] && echo "..." || true; }
err()     { echo "ERROR: $*" >&2; }
die()     { err "$*"; exit 1; }

run() {
  ## Executes command, or prints it in dry-run mode.
  if [[ ${DRYRUN} -eq 1 ]]; then
    echo "[DRY-RUN] $*"
  else
    "$@"
  fi
}
```

**Temp file pattern (always use trap for cleanup):**

```bash
TMPDIR_WORK="$(mktemp -d /tmp/hdhomerun_deploy.XXXXXX)"
trap 'rm -rf "${TMPDIR_WORK}"' EXIT
```

**Find script directory reliably (works from cron, any cwd):**

```bash
DEPLOYDIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
```

---

## systemd Service

The script creates `${SYSTEMD_UNIT_DIR}/hdhomerun.service` on fresh install:

```ini
[Unit]
Description=HDHomerun Record Service
After=network-online.target [remote-fs.target] [television.mount]

[Service]
Type=oneshot
ExecStart=.../hdhomerun_record start --conf .../hdhomerun.conf
ExecStop=.../hdhomerun_record stop --conf .../hdhomerun.conf
RemainAfterExit=yes
Nice=0

[Install]
WantedBy=multi-user.target
```

**Key design decisions:**
- `Type=oneshot` + `RemainAfterExit=yes` - allows stop/start semantics for a daemon-mode binary
- `After=network-online.target` - always required (device discovery over LAN)
- `After=remote-fs.target + <mount>.mount` - added automatically when `SHOW_SYMLINK_TARGET` is set
- Unit goes to `$SYSTEMD_UNIT_DIR` (configurable) - **not** hardcoded to `/lib/systemd/system`

**SteamOS note:** `/usr/lib/systemd/system` and `/lib/systemd/system` are on the read-only
immutable root. Use `SYSTEMD_UNIT_DIR="/etc/systemd/system"` (writable via overlayfs upper).

---

## hdhomerun.conf and StorageID

The app writes a `StorageID` line into `hdhomerun.conf` on first start. **Do not overwrite this file on re-deploy** - the deploy script only creates it during fresh install when the binary is not yet present.

If you need to fully reset (clear StorageID), manually delete the conf file before running the script.

---

## Cron Job

The weekly update cron is installed to `/etc/cron.d/hdhomerun`:

```
00 00 * * 00 root /path/to/deploy_hdhomerun >/var/log/hdhomerun_deploy.log 2>&1
```

- Schedule controlled by `UPDATE_CRON` in `hdhomerun.local`
- Runs as root (required for service stop/start and binary replacement)
- Logs to `UPDATE_LOG` (default `/var/log/hdhomerun_deploy.log`)

---

## Testing

**Before Committing:**

```bash
# 1. Syntax check the script
bash -n deploy_hdhomerun

# 2. Check config files
bash -n hdhomerun.local.example
bash -n hdhomerun.local   # if present

# 3. Shellcheck (if available)
shellcheck deploy_hdhomerun

# 4. Dry-run (no changes, verbose output)
sudo ./deploy_hdhomerun -n -v
```

**Manual Testing on Target Device:**

```bash
# Dry-run first
sudo ./deploy_hdhomerun -n -v

# Fresh install test (as root on target)
sudo ./deploy_hdhomerun

# Verify service is running
systemctl status hdhomerun

# Verify cron job installed
cat /etc/cron.d/hdhomerun

# Verify binary and wrapper installed
ls -la $APPPATH/hdhomerun_record/bin/

# Verify hdhomerun.conf
cat $APPPATH/hdhomerun_record/hdhomerun.conf

# Update test: run again - should report "No update necessary"
sudo ./deploy_hdhomerun -v
```

---

## Commit Format

```
type(scope): brief description

Problem: What was broken/missing/needed
Solution: How you addressed it
Testing: How you verified the change
```

**Types:** `feat`, `fix`, `refactor`, `docs`, `chore`

**Examples:**

```bash
git commit -m "fix(install): write unit to /etc/systemd/system not /lib

Problem: /lib/systemd/system is read-only on SteamOS (immutable root)
Solution: SYSTEMD_UNIT_DIR config variable, defaults to /etc/systemd/system
Testing: Dry-run verified, systemctl status confirmed on SteamOS"

git commit -m "feat(config): replace variables with hdhomerun.local

Problem: variables was committed, causing accidental site-specific commits
Solution: gitignored hdhomerun.local with example template in repo
Testing: Dry-run verified config load, hdhomerun.local gitignored confirmed"
```

---

## Development Tools

**Common Commands:**

```bash
# Syntax check
bash -n deploy_hdhomerun

# Dry-run (verbose, no changes)
sudo ./deploy_hdhomerun -n -v

# Trace execution
bash -x deploy_hdhomerun

# Trace with line numbers
PS4='Line ${LINENO}: ' bash -x ./deploy_hdhomerun

# Check architecture on target
uname -m

# Git status / history
git status && git log --oneline -5

# Service management
systemctl status hdhomerun
systemctl restart hdhomerun
journalctl -u hdhomerun

# View deployment log
tail -f /var/log/hdhomerun_deploy.log
```

---

## Common Patterns

**Architecture Detection:**

```bash
case "$(uname -m)" in
  aarch64) ARCH="arm64" ;;
  x86_64)  ARCH="x64" ;;
  *)        die "Unsupported architecture: $(uname -m)" ;;
esac
```

**sha256sum Checksum Update Guard:**

```bash
CURRENT_SUM="$(sha256sum "${INSTALL_BIN}" | awk '{print $1}')"
DOWNLOADED_SUM="$(sha256sum "${EXTRACTED_BIN}" | awk '{print $1}')"

if [[ "${DOWNLOADED_SUM}" != "${CURRENT_SUM}" ]]; then
  # Perform upgrade
else
  log "No update necessary (sha256 matches)."
fi
```

**Wrapper script with proper quoting:**

```bash
cat > "${WRAPPER}" <<WRAPPER_EOF
#!/bin/bash
exec "${INSTALL_BIN}" "\$@"
WRAPPER_EOF
```

**NFS mount unit dependency:**

```bash
MOUNT_UNIT="$(systemd-escape --path --suffix=mount "${SHOW_SYMLINK_TARGET}")"
SYSTEMD_AFTER="network-online.target remote-fs.target ${MOUNT_UNIT}"
```

---

## Documentation

### What Needs Documentation

| Change Type | Required Documentation |
|-------------|------------------------|
| New config variable | Add to `hdhomerun.local.example` + AGENTS.md Config table |
| New platform support | Update README.md Prerequisites, AGENTS.md Overview |
| systemd unit change | Update AGENTS.md (systemd Service section) |
| New feature or flag | Update README.md + AGENTS.md |
| Bug fix | Clear commit message explaining problem + solution |

---

## Anti-Patterns (What NOT To Do)

| Anti-Pattern | Why It's Wrong | What To Do |
|--------------|----------------|------------|
| Hardcode `/lib/systemd/system` | Read-only on SteamOS and many modern distros | Use `$SYSTEMD_UNIT_DIR` variable |
| Use `sha1sum` for checksums | Deprecated, collision-vulnerable | Use `sha256sum` |
| Use `$*` in wrapper scripts | Breaks arguments with spaces | Use `"$@"` or `exec bin "$@"` |
| Use `$(pwd)` to find script dir | Breaks when called from cron | Use `${BASH_SOURCE[0]}` pattern |
| Leave temp files in `/tmp` | Leaks on error | Use `mktemp -d` + `trap ... EXIT` |
| Overwrite `hdhomerun.conf` on re-deploy | Destroys `StorageID` written by app | Only create conf on fresh install |
| Commit `hdhomerun.local` | Leaks site-specific paths to public repo | It's gitignored - keep it that way |
| Unquoted variable expansions | Word splitting / glob bugs | Always quote: `"${VAR}"` |
| Skip error checks on curl/mv | Silent failures, corrupted install | Use `set -euo pipefail` + `die()` |
| Using backticks for substitution | Hard to nest, less readable | Use `$()` instead |
| Run without root check | Confusing failures partway through | Check `$EUID` at start |

---

## Quick Reference

```bash
# Syntax check
bash -n deploy_hdhomerun

# Dry-run preview
sudo ./deploy_hdhomerun -n -v

# Real run (verbose)
sudo ./deploy_hdhomerun -v

# Trace execution
bash -x deploy_hdhomerun

# Git status
git status && git log --oneline -5

# Service management
systemctl status hdhomerun
systemctl restart hdhomerun
journalctl -u hdhomerun --since "1 hour ago"

# View deployment log
tail -f /var/log/hdhomerun_deploy.log

# Check installed binary
ls -la ~/hdhomerun/app/hdhomerun_record/bin/

# Verify config
cat ~/hdhomerun/app/hdhomerun_record/hdhomerun.conf
```

---

*For project methodology and workflow, see .clio/instructions.md*
