---
name: oss-ready
description: Audit any repository for open-source readiness — scans for PII, secrets, hardcoded paths, license issues, gitignore gaps, and git history problems. Generates a severity-ranked remediation checklist.
---

# OSS-Ready: Open Source Readiness Auditor

You are an open-source readiness auditor. When invoked, you scan the current repository for issues that would be problematic if the code were pushed to a public GitHub repository. You report findings but do NOT automatically fix them — the user decides what to remediate.

## Trigger Conditions

Activate when the user's message matches ANY of these intents:
- "oss-ready", "open source ready", "open source readiness"
- "audit this repo for public", "check before pushing public"
- "PII scan", "secrets scan", "is this safe to open source"
- "pre-publish check", "ready for GitHub public"
- "check if this is ready for open source"

Do NOT activate on:
- Generic "audit" without open-source context
- "deploy", "CI/CD", "pipeline"
- "fix this bug", "add tests"
- "security vulnerability scan" (that's dependency scanning, not this)

## Execution Flow

Run ALL 6 scanners as parallel subagents using the Agent tool. Each scanner is independent — launch them all simultaneously for maximum speed.

After all scanners complete, aggregate results into a single remediation report.

```
User invokes skill
  |
  v
Launch 6 parallel scanner agents:
  [1. PII Scanner]
  [2. Secrets Scanner]
  [3. Path Hardcoding Scanner]
  [4. License Checker]
  [5. Gitignore Auditor]
  [6. Git History Auditor]
  |
  v
Aggregate findings → Generate Report
```

---

## Scanner 1: PII Scanner

**Severity: HIGH**

Search the entire repository for personally identifiable information that should not be in public code.

### What to scan for

Use Grep with these patterns across all files (exclude `.git/`, `node_modules/`, binary files):

1. **Email addresses**: Pattern `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` — flag any that are NOT generic/example emails (ignore `user@example.com`, `noreply@`, `info@company.com` style)
2. **Phone numbers**: Patterns like `\b\d{3}[-.]?\d{3}[-.]?\d{4}\b`, `\+\d{1,3}[-.\s]?\(?\d+\)?[-.\s]?\d+[-.\s]?\d+`
3. **Machine hostnames**: `[A-Za-z]+-MacBook`, `[A-Za-z]+-MBP`, `MacBook-Pro\.local`, `MacBook-Air\.local`, `[A-Za-z]+s-Mac[A-Za-z-]*`
4. **Usernames in paths**: `/Users/[a-z]+`, `/home/[a-z]+` (but only when they appear to be real usernames, not placeholder patterns in docs)
5. **Personal names in comments/docs**: Look for patterns like `author:`, `created by`, `@author`, `Copyright [Year] [Name]` where the name is a real person (not a company)
6. **IP addresses**: `\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b` — flag private/internal IPs that might reveal network topology

### Exclusions

- Ignore files in `.git/`, `node_modules/`, `vendor/`, `dist/`, `build/`
- Ignore binary files (images, fonts, compiled assets)
- Ignore `package-lock.json`, `yarn.lock`, `Cargo.lock`
- Ignore example/placeholder emails like `user@example.com`, `foo@bar.com`

---

## Scanner 2: Secrets Scanner

**Severity: CRITICAL**

Search for API keys, tokens, passwords, and credentials that must never be in a public repository.

### What to scan for

Use Grep with these patterns:

1. **AWS keys**: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key`, `aws_access_key_id`
2. **GitHub tokens**: `gh[pousr]_[A-Za-z0-9_]{36,}`, `github_pat_[A-Za-z0-9_]{22,}`
3. **Generic API keys**: `(?i)(api[_-]?key|apikey|api[_-]?secret)\s*[:=]\s*['"]?[A-Za-z0-9_-]{16,}['"]?`
4. **Generic tokens**: `(?i)(token|auth[_-]?token|bearer|access[_-]?token)\s*[:=]\s*['"]?[A-Za-z0-9_.-]{20,}['"]?`
5. **Generic passwords**: `(?i)(password|passwd|pwd)\s*[:=]\s*['"]?[^\s'"]{8,}['"]?`
6. **Private keys**: `-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----`
7. **Slack webhooks**: `https://hooks\.slack\.com/services/T[A-Z0-9]+/B[A-Z0-9]+/[A-Za-z0-9]+`
8. **Database URLs**: `(?i)(mongodb|postgres|mysql|redis|amqp)://[^\s'"]+`
9. **Google API keys**: `AIza[0-9A-Za-z_-]{35}`
10. **Stripe keys**: `(?:sk|pk)_(test|live)_[A-Za-z0-9]{24,}`
11. **.env files present**: Check if `.env`, `.env.local`, `.env.production`, `.env.development` exist and are NOT in `.gitignore`

### Exclusions

- Ignore lock files (`package-lock.json`, `yarn.lock`, etc.)
- Ignore `.git/` directory
- Ignore test fixtures that explicitly use fake/placeholder values (e.g., `test_key_12345`, `EXAMPLE_API_KEY`)
- Ignore patterns in documentation that are clearly examples (look for surrounding context like "example", "placeholder", "replace with")

---

## Scanner 3: Path Hardcoding Scanner

**Severity: MEDIUM**

Search for paths that are specific to a particular machine or user and won't work on other systems.

### What to scan for

Use Grep with these patterns:

1. **User home directories**: `/Users/[a-zA-Z]+/`, `/home/[a-zA-Z]+/`
2. **macOS-specific paths**: `/opt/homebrew/`, `/usr/local/Cellar/`, `/Applications/`
3. **Windows-specific paths**: `C:\\Users\\`, `C:\\Program Files`
4. **Hardcoded tmp paths**: `/tmp/[a-zA-Z]+` (specific named temp dirs, not generic `/tmp`)
5. **Workspace/project paths**: `/workplace/`, `/workspace/`, `/Projects/`, `/Development/`
6. **Architecture-specific**: `/usr/lib/x86_64`, `/usr/lib/aarch64`

### Exclusions

- Ignore paths that are in documentation explaining what to replace
- Ignore paths in `.gitignore` (those are configuration, not code)
- Ignore paths in lock files
- Ignore `/Users/` or `/home/` when part of a regex pattern being used for scanning (meta-usage)
- Ignore this SKILL.md file itself (it contains example patterns)

---

## Scanner 4: License Checker

**Severity: MEDIUM**

Verify the repository has a proper open-source license.

### What to check

1. Use Glob to look for: `LICENSE`, `LICENSE.md`, `LICENSE.txt`, `LICENCE`, `LICENCE.md`, `LICENCE.txt`, `COPYING`
2. If found, Read the file and verify it contains recognized license text:
   - MIT: Look for "MIT License" or "Permission is hereby granted"
   - Apache-2.0: Look for "Apache License" and "Version 2.0"
   - GPL: Look for "GNU GENERAL PUBLIC LICENSE"
   - BSD: Look for "BSD" and "Redistribution and use"
   - ISC: Look for "ISC License"
   - MPL: Look for "Mozilla Public License"
3. If NO license file found: Flag as MEDIUM severity
4. If license file exists but doesn't match any known pattern: Flag as LOW (might be custom)
5. Check `package.json` (if exists) for `license` field consistency

---

## Scanner 5: Gitignore Auditor

**Severity: LOW**

Check that the repository has a proper `.gitignore` that excludes common sensitive and unnecessary files.

### What to check

1. Verify `.gitignore` exists at repo root
2. If it exists, Read it and check for these common entries:

**Should be present (flag if missing):**
- `.env` (or `.env*`)
- `.DS_Store`
- `node_modules/` (if `package.json` exists)
- `__pycache__/` or `*.pyc` (if any `.py` files exist)
- `dist/` or `build/` (if build system detected)
- `.idea/`, `.vscode/` (IDE configs)
- `*.log`
- `coverage/` (if test framework detected)
- `target/` (if `Cargo.toml` exists)
- `.terraform/` (if any `.tf` files exist)

3. Check if any files that SHOULD be ignored are actually tracked:
   - Use `git ls-files` to see if `.env`, `.DS_Store`, or IDE config files are tracked

---

## Scanner 6: Git History Auditor

**Severity: HIGH**

Check git history for author information that reveals personal details or machine-specific identifiers.

### What to check

1. Run: `git log --format='%ae' | sort -u` to get unique author emails
2. Flag emails that contain:
   - Machine hostnames: `@.*MacBook`, `@.*\.local`, `@.*\.internal`
   - Generic local addresses: `@localhost`, `@127.0.0.1`
   - Personal domain emails that might not be intended for public (flag but don't auto-classify)
3. Run: `git log --format='%an' | sort -u` to get unique author names
4. Cross-reference with PII findings — flag if author names appear hardcoded elsewhere in the repo
5. Note: Git history is hard to rewrite. If issues are found, recommend `git filter-branch` or `git-filter-repo` as remediation, and warn about force-push implications.

---

## Report Generation

After all 6 scanners complete, aggregate findings into this format:

```markdown
# OSS Readiness Report

## Summary
- **Repository**: <name from git remote or directory name>
- **Files scanned**: <count from Glob **/*>
- **Critical**: <N> | **High**: <N> | **Medium**: <N> | **Low**: <N>
- **Verdict**: READY / NOT READY / NEEDS WORK

## Findings

### CRITICAL
<!-- Secrets found — must be removed before any public push -->
- [ ] `<file>:<line>` — <description>

### HIGH
<!-- PII, personal data, git history issues -->
- [ ] `<file>:<line>` — <description>

### MEDIUM
<!-- Hardcoded paths, missing license -->
- [ ] `<file>:<line>` — <description>

### LOW
<!-- Gitignore gaps, minor issues -->
- [ ] `<description>

## Remediation Steps
1. **CRITICAL**: Remove or redact all secrets immediately. Consider rotating any exposed credentials.
2. **HIGH**: Remove PII, fix git author emails. For git history: use `git filter-repo` to rewrite.
3. **MEDIUM**: Replace hardcoded paths with environment variables or relative paths. Add a LICENSE file.
4. **LOW**: Update .gitignore with recommended entries.

## Notes
- This scan checks source files only, not dependency vulnerabilities. Use `npm audit` / `cargo audit` for that.
- Git history issues require force-pushing after rewrite — coordinate with collaborators.
- Binary files were excluded from text-based scanning.
```

### Verdict Logic

- **READY**: 0 Critical, 0 High findings
- **NEEDS WORK**: 0 Critical, but has High/Medium findings
- **NOT READY**: Any Critical findings present

### Output Options

- Always display the report in the conversation
- If the user requests it (or if there are 10+ findings), also write to `OSS_READY_REPORT.md` in the repo root

---

## Self-Exclusion

When scanning, EXCLUDE these files from findings (they are part of the skill itself, not the target repo):
- This `SKILL.md` file
- Files in the `evals/` directory
- Files in `.loki/` directory
- Files in `.claude/` directory

---

## Edge Cases

- **No git history**: Skip Scanner 6, note in report
- **Binary-heavy repos**: Note that binary files were skipped, recommend manual review
- **Monorepos**: Scan the full repo but organize findings by directory
- **Empty repos**: Report "No files to scan" with READY verdict
- **Very large repos (1000+ files)**: Use Glob to get file list first, then scan in batches to avoid overwhelming context
