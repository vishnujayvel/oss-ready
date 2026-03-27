# oss-ready MVP: Generalized Open Source Readiness Skill

## Problem

The current oss-ready skill only does security scanning (secrets, PII, credentials, git history). While valuable, it's a one-trick pony. Developers need a comprehensive open source readiness tool that goes beyond security to cover community health, documentation quality, strategic guidance, and release readiness — all in one skill.

There's a clear market gap: no existing tool provides holistic, AI-driven open source management that measures both technical and sociological repository health.

## Goals

Expand oss-ready from a 1-feature security scanner into a 5-feature open source readiness suite:

1. **Security Scanner** (EXISTING — keep as-is) — secrets, PII, credentials, hardcoded paths, git history audit
2. **Repo Health Scorecard** (NEW) — scan for community health files, output a letter grade A+ to F with visual progress bars
3. **Open Source Coach** (NEW) — interactive session: "Should I open source this?" with viability, licensing, positioning guidance
4. **README Auditor** (NEW) — score existing README against best practices, identify missing sections, suggest improvements
5. **Release Readiness Check** (NEW) — pre-push gate: changelog, version tags, CI config, TODOs in code, clean git history

## Non-Goals

- NOT an auto-fixer (reports only, user decides what to fix)
- NOT a dependency vulnerability scanner (use npm audit for that)
- NOT a CI/CD integration or GitHub Action (this is a Claude Code skill)
- NOT using external CLI tools for visualization (all visuals rendered in Claude's text response)
- No MCP server dependencies for MVP
- No external tool installations required

## Architecture

This is a **pure prompt-based skill** — one SKILL.md file with no runtime dependencies. Claude uses its built-in tools (Grep, Glob, Read, Bash) for scanning and outputs results directly in its text response.

### Execution Model

The skill detects which mode the user wants based on their message:

- "oss-ready" / "full audit" → Run ALL 5 features, output combined report
- "repo health" / "scorecard" → Run only Repo Health Scorecard
- "should I open source" / "oss coach" → Run only Open Source Coach
- "readme audit" / "check my readme" → Run only README Auditor
- "release check" / "ready to ship" → Run only Release Readiness Check
- "security scan" / "secrets scan" → Run only Security Scanner (existing)

### Visualization Approach

ALL output rendered as Claude's own text response (not via Bash commands, which get collapsed in Claude Code). Visual toolkit:

- **Emoji as color**: 🟢🟡🔴 for status, ✅❌⚠️ for pass/fail/warn
- **Unicode block chars**: `████░░` for progress bars
- **ASCII box-drawing**: `══ ── ┌┐└┘` for structure
- **diff code blocks**: ```diff for green/red pass/fail lists
- **Open-right borders**: Left edge only (avoids emoji width mismatch on right border)
- **Markdown tables**: For structured data in findings

### Report Format (Full Audit)

```text
  ══════════════════════════════════════════
    OPEN SOURCE READINESS REPORT
    {repo-name}  |  {date}
  ══════════════════════════════════════════

    GRADE: 🏆 {letter-grade}    VERDICT: {READY|NEEDS WORK|NOT READY}

  ------------------------------------------

    🟢 Security      ████████░░  83%
    🟢 License       ██████████ 100%
    🟡 README        █████░░░░░  50%
    🔴 Community     ███░░░░░░░  33%
    🟢 Git Health    ██████████ 100%

  ------------------------------------------
    ... findings by severity ...
    ... remediation steps ...
  ══════════════════════════════════════════
```

## Scope: What Changes in SKILL.md

The existing SKILL.md has:
- Trigger conditions ✓
- 6 parallel scanners (PII, Secrets, Paths, License, Gitignore, Git History) ✓
- Report generation template ✓
- Verdict logic ✓

### ADDED — Feature 2: Repo Health Scorecard

Scan for GitHub community health files and score them:

| Check | Weight | What to scan |
|-------|--------|-------------|
| README.md exists and has content | 15 | Glob + Read for length, sections |
| LICENSE file exists and is recognized | 15 | Glob + Read, match known licenses |
| CONTRIBUTING.md exists | 10 | Glob |
| CODE_OF_CONDUCT.md exists | 10 | Glob |
| SECURITY.md exists | 10 | Glob |
| .gitignore exists and is comprehensive | 10 | Glob + Read |
| Issue templates exist | 5 | Glob for .github/ISSUE_TEMPLATE/ |
| PR template exists | 5 | Glob for .github/pull_request_template.md |
| CHANGELOG.md exists | 5 | Glob |
| FUNDING.yml exists | 5 | Glob for .github/FUNDING.yml |
| CI config exists | 5 | Glob for .github/workflows/ or similar |
| CODEOWNERS exists | 5 | Glob |

**Grading scale**: A+ (95-100), A (90-94), A- (85-89), B+ (80-84), B (75-79), B- (70-74), C+ (65-69), C (60-64), C- (55-59), D (50-54), F (<50)

**Output**: Visual scorecard with progress bars per category and overall letter grade.

### ADDED — Feature 3: Open Source Coach

Interactive conversational mode. Claude asks questions ONE AT A TIME:

1. **What does this project do?** — Understand the project
2. **Does something similar already exist?** — Search GitHub for similar repos (user confirms)
3. **What's your goal?** — Adoption? Community? Portfolio? Resume?
4. **Can you maintain it?** — Estimate effort based on repo size, issue velocity expectations
5. **Any IP/legal concerns?** — Check for proprietary dependencies, company code
6. **License recommendation** — Based on goals (permissive for adoption, copyleft for protection)
7. **Positioning advice** — Suggest README framing, where to announce (HN, Reddit, Twitter, dev.to)

**Output**: Summary verdict with recommendation and next steps.

### ADDED — Feature 4: README Auditor

Score the existing README against best practices:

| Section | Points | What to check |
|---------|--------|---------------|
| Project title + description | 10 | First line has clear name and one-line description |
| Badges | 5 | Has at least 1 badge (license, CI, version) |
| Installation instructions | 15 | Has install/setup section with code blocks |
| Usage examples | 15 | Has usage section with code examples |
| API documentation or link | 10 | Has API section or links to docs |
| Contributing section or link | 10 | Mentions how to contribute |
| License mention | 10 | Mentions which license |
| Table of contents | 5 | Has TOC if README > 100 lines |
| Screenshots/diagrams | 10 | Has images, GIFs, or diagrams |
| Contact/support info | 5 | Has support info or links |
| No broken links | 5 | Check for common broken link patterns |

**Output**: Section-by-section scoring with specific suggestions for missing/weak sections.

### ADDED — Feature 5: Release Readiness Check

Pre-push checklist:

| Check | Severity | How to scan |
|-------|----------|-------------|
| CHANGELOG.md exists and has entries | HIGH | Glob + Read |
| Latest version tag exists | MEDIUM | `git tag --sort=-v:refname` |
| No TODO/FIXME/HACK in code | LOW | Grep for TODO, FIXME, HACK patterns |
| No console.log/print debug statements | LOW | Grep for common debug patterns |
| Package version matches git tag | MEDIUM | Read package.json version vs latest tag |
| Build succeeds | HIGH | Detect build system, run build |
| Tests pass | HIGH | Detect test runner, run tests |
| No uncommitted changes | HIGH | `git status` |
| Branch is up to date with remote | MEDIUM | `git fetch && git status` |
| .env.example exists if .env is in .gitignore | LOW | Cross-reference .gitignore with file existence |

**Output**: Pass/fail checklist using diff code block format.

### MODIFIED — Trigger Conditions

Add new trigger phrases for each feature:

- Repo Health: "repo health", "scorecard", "community health", "health check"
- Coach: "should I open source", "oss coach", "open source coach", "is this worth open sourcing"
- README: "readme audit", "check my readme", "readme score", "readme review"
- Release: "release check", "release ready", "ready to ship", "pre-release", "ship check"
- Full: "oss-ready", "full audit", "open source readiness", (all existing triggers)

### MODIFIED — Report Generation

Update to include all 5 features in the combined report, with the visual scorecard format.

### MODIFIED — Evals

Add eval test cases for each new feature's trigger phrases (both positive and negative).

## Testing Strategy

1. Update `evals/evals.json` with trigger tests for all 5 features (positive + negative)
2. Test against the oss-ready repo itself as a known test target
3. Verify visual output renders correctly (no collapsed output since it's in Claude's text response)

## Technology

- Pure prompt-based skill (SKILL.md only)
- No runtime dependencies
- No external tools for visualization
- Uses Claude's built-in tools: Grep, Glob, Read, Bash (for git commands only)

## Success Criteria

- All 5 features work independently and combined
- Visual scorecard renders correctly in Claude Code
- Evals pass for all trigger phrases
- README.md updated with new features documentation

## Reference

- Deep research: `.claude/specs/oss-ready/deep-research-gemini.md` (128+ sources from Gemini)
- Background research: Agent findings on OpenSSF best practices, community health files, licensing
- Existing SKILL.md: Current 6-scanner security implementation
