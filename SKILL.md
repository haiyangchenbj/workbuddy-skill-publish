---
name: skill-publish
description: >
  Audit and publish agent skills to ClawHub and GitHub. Scans for personal data,
  validates frontmatter, enforces bilingual docs, removes internal-only content,
  and pushes clean distributions. Trigger keywords: publish skill, audit skill,
  发布skill, 检查发布, skill发布, push to clawhub.
version: 1.0.2
read_when:
  - publish skill
  - audit skill
  - 发布skill
  - 检查发布
  - skill发布
  - push to clawhub
  - sync to clawhub
  - clean publish
disable: true
---

# Skill Publish

> Audit, clean, and publish agent skills to ClawHub and GitHub.
> Complements `skill-design-guide` (design-time) with publish-time rules. Works with any SKILL.md-based skill in the OpenClaw ecosystem.

## Overview

Local skill directories often contain personal scripts, internal notes, and ticker-specific code that should never appear in public distributions. This skill standardizes the audit → cleanup → publish → verify pipeline.

**Two modes**:
- `audit` — **read-only**. Scans and reports; touches nothing, transmits nothing.
- `publish` — **has external side effects**. It transmits the (cleaned) skill contents to
  ClawHub and GitHub, which are **public services**, and may **delete** files in the remote
  GitHub repo. It never edits the user's local files. Publish only runs after an audit and
  explicit user confirmation of the file list, target repos, and version.

> **Before publishing, understand:** anything in the whitelisted file set leaves your
> machine and becomes publicly visible. Personal data, credentials, and internal notes
> must be resolved in the audit first — this skill blocks on them but you are the last
> line of review.

## Workflow

### Mode: Audit (default when user says "check" or "audit")

### Step 1: [Deterministic] Confirm Target

- Identify the skill directory to audit. If user didn't specify, ask.
- Verify SKILL.md exists at `<dir>/SKILL.md`.
- If missing, stop: "Not a skill directory — no SKILL.md found."

### Step 2: [Deterministic] Load Rules

- Read `references/publish-rules.md` for detailed validation criteria.

### Step 3: [Deterministic] Scan Files

For each file in the skill directory (recursive), check:

| Scan | What To Flag |
|------|-------------|
| **Personal data** | Ticker symbols with share counts, cost basis, account values, personal names/emails |
| **Ticker-specific code** | Scripts or configs hardcoded to single tickers (e.g. `check_tsla_entry.py`) |
| **Internal dev notes** | "审计清理版", "next N months: no new rules", personal reminders |
| **Meta-documents** | Files like `SKILL_PUBLISH_RULES.md` that aren't part of the skill itself |
| **Old version files** | `Q2_2026_交易策略.md` — outdated strategy docs |

### Step 4: [LLM] Validate SKILL.md Content

Read the full SKILL.md. Check:

| Check | Rule |
|-------|------|
| Frontmatter `name` | No version number embedded |
| Frontmatter `description` | ≤3 sentences, English, no tickers/versions/keywords |
| H1 title | No version number |
| `## 详细参考` / `## References` section | Must NOT exist |
| Version history | Only published versions, max 5 rows, no "planned" entries |
| Internal dev notes | Must be absent from body text |
| Language | ClawHub's default is an English body (with an optional Chinese summary at the end) for discoverability. This is a **recommended platform convention, not a hard requirement** — if the skill's audience or the user intends another primary language, keep it and just flag the deviation in the report. Never delete or reject valid content solely because of its language. |

### Step 5: [Deterministic] Validate Bilingual Docs

- `README.md` exists and is in English
- `README_zh.md` exists and is in Chinese
- Version numbers in both README files match
- `CONTRIBUTING.md` exists and is in English (if present)

### Step 6: [Deterministic] Generate Audit Report

Output format:

```
## Skill Publish Audit: <skill-name>

### Issues Found

| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | HIGH | SKILL.md | description 含版本号和关键词列表 |
| 2 | HIGH | SKILL.md | H1 标题带版本号 |
| 3 | MEDIUM | check_tsla_entry.py | 个股脚本，不应发布 |
| 4 | LOW | SKILL.md | 版本历史含未发布版本 |

### Clean Files (safe to publish)
- SKILL.md (after fixes above)
- README.md ✓
- README_zh.md ✓
- references/*.md (3 files) ✓
- scripts/*.py (8 files) ✓

### Action Required Before Publish
- [ ] Fix issue #1: rewrite description
- [ ] Fix issue #2: remove version from H1
- [ ] Fix issue #3: exclude check_tsla_entry.py
- [ ] Fix issue #4: trim version history
```

Stop here in audit mode. User reviews and decides next step.

### Mode: Publish (when user says "publish" or confirms audit)

> **⚠️ Publish mode performs external and irreversible actions.** It transmits skill
> contents to ClawHub and GitHub (public services) and may delete remote files. Before
> running any of the steps below, the agent MUST (a) present the audit report, (b) show
> the exact file list to be transmitted and the target repos, and (c) obtain explicit
> user confirmation. Never enter publish mode automatically from an audit request.

### Step 7: [LLM] Fix Content Issues — in the publish copy only

Apply all fixes identified in the audit:
- Rewrite frontmatter description (≤3 English sentences)
- Strip version from H1 title
- Remove `## 详细参考/References` section
- Trim version history to 5 published versions max
- Remove internal dev notes throughout

**Do NOT edit the files in the user's local skill directory.** First copy the source
files into the temp publish directory (Step 8), then write all fixed content ONLY to the
copies inside that temp dir (`<slug>/SKILL.md`, `<slug>/README.md`, `<slug>/README_zh.md`).
The user's working copy stays byte-for-byte unchanged. This preserves the "local stays
intact" guarantee in Hard Rule #1 and lets the user review the source before publishing.

### Step 8: [Deterministic] Prepare Clean Directory

> Do all content edits from Step 7 **inside this temp dir**, never on the source files.

1. Create temp dir named **exactly `<slug>`** (e.g. `skill-design-guide-skill`), NOT an arbitrary name like `sdg-publish-142`. When `--name` is omitted, ClawHub derives the display name from the temp dir name (so a junk dir name becomes a junk display name like "Sdg Publish 142").
2. Copy only whitelisted files:
   - `SKILL.md`, `README.md`, `README_zh.md`
   - `CONTRIBUTING.md`, `LICENSE` (if exist)
   - `references/` (all `.md` files)
   - `scripts/` (all `.py` files — filters out personal scripts)
   - `requirements.txt`, `_meta.json`
3. Verify: NO ticker-specific scripts, NO personal files, NO meta-documents in temp dir.
4. **EXCLUDE `skill-card.md`** — auto-generated by ClawHub; publishing it errors with "skill-card.md is generated by ClawHub and cannot be published directly".
5. **EXCLUDE dot-prefixed dirs** (`.local/`, `.clawhub/`, `.git/`) — never copy them into the temp dir.

### Step 9: [Deterministic] Determine Version

- Read `_meta.json` for current published version
- If user specified a version, validate it's higher than current
- If no version specified, ask user for target version
- Generate changelog summary from audit fixes

### Step 10: [Deterministic] Push to ClawHub

```bash
clawhub publish <clean-dir> --slug <slug> --name "<Display Name>" --version <version> --changelog "<summary>"
```

- **Always pass `--name`** to preserve the display name (omitting it makes ClawHub guess from the dir name).
- If "Version already exists": bump patch version and retry
- If rate-limited (e.g. "remaining: N/3000, reset in Ns"): wait the stated seconds, then retry
- If network error: report and stop
- On success: capture new version and publish ID

### Step 11: [Deterministic] Push to GitHub

For each whitelisted file:
1. GET `https://api.github.com/repos/<owner>/<repo>/contents/<path>` → get SHA
2. PUT with base64 content + SHA (or POST if new file)

For files to delete (from audit list):
- DELETE with SHA

**Credentials.** Read the GitHub PAT from the connector token file managed by WorkBuddy:
`~/.workbuddy/connectors/<id>/tokens/github.txt`. Use it **only** to authenticate the
repo writes/deletes for this publish, and only against the single target repo the user
confirmed. Do not read, copy, log, echo, or transmit the token value anywhere else, and
do not persist it outside the request. If the file is absent, stop and ask the user to
enable the GitHub connector — never prompt for or hardcode a raw token.

### Step 12: [Deterministic] Verify

```
[ ] clawhub inspect <slug> → latest version matches, English description
[ ] GitHub repo: correct files present, no personal scripts
[ ] Clean temp dir deleted
```

## Hard Rules

> Safety guarantees for this workflow.

1. **Never modify the user's local files.** Copy sources into a temp publish dir first; apply every fix only to the copies there. The local working copy stays byte-for-byte unchanged.
2. **Never publish without audit + explicit confirmation.** Every file must be scanned, and the user must approve the exact file list, target repos, and version before any external transmission or remote deletion.
3. **Personal data is a blocking error.** Any finding of account values, share counts, cost basis → must be resolved before publish.
4. **No version in name.** `name` field and H1 title must never contain version numbers.
5. **English body is the platform default, not a mandate.** ClawHub favors an English SKILL.md body for discoverability; recommend it and flag deviations, but respect the user's intended language and never rewrite or reject valid content on language grounds alone.
6. **Temp dir must be deleted** after successful publish.

## Failure Handling

| Scenario | Action |
|----------|--------|
| SKILL.md not found | Stop: "Not a skill directory" |
| Personal data found | Block publish; report exact file+line; user must manually remove |
| ClawHub auth missing | Guide: "Run `clawhub login --token <token>`" |
| Version already exists | Auto-bump patch version; ask user to confirm |
| "skill-card.md ... cannot be published" | Remove `skill-card.md` from temp dir; retry |
| Rate limited ("reset in Ns") | Wait N seconds, then retry the same publish |
| Display name shows junk (e.g. "Sdg Publish 142") | Temp dir was misnamed; rename to `<slug>` and re-publish with `--name` |
| GitHub PAT not found | Guide: "Ensure GitHub connector is active" |
| GitHub write fails (403) | Report: "Token lacks write permission on this repo" |
| Network timeout | Retry once; if still fails, report and stop |
| User cancels mid-publish | Clean up temp dir, report: "Publish aborted, local files untouched" |

## Output Format

### Audit Mode Output
```
## Skill Publish Audit: <name> v<version>

### Issues (N found)
[table: severity, file, issue, fix]

### Clean Files (M files, K KB)
[file list]

### Publish Target
- ClawHub: <owner>/<slug>
- GitHub: <owner>/<repo>
- Version: <current> → <target>
```

### Publish Mode Output
```
## Publish Complete: <name>@<version>

### ClawHub
- URL: https://clawhub.ai/<owner>/<slug>
- Version: <version>
- Publish ID: <id>

### GitHub
- Repo: <owner>/<repo>
- Commit: <sha>
- Files: <N> updated, <M> deleted
```
