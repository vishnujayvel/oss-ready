# oss-ready

A Claude Code skill that audits repositories for open-source readiness before pushing to public GitHub.

## What this project IS

- A Claude Code skill (SKILL.md + evals + optional ref docs)
- Pure prompt-based — no runtime, no server, no dependencies
- Uses Claude's built-in tools (Grep, Glob, Read, Bash) for scanning
- Outputs a remediation checklist with severity levels

## What this project is NOT

- Not a CI/CD tool or pipeline integration
- Not an automated fixer (it reports, user fixes)
- Not a dependency vulnerability scanner (use npm audit for that)

## File structure

```
SKILL.md              — The skill definition (main file)
evals/evals.json      — Activation test cases
README.md             — Public documentation
LICENSE               — MIT
.gitignore            — Standard exclusions
```

## Key design decisions

- Scanners should run as parallel subagents for speed
- Secret patterns use regex, not external tools
- Git history audit uses `git log --format` to check author emails
- Report format is markdown with checkboxes for actionability
- Severity levels: CRITICAL (secrets), HIGH (PII), MEDIUM (paths/license), LOW (gitignore)

## Known Pitfalls (from prior runs)

- [workflow] Always test the skill against a known-good/bad repo before shipping
- [loki] Loki often writes everything in ONE atomic commit — this is normal
- [testing] Create evals with both positive and negative triggers
