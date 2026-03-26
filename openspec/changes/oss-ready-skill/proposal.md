# Proposal: oss-ready — Open Source Readiness Auditor

## Problem

Before pushing a private repository to public GitHub, you need to manually audit every file for PII, secrets, hardcoded paths, and personal customizations. This is tedious, error-prone, and easy to skip. We just did this audit by hand for the Wishloop skill and it caught 5 instances of PII, 3 hardcoded paths, and a missing .gitignore — all of which would have been exposed publicly.

## Goals

Build a Claude Code skill called `oss-ready` that automates the pre-publish audit:

1. **PII Scanner** — Detect names, emails, phone numbers, machine hostnames (e.g., `Vishnus-MacBook-Air`), usernames in paths (`/Users/vishnu`), and personal references in any file
2. **Secrets Scanner** — Find API keys, tokens, passwords, credentials, .env files, and anything matching common secret patterns (AWS keys, GitHub tokens, Slack webhooks, etc.)
3. **Path Hardcoding Scanner** — Flag user-specific paths (`/Users/X`, `/home/X`, `/opt/homebrew/bin/X`), architecture-specific paths, and paths that won't work on other machines
4. **License Checker** — Verify LICENSE file exists and is a recognized open-source license (MIT, Apache-2.0, GPL, etc.)
5. **Gitignore Auditor** — Check for missing .gitignore entries (node_modules, .env, .DS_Store, build artifacts, IDE configs)
6. **Git History Auditor** — Check git author emails for machine hostnames or local-only addresses
7. **Remediation Checklist** — Generate a markdown checklist with specific fixes, organized by severity (CRITICAL/HIGH/MEDIUM/LOW)

## Scope

- The skill is a Claude Code SKILL.md (no runtime, no server, no build system)
- It uses Claude's built-in tools (Grep, Glob, Read, Bash) to scan files
- It outputs results directly in the conversation + optionally writes a report file
- It should work on ANY repository, not just Claude Code skills

## Non-Goals

- No CI/CD integration (this is a pre-push check, not a pipeline)
- No automated fixing (it reports, the user fixes)
- No cloud-based scanning (everything runs locally)
- No dependency vulnerability scanning (use `npm audit` / `cargo audit` for that)

## Technology Choices

- **Pure SKILL.md** — no dependencies, no build system, no package manager
- **Regex patterns** for secret detection (no external tools needed)
- **Git CLI** for history analysis
- **Glob + Grep** tools for file scanning

## Testing Strategy

- **Evals** — Create eval test cases in evals/evals.json with positive triggers ("audit this repo for open source readiness") and negative triggers ("fix this bug", "deploy to production")
- **Manual testing** — Run the skill against the Wishloop repo (known good/bad state) and verify it catches the same issues we found manually
- **Edge cases** — Binary files, large repos, repos with no git history, repos with no files

## Architecture

The skill runs as a single-pass audit with 6 parallel scanners:

```
User: "audit this repo" or "oss-ready" or "check if this is ready for open source"
  |
  v
[1. PII Scanner]      — grep for names, emails, hostnames, /Users/X patterns
[2. Secrets Scanner]   — grep for API keys, tokens, .env files, password patterns
[3. Path Scanner]      — grep for /Users/, /home/, /opt/homebrew, hardcoded absolute paths
[4. License Checker]   — glob for LICENSE*, verify content is recognized
[5. Gitignore Auditor] — check for .gitignore, verify common entries present
[6. Git History Audit] — git log --format for author emails with local hostnames
  |
  v
[Remediation Checklist] — aggregate findings, classify severity, generate markdown
```

Each scanner runs as a subagent for parallelism. Results are aggregated into a single report.

## Output Format

```markdown
# OSS Readiness Report

## Summary
- Repository: <name>
- Files scanned: <N>
- Critical: <N> | High: <N> | Medium: <N> | Low: <N>
- Verdict: READY / NOT READY / NEEDS WORK

## Findings

### CRITICAL
- [ ] `src/config.js:12` — AWS_SECRET_ACCESS_KEY exposed
- [ ] `.env` — Not in .gitignore, contains DATABASE_URL

### HIGH
- [ ] `README.md:5` — Personal email: vishnu@example.com
- [ ] Git author email uses machine hostname: user@MacBook-Pro.local

### MEDIUM
- [ ] `scripts/deploy.sh:3` — Hardcoded path: /Users/vishnu/workspace
- [ ] No LICENSE file found

### LOW
- [ ] .gitignore missing: .DS_Store
- [ ] .gitignore missing: .idea/

## Remediation Steps
1. Remove or redact all CRITICAL findings before pushing
2. Fix HIGH findings (PII, git history)
3. Address MEDIUM findings (paths, license)
4. LOW findings are optional but recommended
```

## Trigger Words

The skill should trigger on:
- "oss-ready", "open source ready", "open source readiness"
- "audit this repo for public", "check before pushing public"
- "PII scan", "secrets scan", "is this safe to open source"
- "pre-publish check", "ready for GitHub public"

Should NOT trigger on:
- Generic "audit" without open-source context
- "deploy", "CI/CD", "pipeline"
- "fix this bug", "add tests"
- "security vulnerability scan" (different scope — that's dependency scanning)
