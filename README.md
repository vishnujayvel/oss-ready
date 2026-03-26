# oss-ready

Claude Code skill that audits repositories for open-source readiness before pushing to public GitHub.

Scans for PII, secrets, hardcoded paths, license issues, gitignore gaps, and git history problems. Generates a severity-ranked remediation checklist.

## Install

Copy `SKILL.md` to your Claude Code skills directory:

```bash
cp SKILL.md ~/.claude/skills/oss-ready/SKILL.md
```

Or clone this repo and symlink:

```bash
git clone https://github.com/vishnu/oss-ready.git
ln -s $(pwd)/oss-ready/SKILL.md ~/.claude/skills/oss-ready/SKILL.md
```

## Usage

In any repository, tell Claude Code:

```
oss-ready
```

Or use natural language:

```
Is this repo ready for open source?
Check this codebase before I push it public.
Run a PII scan on this project.
```

## What It Scans

| Scanner | Severity | What It Finds |
|---------|----------|---------------|
| PII Scanner | HIGH | Emails, phone numbers, hostnames, usernames in paths |
| Secrets Scanner | CRITICAL | API keys, tokens, passwords, .env files, private keys |
| Path Scanner | MEDIUM | `/Users/X`, `/home/X`, `/opt/homebrew`, hardcoded absolute paths |
| License Checker | MEDIUM | Missing or unrecognized LICENSE file |
| Gitignore Auditor | LOW | Missing `.gitignore` entries for common files |
| Git History Auditor | HIGH | Author emails with machine hostnames, local addresses |

All 6 scanners run as parallel subagents for speed.

## Output

Generates a markdown report with:

- **Summary** — file count, severity breakdown, READY/NOT READY/NEEDS WORK verdict
- **Findings** — organized by severity (CRITICAL > HIGH > MEDIUM > LOW) with file:line references
- **Remediation Steps** — prioritized action items

### Verdict Logic

- **READY**: 0 Critical, 0 High findings
- **NEEDS WORK**: 0 Critical, but has High/Medium findings
- **NOT READY**: Any Critical findings

## What This Is NOT

- Not a CI/CD tool — this is a pre-push check you run manually
- Not an auto-fixer — it reports, you fix
- Not a dependency scanner — use `npm audit` / `cargo audit` for that

## License

MIT
