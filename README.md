# oss-ready

Claude Code skill that audits repositories for open-source readiness. 5 features in one skill: security scanning, repo health scoring, open source coaching, README auditing, and release readiness checks.

## Install

Copy `SKILL.md` to your Claude Code skills directory:

```bash
cp SKILL.md ~/.claude/skills/oss-ready/SKILL.md
```

Or clone this repo and symlink:

```bash
git clone https://github.com/vishnujayvel/oss-ready.git
ln -s $(pwd)/oss-ready/SKILL.md ~/.claude/skills/oss-ready/SKILL.md
```

## Features

### 1. Security Scanner

Scans for PII, secrets, hardcoded paths, license issues, gitignore gaps, and git history problems. Runs 6 parallel scanners for speed.

```
secrets scan
PII scan
```

### 2. Repo Health Scorecard

Scores your repository against 12 community health checks (README, LICENSE, CONTRIBUTING, CODE_OF_CONDUCT, SECURITY, etc.) and assigns a letter grade A+ through F with visual progress bars.

```
repo health
scorecard
health check
```

### 3. Open Source Coach

Interactive session that walks you through whether you should open source a project. Covers viability, licensing, positioning, and maintenance capacity. Questions asked one at a time.

```
should I open source this?
oss coach
```

### 4. README Auditor

Scores your existing README against 11 best practices (title, badges, install instructions, usage examples, API docs, contributing, license, TOC, visuals, support info, broken links). Provides section-by-section scoring with specific suggestions.

```
readme audit
check my readme
readme score
```

### 5. Release Readiness Check

Pre-push gate that checks: changelog, version tags, TODOs in code, debug statements, package version vs git tag, build/test status, uncommitted changes, branch freshness, and .env.example existence.

```
release check
ready to ship
ship check
```

### Full Audit

Run all features (except Coach, which is interactive) in one go:

```
oss-ready
full audit
```

## Output

### Full Audit Report

```text
  ══════════════════════════════════════════
    OPEN SOURCE READINESS REPORT
    my-project  |  2026-03-27
  ══════════════════════════════════════════

    GRADE: B+    VERDICT: NEEDS WORK

  ──────────────────────────────────────────

    🟢 Security      ████████░░  83%
    🟢 Repo Health   ██████████ 100%
    🟡 README        █████░░░░░  50%
    🟢 Release       ███████░░░  70%

  ──────────────────────────────────────────
    ... findings by severity ...
    ... remediation steps ...
  ══════════════════════════════════════════
```

All output rendered directly in Claude's text response (no collapsed Bash output).

### Verdict Logic

| Feature | READY | NEEDS WORK | NOT READY |
|---------|-------|------------|-----------|
| Security | 0 Critical + 0 High | 0 Critical, has High/Medium | Any Critical |
| Release | 0 HIGH failures | 1+ HIGH failures | CRITICAL security issues |
| Overall | Grade >= B-, 0 Critical | Grade >= D or HIGH only | Any Critical or grade < D |

## What This Is NOT

- Not a CI/CD tool — this is a skill you run in Claude Code
- Not an auto-fixer — it reports, you fix
- Not a dependency scanner — use `npm audit` / `cargo audit` for that
- Not using external tools — pure prompt, no installations needed

## License

MIT
