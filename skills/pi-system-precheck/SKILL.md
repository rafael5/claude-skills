---
name: pi-system-precheck
version: 1.0.0
description: >
  Generate a comprehensive pre-installation validation script for Raspberry Pi systems
  (Pi 4/5, Debian Bookworm/Bullseye, aarch64). Use this skill whenever the user wants
  to check system readiness before installing any application, service, or platform on
  a Raspberry Pi — including Docker stacks, Kubernetes distributions, home automation,
  media servers, or any server application. Also triggers for: "is my Pi ready for X",
  "check my Pi before installing", "Pi system health check", "validate Pi prerequisites".
  The output is always a self-contained, executable bash script.
targets: [raspberry-pi-4, raspberry-pi-5, debian-bookworm, debian-bullseye, aarch64]
changelog:
  - 1.0.0: Initial version. Distilled from PodCloud/Pi5 precheck debugging sessions.
---

# Pi System Pre-Installation Checker Skill

## Purpose

Produce a single, self-contained bash script that validates a Raspberry Pi system
before any application installation. The script must be safe to run repeatedly
(idempotent reads), actionable (every failure has a remediation command), and honest
(no false positives, no masking of real problems).

## Clarify Before Writing

Ask if not clear from context:

1. **Target application** — what are they installing? Determines which checks are
   critical vs optional and which ports/packages/services to validate.
2. **Pi model and OS** — Pi 4 or Pi 5? Bookworm or Bullseye? Affects boot config
   paths, cgroup locations, and available commands.
3. **Access model** — runs as root or sudo user? Affects whether auto-fix sections
   can execute.

If "generic" or unspecified, default to the most comprehensive check set (treat as
container orchestration platform — that covers all bases).

---

## Script Architecture

Every generated script MUST follow this exact section order:

```
1.  Shebang + metadata header (version, date, purpose, usage, exit codes)
2.  Strict mode: set -uo pipefail  (NOT set -e — see Safety Patterns)
3.  Color and output constants
4.  Global counters and arrays: PASS, WARN, FAIL, SKIP, FAILURES[], WARNINGS[]
5.  Global variable initialization (ALL variables used later — prevents set -u errors)
6.  Helper functions: pass/warn/fail/info/skip/section/inc/version_gte/has_cmd/pad
7.  Section 1:  Platform & Hardware
8.  Section 2:  Operating System
9.  Section 3:  Users & Permissions
10. Section 4:  Networking (DNS, ports, firewall, MTU)
11. Section 5:  Container Runtime       [omit if not needed]
12. Section 6:  Orchestration/K8s       [omit if not needed]
13. Section 7:  System Packages
14. Section 8:  Runtime Versions
15. Section 9:  systemd & Services
16. Section 10: TLS / Certificates
17. Section 11: Storage & Limits
18. Section 12: Security
19. Section 13: Raspberry Pi Specific
20. Section 14: Package Manager & Updates
21. Section 15: Auto-Tune (apply all fixable issues)
22. Section 16: Optional Integrations   [omit if not needed]
23. Final Summary
```

Never merge sections. Keep them numbered and delimited with `# ===` banner comments.

---

## Critical Safety Patterns

These were learned through real debugging. Each one has caused a real production bug.

### 1. Arithmetic increments — NEVER use `((VAR++))`

When `VAR=0`, `((VAR++))` evaluates to 0 (falsy) and returns exit code 1. With
`set -uo pipefail` this silently kills the script at the very first pass() call.

```bash
# WRONG
((PASS++))

# CORRECT — define once, use everywhere
inc() { local v="${!1}"; eval "$1=$(( v + 1 ))"; return 0; }
inc PASS
```

### 2. Pipeline + fallback in command substitution — the "degradedunknown" bug

When a command exits non-zero inside a pipeline that has a `|| fallback`, pipefail
propagates the non-zero exit through the whole pipe — so BOTH the pipeline output
AND the fallback are captured in the variable.

```bash
# WRONG — produces "degradedunknown" when systemctl exits 1 for "degraded" state
STATE=$(systemctl is-system-running 2>/dev/null | head -1 | tr -d '[:space:]' || echo "unknown")

# CORRECT — capture raw with || true, then process in a second step
STATE_RAW=$(systemctl is-system-running 2>/dev/null || true)
STATE=$(printf "%s" "$STATE_RAW" | head -1 | tr -d '[:space:]')
STATE="${STATE:-unknown}"
```

Apply this pattern to ANY command that: (a) exits non-zero by design on non-error
conditions, AND (b) is part of a pipeline feeding a variable. Key offenders:
`systemctl is-system-running`, `systemctl is-active`, `docker info` (when docker is
stopped), `k3s --version` (when k3s is absent).

### 3. Count variables through pipes — always normalize

Pipes through `tee` (used for logging) double the output. `wc -l` and `grep -c`
can embed trailing newlines. Always strip:

```bash
# WRONG
COUNT=$(docker ps -q | wc -l)

# CORRECT
COUNT_RAW=$(docker ps -q 2>/dev/null | wc -l || echo 0)
COUNT=$(echo "$COUNT_RAW" | tr -d '[:space:]')
COUNT="${COUNT:-0}"
```

### 4. NEVER use `set -e`

`set -e` causes silent aborts on any non-zero exit, including legitimate "false"
returns from test commands. Handle all errors explicitly. Use `set -uo pipefail` only.

### 5. Firewall: check-then-fix, not warn-then-fix

```bash
# WRONG — warn() adds to WARNINGS[] before the fix runs;
# the warning appears in the final summary even if auto-fix succeeded
if ! port_allowed "$PORT"; then
  warn "Port $PORT not open"
  sudo ufw allow "$PORT"   # Too late — warning already recorded
fi

# CORRECT — fix first, only warn on failure
if ! port_allowed "$PORT"; then
  if sudo ufw allow "$PORT" 2>/dev/null; then
    pass "Port $PORT — opened automatically"
  else
    warn "Port $PORT blocked — run: sudo ufw allow $PORT"
  fi
fi
```

### 6. Docker registry endpoint

The registry root `/` returns HTTP 404 by design. Use `/v2/` which returns 401
(auth challenge) — any non-000 HTTP code means the server is reachable and TLS works.

```bash
HTTP_CODE=$(curl -o /dev/null -sw "%{http_code}" https://registry-1.docker.io/v2/ 2>/dev/null || echo "000")
[[ "$HTTP_CODE" != "000" ]] && pass "registry reachable (HTTP $HTTP_CODE)" || fail "registry unreachable"
```

### 7. Initialize ALL variables at the top of the script

`set -u` will abort if any variable is referenced before assignment, including
variables that are only set inside conditional blocks (BOOT_CONFIG, CMDLINE_FILE,
AUTOFIX_COUNT, etc.). Initialize everything to safe defaults at the top.

---

## Output Conventions

Every check produces exactly one output line:

```bash
pass "message"   # ✔ green   — requirement met, no action needed
warn "message"   # ⚠ yellow  — suboptimal but not blocking
fail "message"   # ✘ red     — blocking, must fix before install
info "message"   # · dim     — informational only
skip "message"   # ○ dim     — check not applicable in this context
```

**Every `fail` and `warn` MUST include a concrete remediation inline:**
```bash
fail "cgroup memory not in cmdline — fix: sudo sed -i 's/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/cmdline.txt"
warn "vm.swappiness=60 — fix: echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-app.conf"
```

---

## Auto-Tune Section (Section 15)

Section 15 applies fixes for all issues detected in earlier sections. Do not warn
about fixable issues in earlier sections — mark them with `info` and fix in Section 15.

**Fixable issues:**

| Issue | Fix method |
|---|---|
| sysctl params wrong | Write `/etc/sysctl.d/99-<app>.conf`, run `sysctl --system` |
| File descriptor limits | Write `/etc/security/limits.d/99-<app>.conf` + systemd override |
| UFW ports missing | `sudo ufw allow <port>/<proto>` per port + `ufw reload` |
| `gpu_mem` not set | `sudo sed -i` or `tee -a` to config.txt |
| `cmdline.txt` params | Backup, then overwrite single line with appended params |
| PCIe Gen 3 missing | Append `dtparam=pciex1_gen=3` to config.txt |

**Auto-tune rules:**
- Backup before modifying: `cp "$FILE" "${FILE}.bak.$(date +%Y%m%d_%H%M%S)"`
- Track `NEEDS_REBOOT` (integer) — print prominent `sudo reboot` notice if > 0
- Track `AUTOFIX_COUNT` — report in final summary
- Degrade gracefully: if no sudo, print manual commands instead of failing

---

## Version Comparison Helper

```bash
version_gte() {
  # Returns 0 (true) if $1 >= $2 using version-aware sort
  printf '%s\n%s' "$2" "$1" | sort -V -C
}
```

---

## Raspberry Pi Specific Checks (always include)

**Model detection:**
```bash
PI_MODEL=$(tr -d '\0' < /proc/device-tree/model 2>/dev/null || echo "unknown")
```

**Boot config paths** (check both, use first that exists):
- Bookworm: `/boot/firmware/config.txt`, `/boot/firmware/cmdline.txt`
- Bullseye: `/boot/config.txt`, `/boot/cmdline.txt`

**Required cmdline.txt parameters for containers:**
`cgroup_enable=memory`, `cgroup_memory=1`, `cgroup_enable=cpuset`, `swapaccount=1`

**Headless config.txt optimizations:**
- `gpu_mem=16` — frees RAM (headless servers don't need GPU memory)
- `dtparam=pciex1_gen=3` — Pi 5 NVMe Gen3 speed (skip on Pi 4)

**Throttle check:**
```bash
THROTTLE=$(vcgencmd get_throttled 2>/dev/null | grep -oP '0x[0-9a-fA-F]+' || echo "0x0")
THROTTLE_INT=$(( 16#${THROTTLE#0x} ))
# 0x0 = healthy; non-zero = under-voltage, throttled, or temp-limited
```

---

## Application-Specific Customization

Parameterize these for each target application:

| Parameter | Purpose |
|---|---|
| `APP_NAME` | Used in file names, conf file names, section titles |
| `MIN_RAM_KB` | Minimum RAM in KB (8388608 = 8GB) |
| `MIN_DISK_KB` | Minimum free disk in KB |
| `REQUIRED_PORTS[]` | Array of `proto:port:description` entries |
| `REQUIRED_PKGS[]` | Packages that must be installed |
| `OPTIONAL_PKGS[]` | Packages that are helpful but not required |
| `CONFLICT_SERVICES[]` | Services that must NOT be running |
| `REQUIRED_SERVICES[]` | Services that must be running |
| `SYSCTL_PARAMS{}` | Associative array of `param=value` |
| `MIN_KERNEL` | Minimum kernel version string |
| `NEEDS_CGROUPS_V2` | true/false |

---

## Script Header Template

```bash
#!/usr/bin/env bash
# =============================================================================
# <APP_NAME> Pre-Installation System Checker for Raspberry Pi
# Version:  1.0.0
# Target:   Raspberry Pi 4/5 · Debian Bookworm/Bullseye · aarch64
# Purpose:  Validate all prerequisites before installing <APP_NAME>
# Usage:    sudo ./<script>.sh
# Output:   Console (ANSI color) + log: /tmp/<script>_YYYYMMDD_HHMMSS.log
# Exit:     0 = no critical failures | 1 = one or more failures present
# Safe:     Read-only checks; Section 15 applies fixes (requires sudo)
# =============================================================================
```

---

## Validation Gate

Run these in the container before presenting the script:

```bash
bash -n script.sh && echo "SYNTAX OK"
grep -n '((' script.sh | grep -v 'version_gte\|#' | grep -v 'inc ' && echo "WARNING: raw (( found"
grep -n 'warn\|fail' script.sh | grep -v '#' | grep -v ' — ' && echo "WARNING: message without fix hint"
wc -l script.sh  # warn user if >1500 lines
```

---

## Reference Files

Read these when relevant to the target application:

- `references/sysctl-catalog.md` — sysctl params by use case (networking, containers, memory)
- `references/port-registry.md` — common application port assignments
- `references/package-matrix.md` — required/optional packages by category
- `references/pi-hardware-matrix.md` — Pi model specs, memory limits, PCIe support