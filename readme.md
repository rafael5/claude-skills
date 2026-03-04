# Skills Library — GitHub Setup & Version Control Guide

## Repository Structure

```
skills-library/
├── README.md                          # Overview, how to use, how to contribute
├── CHANGELOG.md                       # Cross-skill version history
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── skill-improvement.md       # Template for filing skill gaps
│       └── new-skill-request.md       # Template for new skill requests
├── skills/
│   ├── pi-system-precheck/
│   │   ├── SKILL.md                   # Primary skill definition (< 500 lines)
│   │   ├── evals/
│   │   │   └── evals.json             # Test cases and assertions
│   │   └── references/                # Deep reference material loaded on demand
│   │       ├── sysctl-catalog.md
│   │       ├── port-registry.md
│   │       ├── package-matrix.md
│   │       └── pi-hardware-matrix.md
│   └── bash-quality-checker/
│       ├── SKILL.md
│       ├── evals/
│       │   └── evals.json
│       └── references/
│           └── common-gotchas.md
└── templates/
    └── SKILL_TEMPLATE.md              # Blank skill template for new skills
```

---

## Initial Setup (one-time)

```bash
# 1. Create the repo on GitHub first (via UI or gh CLI)
gh repo create skills-library --private --description "Personal Claude skills library"

# 2. Initialize locally
mkdir ~/skills-library && cd ~/skills-library
git init
git remote add origin git@github.com:<your-username>/skills-library.git

# 3. Copy your existing skill files in
cp -r /path/to/skills/* skills/

# 4. Initial commit
git add .
git commit -m "feat: initial skills library — pi-system-precheck v1.0.0, bash-quality-checker v1.0.0"
git push -u origin main
```

---

## Versioning Convention

Skills use **semantic versioning** (MAJOR.MINOR.PATCH):

| Change type | Version bump | Example |
|---|---|---|
| New bug pattern added to checker | PATCH | 1.0.0 → 1.0.1 |
| New section or major capability added | MINOR | 1.0.1 → 1.1.0 |
| Breaking change to output format or structure | MAJOR | 1.1.0 → 2.0.0 |

The version lives in two places — keep them in sync:
1. YAML frontmatter: `version: 1.0.1`
2. Script header comments in any generated scripts

---

## Workflow: Adding New Knowledge to a Skill

After each Claude session where a skill produces incorrect output or you discover
a new pattern/bug:

```bash
cd ~/skills-library

# 1. Create a branch for the update
git checkout -b fix/bash-quality-checker-new-pattern

# 2. Edit the SKILL.md
#    - Add the new pattern to the relevant section
#    - Add a changelog entry to the YAML frontmatter
#    - Bump the version number

# Example changelog entry in SKILL.md frontmatter:
# changelog:
#   - 1.0.1: Added UFW check-then-fix ordering pattern
#   - 1.0.0: Initial version

# 3. Add a test case to evals.json if applicable

# 4. Commit with a descriptive message
git add skills/bash-quality-checker/
git commit -m "fix(bash-quality-checker): add UFW check-then-fix pattern v1.0.1

- Added pattern: firewall ports should be opened BEFORE recording a warning,
  not after. Warning recorded before fix stays in output even if fix succeeds.
- Added eval case #4 testing this pattern
- Bump to 1.0.1"

# 5. Push and merge
git push origin fix/bash-quality-checker-new-pattern
gh pr create --title "fix: UFW ordering pattern" --body "..." --base main
# Or just push directly to main for a personal library
git checkout main && git merge fix/bash-quality-checker-new-pattern
git push origin main
```

---

## Workflow: Using Skills with Claude

### Option A — Paste the skill content directly

For single sessions, copy the SKILL.md content into your conversation:
```
Please read and follow this skill definition:
<paste SKILL.md content>

Now: [your task]
```

### Option B — Reference by path (when using Claude with file access)

If you're using Claude Code or a setup with filesystem access:
```
Read /Users/rafael/skills-library/skills/pi-system-precheck/SKILL.md
then generate a precheck script for [application]
```

### Option C — Claude.ai with file upload

Upload the SKILL.md file as an attachment, then give your task.

### Option D — System prompt injection (advanced)

If you use the API or Claude Code with a system prompt, prepend your most-used
skills directly into the system prompt so they're always active.

---

## Workflow: Capturing Session Learnings

After a debugging session (like the Pi precheck sessions), extract new patterns:

```bash
# Create a learning note in the skill
cat >> skills/pi-system-precheck/references/session-learnings.md << 'EOF'

## Session: 2026-03-04 — UFW auto-fix ordering bug

**Symptom:** 10 UFW port warnings appeared in summary even though the script
successfully opened all ports.

**Root cause:** The script called warn() for each missing port BEFORE calling
sudo ufw allow. The WARNINGS[] array was already populated before fixes ran.

**Pattern added:** Check-then-fix ordering (see SKILL.md Section: Firewall checks)

**Pattern added to bash-quality-checker:** Dimension 4, Logic & Correctness,
"Firewall check-then-fix ordering"
EOF

git add .
git commit -m "docs: capture UFW ordering lesson from 2026-03-04 session"
```

---

## Workflow: Cross-referencing Between Skills

When a pattern is relevant to both skills (e.g., a bash bug that the quality checker
should catch AND the precheck skill should avoid), add it to both:

```bash
# Tag cross-references in SKILL.md with a comment
# See also: bash-quality-checker/SKILL.md — Dimension 2, Pattern: pipeline+fallback
```

---

## Tags and Discoverability

Add a `topics` section to each SKILL.md frontmatter for searchability:

```yaml
topics: [raspberry-pi, bash, pre-installation, debian, aarch64, docker, kubernetes]
```

And tag the GitHub repo:
```bash
gh repo edit skills-library --add-topic raspberry-pi --add-topic bash-scripting --add-topic claude-skills
```

---

## Periodic Review Checklist

Run this monthly or after any major session:

- [ ] Have all new bug patterns from recent sessions been added to the relevant skill?
- [ ] Are version numbers bumped in both YAML frontmatter and any templates?
- [ ] Are new test cases added for any new patterns?
- [ ] Is CHANGELOG.md updated?
- [ ] Do reference files need updating (new package versions, new Pi models, etc.)?

---

## Connecting Claude to GitHub (Optional, Advanced)

If you want Claude to be able to read and update skills directly from GitHub:

### Using Claude's web fetch capability
```
Fetch this skill from GitHub:
https://raw.githubusercontent.com/<username>/skills-library/main/skills/pi-system-precheck/SKILL.md
Then use it to generate a precheck script for [application].
```

### Using GitHub Codespaces
Open your skills repo in Codespaces, then use Claude Code — it has direct filesystem
access and can read, update, and commit skill files as part of your session.

### MCP GitHub server (most powerful)
If you set up the GitHub MCP server with Claude Code:
```bash
# In your Claude Code config, add:
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "<your-token>" }
    }
  }
}
```
This lets Claude read, create, and update files in your skills repo directly
during a session, and commit the changes when done.

---

## Skill File Size Guidelines

| Content | Where it goes | Size limit |
|---|---|---|
| Trigger logic, core patterns, output format | SKILL.md | < 500 lines |
| Deep reference catalogs (sysctl params, port lists) | references/*.md | No limit |
| Test cases and assertions | evals/evals.json | No limit |
| Session learning notes | references/session-learnings.md | No limit |

When SKILL.md approaches 500 lines, extract catalog-style content to references/
and add a pointer: "Read references/sysctl-catalog.md for the full parameter list."