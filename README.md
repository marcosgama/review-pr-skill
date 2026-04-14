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
9. Optionally posts the review as a PR comment

## Why not just use the PR diff?

Reviewing only the diff misses cross-file context. This skill reads the full repo to find:

- Downstream models that reference renamed/removed columns
- Schema mismatches between pipeline layers
- Snapshot `check_cols` that no longer match the source model
- Glue field extractions that drift from base model expectations
- Airflow DAG dependencies that don't match the data flow

## Installation

Copy the skill files into your target repo:

```bash
# From your data pipeline repo root
mkdir -p .claude/skills/review-pr
cp SKILL.md pipeline-layers.md bug-patterns.md .claude/skills/review-pr/
```

Or clone directly into your repo's skills directory:

```bash
cd your-repo/.claude/skills
git clone https://github.com/marcosgama/review-pr-skill.git review-pr
```

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- `gh` CLI installed and authenticated (`gh auth status`)
- Running from within a local clone of the target repo

## Usage

```
/review-pr https://github.com/org/repo/pull/280
/review-pr 280
```

## Customization

- **pipeline-layers.md** — Edit layer classification rules and glob patterns
- **bug-patterns.md** — Add or remove bug patterns to check for
- **SKILL.md** — Modify the review steps or output format

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — 7-step review orchestration and output template |
| `pipeline-layers.md` | Layer classification rules for 12 pipeline layer types |
| `bug-patterns.md` | 8 bug pattern checks with code examples and fixes |
