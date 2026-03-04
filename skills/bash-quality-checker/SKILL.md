---
name: bash-quality-checker
version: 1.0.0
description: >
  Review, audit, and improve bash scripts targeting Raspberry Pi (Debian/Bookworm,
  aarch64, bash 5.x). Use this skill whenever the user wants to: check a bash script
  for bugs, portability issues, or reliability problems; improve script quality before
  running on a Pi; audit shell scripts for correctness; add error handling or logging
  to a script; or asks "is this script safe to run?", "review my bash script",
  "make this script more robust", "check my shell script for issues". Always use
  this skill when the user uploads or pastes a bash script and asks for feedback.
targets: [bash-5.x, raspberry-pi, debian-bookworm, aarch64]
changelog:
  - 1.0.0: Initial version. Patterns distilled from Pi5 PodCloud precheck debugging.
---

# Bash Script Quality Checker Skill

## Purpose

Audit bash scripts for correctness, portability, reliability, and clarity —
specifically targeting Raspberry Pi systems (Debian Bookworm, bash 5.x, aarch64).
Produce a structured report and, when asked, a corrected version of the script.

---

## Audit Dimensions

Evaluate every script across these six dimensions, in this order:

### 1. Syntax & Parseability
- Does `bash -n script.sh` pass with zero errors?
- Are all brackets, quotes, heredocs, and function braces balanced?
- Are all variable expansions properly quoted?

### 2. Reliability & Error Handling
These are the most impactful issues. Check each one explicitly:

**Strict mode usage:**
```bash
# GOOD — explicit error handling with pipefail
set -uo pipefail

# BAD — set -e causes silent unexpected aborts on test commands
set -e        # or: set -euo pipefail
```

**Arithmetic increments on counters that start at zero:**
```bash
# BUG — ((VAR++)) returns exit code 1 when VAR=0; kills script under any strict mode
((COUNT++))

# FIX
inc() { local v="${!1}"; eval "$1=$(( v + 1 ))"; return 0; }
inc COUNT
```

**Pipeline + fallback in command substitution (the "silent concatenation" bug):**
```bash
# BUG — with pipefail, if cmd exits non-zero, BOTH the pipeline output AND the
# fallback are captured. Produces e.g. "degradedunknown" instead of "degraded".
STATE=$(cmd_that_exits_nonzero 2>/dev/null | head -1 | tr -d '[:space:]' || echo "unknown")

# FIX — two-step: capture raw with || true, then process
STATE_RAW=$(cmd_that_exits_nonzero 2>/dev/null || true)
STATE=$(printf "%s" "$STATE_RAW" | head -1 | tr -d '[:space:]')
STATE="${STATE:-unknown}"
```

Commands that exit non-zero on non-error conditions (must apply this pattern):
`systemctl is-system-running` (exits 1 for "degraded"), `systemctl is-active`
(exits 3 for "inactive"), `grep` (exits 1 for no match), `docker info` (exits
non-zero when daemon is stopped).

**Unbound variables under `set -u`:**
```bash
# BUG — variable only set in one branch; triggers set -u error in other branch
if [[ condition ]]; then
  MY_VAR="something"
fi
echo "$MY_VAR"   # Aborts if condition was false

# FIX — initialize all variables at the top of the script
MY_VAR=""
```

**Count variables from pipes:**
```bash
# BUG — wc -l, grep -c can embed trailing newlines; tee logging doubles output
COUNT=$(cmd | wc -l)

# FIX
COUNT_RAW=$(cmd 2>/dev/null | wc -l || echo 0)
COUNT=$(echo "$COUNT_RAW" | tr -d '[:space:]')
COUNT="${COUNT:-0}"
```

**Missing `|| true` on commands used in boolean contexts:**
```bash
# BUG — with pipefail, grep failure propagates and aborts
if grep -q "pattern" file; then ...

# FIX — only necessary inside pipelines; direct if-conditions are fine
result=$(grep -c "pattern" file || true)
```

### 3. Portability (Debian/Bookworm/aarch64 specific)

**Shell compatibility:**
- Shebang must be `#!/usr/bin/env bash` — never `/bin/sh` if using bash features
- `bash` version on Bookworm is 5.2 — all bash 4.x+ features are available
- `aarch64` — no `x86`-specific assumptions; no hardcoded paths like `/usr/lib/x86_64-linux-gnu/`

**Commands that may be missing on a minimal Pi install:**
- `bc` — not always installed; replace with `$(( ))` integer arithmetic or `awk`
- `python3` — present on Bookworm but version may vary; always check before use
- `jq` — not default; check with `has_cmd jq`
- `netstat` — deprecated; use `ss` instead (both should be tried)
- `conntrack` — CLI binary may be absent even when `nf_conntrack` kernel module is loaded

**Paths that differ between Bookworm and Bullseye:**
```bash
# Boot config
BOOT_CONFIG=""
for CFG in /boot/firmware/config.txt /boot/config.txt; do
  [[ -f "$CFG" ]] && BOOT_CONFIG="$CFG" && break
done
```

**Do not assume tools exist — always guard:**
```bash
has_cmd() { command -v "$1" >/dev/null 2>&1; }
has_cmd docker || { echo "docker not found"; exit 1; }
```

### 4. Logic & Correctness

**Version comparisons — never use string comparison:**
```bash
# BUG — string comparison of versions is incorrect ("10" < "9" as strings)
if [[ "$VER" > "1.21" ]]; then ...

# FIX — use sort -V
version_gte() { printf '%s\n%s' "$2" "$1" | sort -V -C; }
version_gte "$VER" "1.21" && echo "OK"
```

**HTTP endpoint assumptions:**
```bash
# BUG — Docker registry root returns 404 (correct behavior, not an error)
curl -sf https://registry-1.docker.io/ && echo "reachable"

# FIX — use /v2/ which returns 401 (auth challenge = server reachable)
HTTP=$(curl -o /dev/null -sw "%{http_code}" https://registry-1.docker.io/v2/ 2>/dev/null || echo "000")
[[ "$HTTP" != "000" ]] && echo "reachable (HTTP $HTTP)"
```

**Firewall check-then-fix ordering:**
```bash
# BUG — warning is recorded before fix runs; stays in output even if fix succeeds
warn "port blocked"
sudo ufw allow 6443/tcp

# FIX — fix first, warn only on fix failure
if sudo ufw allow 6443/tcp 2>/dev/null; then
  pass "port 6443 opened"
else
  warn "port 6443 blocked — run: sudo ufw allow 6443/tcp"
fi
```

### 5. Structure & Clarity

**Section organization — every script should have:**
```bash
# =============================================================================
#  N. SECTION NAME
# =============================================================================
```

Sections should be: metadata header, strict mode, constants, helpers, then logical
groups of checks/actions. Never mix concerns (e.g., don't run a fix inside a check
section; put fixes in a dedicated section).

**Function design:**
- Each function does one thing and has a clear name
- Helper functions (output, version comparison, command detection) are declared before use
- No function longer than ~30 lines without a comment explaining its purpose

**Inline documentation standards:**
- Every non-obvious command has a comment explaining WHY (not just what)
- Every `warn`/`fail` message is actionable — includes the exact remediation command
- Magic numbers are named constants: `MIN_RAM_KB=8388608  # 8 GB`
- External tool behaviors that aren't obvious are documented:
  ```bash
  # Note: registry /v2/ returns 401 (auth challenge) = server reachable, TLS working
  # This is correct behavior — not an error
  ```

**Variable naming:**
- SCREAMING_SNAKE_CASE for globals and constants
- lowercase for local variables inside functions
- Suffix `_RAW` for unprocessed command output, stripped version without suffix

### 6. Security & Privilege

**Principle of least privilege:**
- Scripts should detect their own privilege level and degrade gracefully
- Never hardcode sudo use — check first: `sudo -n true >/dev/null 2>&1`
- Document which sections require root/sudo in comments and in the header

**Input handling:**
- Never `eval` user-supplied input
- Quote all variable expansions: `"$VAR"` not `$VAR`
- Use `--` to end option processing when passing variables to commands:
  `grep -- "$PATTERN" "$FILE"`

**Sensitive output:**
- Never log passwords, tokens, or keys — even to a log file
- If a script creates files with sensitive content, set `chmod 600` immediately

---

## Report Format

Produce the audit report in this exact structure:

```
## Bash Script Quality Report
**Script:** <filename>
**Target:** Raspberry Pi (Debian Bookworm, bash 5.x, aarch64)
**Date:** <date>

### Summary
| Dimension | Score | Issues Found |
|---|---|---|
| Syntax | PASS/WARN/FAIL | N |
| Reliability | PASS/WARN/FAIL | N |
| Portability | PASS/WARN/FAIL | N |
| Logic | PASS/WARN/FAIL | N |
| Structure | PASS/WARN/FAIL | N |
| Security | PASS/WARN/FAIL | N |

**Overall: PASS / WARN / FAIL**

### Critical Issues (must fix)
[numbered list with: file:line, issue description, exact fix]

### Warnings (should fix)
[numbered list with: file:line, issue description, recommended fix]

### Suggestions (consider)
[numbered list with low-priority improvements]

### Positive Patterns (what's done well)
[brief acknowledgment of good practices found]
```

---

## Corrected Script Delivery

When producing a corrected version of the script:

1. Preserve all original logic — only fix issues found in the audit
2. Add a `# FIXED: reason` comment on every changed line
3. Add a `# ADDED: reason` comment on every inserted line
4. Update the script version number in the header (bump patch: 1.0.0 → 1.0.1)
5. Add a changelog entry to the header block
6. Run `bash -n` on the corrected script before delivering

---

## Quick Reference: Common Pi/Debian Gotchas

| Pattern | Issue | Fix |
|---|---|---|
| `((VAR++))` | Exits 1 when VAR=0 | Use `inc VAR` helper |
| `cmd \| pipe \|\| echo fallback` | Both outputs captured when cmd exits non-zero | Two-step: raw capture then process |
| `set -e` | Silent abort on test commands | Remove; use explicit error handling |
| `$(cmd \| wc -l)` | Trailing newline in count | Add `tr -d '[:space:]'` |
| `[[ "$A" > "$B" ]]` for versions | Lexicographic, not version-aware | Use `sort -V -C` |
| `/bin/sh` shebang with bash syntax | Fails on strict sh | Use `#!/usr/bin/env bash` |
| `bc` for math | Not always installed | Use `$(( ))` or `awk` |
| Hardcoded `/boot/config.txt` | Wrong on Bookworm | Check both paths |
| `netstat` for port checks | Deprecated on Bookworm | Use `ss` with `netstat` fallback |
| Variables set in one branch only | `set -u` abort in other branch | Initialize all vars at top |
| Unquoted `$VAR` in commands | Word splitting on spaces | Always quote: `"$VAR"` |

---

## Validation Commands

Always run these on the input script first:

```bash
# Syntax
bash -n "$SCRIPT" 2>&1

# ShellCheck (if available)
shellcheck -S warning "$SCRIPT" 2>/dev/null

# Find raw arithmetic increments (potential exit-code trap)
grep -n '((' "$SCRIPT" | grep -v 'version_gte\|sort\|#'

# Find pipeline+fallback patterns that could concatenate
grep -n '|| echo\||| true' "$SCRIPT" | grep '\$('

# Find unquoted variable expansions (basic check)
grep -n '\$[A-Z_][A-Z_]*[^"]' "$SCRIPT" | grep -v '#\|echo\|="'

# Find messages without fix hints (warn/fail without — )
grep -n 'warn\|fail' "$SCRIPT" | grep '"' | grep -v ' — \|#'
```