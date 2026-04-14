# /review-pr — Claude Code Skill

Deep, repo-aware PR review skill for data pipeline repositories using dbt (Athena/Trino), Airflow DAGs, and AWS Glue.

## What it does

When invoked as `/review-pr <PR-URL>`, the skill:

1. Fetches the PR diff, description, and commits via `gh`
2. Reads every changed file in full (not just the diff)
3. Classifies files by pipeline layer (landing, raw, base, staging, intermediate, refined, mart)
4. Traces upstream sources and verifies field consistency
5. Traces downstream consumers and checks for breakage
6. Cross-references schemas, snapshots, tests, and configs
7. Detects common SQL/pipeline bug patterns
8. Outputs a structured review with severity-ordered findings
9. Saves the review to the Pulse inbox (if available) for downstream ingestion
10. Optionally posts the review as a PR comment

## Why not just use the PR diff?

Reviewing only the diff misses cross-file context. This skill reads the full repo to find:

- Downstream models that reference renamed/removed columns
- Schema mismatches between pipeline layers
- Snapshot `check_cols` that no longer match the source model
- Glue field extractions that drift from base model expectations
- Airflow DAG dependencies that don't match the data flow

## Installation

Claude Code discovers skills from a `SKILL.md` inside a named directory. Pick the scope that matches who should have the skill:

| Scope | Path | Applies to |
|-------|------|------------|
| Project | `<repo>/.claude/skills/review-pr/` | Everyone who clones the repo (commit it) |
| Personal | `~/.claude/skills/review-pr/` | You, across all your projects |

Install as a **project** skill (recommended — the skill is repo-specific):

```bash
# From clever-datalake-workflows root
mkdir -p .claude/skills/review-pr
cp /path/to/review-pr-skill/{SKILL.md,pipeline-layers.md,bug-patterns.md} .claude/skills/review-pr/
```

Or clone directly into the skills directory:

```bash
cd clever-datalake-workflows/.claude/skills
git clone https://github.com/marcosgama/review-pr-skill.git review-pr
```

Claude Code live-reloads skill changes, so edits to these files take effect in the current session.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- `gh` CLI installed and authenticated (`gh auth status`)
- Running from within a local clone of `clever-datalake-workflows`

## Usage

```
/review-pr https://github.com/org/repo/pull/280
/review-pr 280
```

## Pulse integration (loose coupling)

Reviews are written to a Pulse inbox directory as self-contained markdown files. Pulse ingests these on its own schedule and updates its internal data structures — the skill does not call Pulse directly.

**Target directory**, in priority order:

1. `$PULSE_INBOX` environment variable, if set
2. `~/.pulse/data/inbox/` if it exists
3. Otherwise the step is skipped silently

**File naming**: `pr-<number>-<UTC-timestamp>.md` (timestamps prevent collisions on re-review).

**File shape**: YAML frontmatter with `pr_url`, `pr_number`, `pr_title`, `pr_author`, `base_ref`, `head_ref`, `source_repo`, `reviewed_at`, `reviewer`, followed by the full rendered review body. Pulse parses this however it wants — the markdown is the interface.

To enable:

```bash
# Option A: use the default Pulse data dir
mkdir -p ~/.pulse/data/inbox

# Option B: point it somewhere else
export PULSE_INBOX=/path/to/pulse/inbox
```

## Customization

- **pipeline-layers.md** — Edit layer classification rules and glob patterns
- **bug-patterns.md** — Add or remove bug patterns to check for
- **SKILL.md** — Modify the review steps or output format

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — 10-step review orchestration, Pulse handoff, and output template |
| `pipeline-layers.md` | Layer classification rules for 12 pipeline layer types |
| `bug-patterns.md` | 8 bug pattern checks with code examples and fixes |
