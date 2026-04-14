---
name: review-pr
description: Deep, repo-aware PR review for clever-datalake-workflows data pipelines. Traces upstream sources, downstream consumers, cross-references schemas, and detects bugs with code evidence. Use when reviewing PRs that touch dbt models, Glue jobs, Airflow DAGs, or pipeline DDL.
when_to_use: Trigger when the user asks to review, audit, or check a PR in clever-datalake-workflows, or pastes a PR URL/number for review. Example phrases "review PR 280", "check this PR", "look at pull request <url>".
argument-hint: <PR-URL-or-number>
disable-model-invocation: true
allowed-tools: Bash(gh *) Read Write Grep Glob
---

# /review-pr — Deep Pipeline PR Review

You are reviewing a pull request in the **clever-datalake-workflows** repository: a data pipeline repo using dbt (Athena/Trino SQL), Airflow DAGs, AWS Glue ETL jobs, and a layered architecture (landing → raw → base → staging → intermediate → refined → mart).

The PR to review is: `$ARGUMENTS`

## Reference files

Load these only when you reach the step that needs them — they are one level deep from this file.

- [pipeline-layers.md](pipeline-layers.md) — classification rules for each file (Step 3)
- [bug-patterns.md](bug-patterns.md) — bug pattern catalog to scan for (Step 7)

## Review checklist

Copy this checklist into your response before starting Step 1, and check off each item as you finish it. This keeps the review sequenced and nothing gets skipped.

```
Review Progress:
- [ ] Pre-flight: gh auth OK, PR exists, has changed files
- [ ] Step 1: Understand the PR — theme, intent, developer's approach
- [ ] Step 2: Load repo conventions (root + nested CLAUDE.md, CONTRIBUTING.md)
- [ ] Step 3: Read every changed file in full, classify by layer, identify change type
- [ ] Step 4: Trace upstream — verify column consistency against sources
- [ ] Step 5: Trace downstream — enumerate every ref() consumer
- [ ] Step 6: Cross-reference snapshots, schema.yml, Glue, tests, DAGs
- [ ] Step 7: Quality scan — style/conventions, Glue complexity, Airflow practices, bug patterns
- [ ] Step 8: Assess PR description completeness vs what you found
- [ ] Output: Render the review in the format below
- [ ] Step 9: Save the review to the Pulse inbox (if available)
- [ ] Step 10: Ask the user whether to post the review as a PR comment
```

## Pre-Flight Checks

Validate the environment before doing anything else. If any check fails, stop and report the failure.

1. `gh auth status` — if it fails, tell the user to run `gh auth login`.
2. `gh pr view $ARGUMENTS --json number,title,state` — if it fails, the PR URL/number is invalid.
3. `gh pr view $ARGUMENTS --json files` — if zero changed files, report "No code changes to review".

## Step 1: Understand the PR

**Start here, before touching any code.** A PR is almost always about a single theme (e.g. "Hubspot — add contact sync", "fix orders null-handling"). Grasping the theme and how the developer approached it shapes every downstream finding.

1. Run `gh pr view $ARGUMENTS --json number,title,body,author,commits,files,additions,deletions,baseRefName,headRefName`.
2. Read the **title** and **description** carefully. Also skim the commit messages — the commit history often reveals the thought process the description omits.
3. Write down, in your own words:
   - **Theme** — one sentence. "This PR is about X."
   - **Intent** — what outcome the author is after (new capability, fix, cleanup, migration, etc.).
   - **Approach** — the strategy you can infer from the title/body/commits (e.g. "add a new staging model and wire it into the existing hubspot mart").
   - **Open questions** — anything the description leaves ambiguous that you'll need to answer by reading the code.
4. Keep this framing visible through the rest of the review — every finding should be evaluated against whether it helps or hurts the stated intent.

If the description is empty or misleading, note it now; Step 8 will give concrete suggestions.

## Step 2: Load Repo Conventions

The repo's own conventions are the primary review rubric. Load them **before** critiquing style or patterns.

1. Read the root `CLAUDE.md` if it exists.
2. Read `CONTRIBUTING.md` if it exists.
3. For each directory touched by the PR, check for a nested `CLAUDE.md` (e.g. `models/staging/CLAUDE.md`, `dags/CLAUDE.md`). Use Glob: `**/CLAUDE.md`.
4. Record the conventions that apply to the files in this PR. Treat any departure from them as a first-class finding in Step 7, and cite the source (`CLAUDE.md:line` or the sibling file that establishes the precedent).

If none of these files exist, fall back to the nearest existing sibling of each changed file as the implicit style reference.

## Step 3: Read and Classify Changed Files

1. Run `gh pr diff $ARGUMENTS` for the complete diff.
2. For **every** changed file, read the full file with the Read tool. Diff hunks miss surrounding context.
3. Classify each file using the rules in [pipeline-layers.md](pipeline-layers.md). Record the layer per file.
4. Identify the change type for each file:
   - **Schema change** — column additions, removals, renames, type changes
   - **New model** — file didn't exist before
   - **Bug fix** — fixes logic errors, null handling, type casting
   - **Refactor** — structural changes without behavior change
   - **Config change** — dbt project config, source definitions, test definitions

**For files classified as "Other"** (docs, CI, tooling): review for general quality but skip pipeline-specific tracing steps below.

## Step 4: Trace Upstream

For each changed dbt model (base, staging, intermediate, refined, mart):

1. Find its sources:
   - Look for `{{ source('schema', 'table') }}` calls → read the corresponding `sources.yml` to find the actual table
   - Look for `{{ ref('model_name') }}` calls → read the referenced model
2. For base models specifically:
   - Find the corresponding Glue job (grep for the table name in `**/glue/**/*.py`)
   - Read the Glue job to see what fields it extracts
   - Find the landing DDL (grep for the table name in `**/landing/**/*.sql`)
3. **Verify field consistency**: Do the column names the model expects match what the upstream source actually provides? Check:
   - Column names match exactly (case-sensitive)
   - Data types are compatible
   - No references to columns that don't exist upstream

## Step 5: Trace Downstream

For each changed model, find every consumer:

1. Extract the model name from the file name (e.g., `staging_orders.sql` → `staging_orders`).
2. Use the Grep tool to find every consumer. Search both quote styles across `*.sql`:
   - pattern `ref\(['"]staging_orders['"]\)` with `glob: "**/*.sql"`
3. Read each downstream model in full.
4. For each downstream consumer, check:
   - Does it reference any columns that were **removed** or **renamed** in this PR?
   - Does it use `SELECT *` from the changed model? If so, flag it — schema changes propagate silently through `SELECT *`.
   - Does it join on columns that changed type?
5. **Report the blast radius**: For every affected downstream model, record:
   - File path
   - The specific `ref()` call connecting it to the changed model
   - Which columns are at risk
   - Whether it will break (hard failure) or silently produce wrong data (soft failure)

## Step 6: Cross-Reference Consistency

Check these cross-cutting concerns:

### Snapshot check_cols
- Use Grep to find snapshots that reference changed models (`ref\(['"]model_name['"]\)` scoped to `**/snapshots/**/*.sql`).
- Read the snapshot config and extract `check_cols`.
- Verify every column in `check_cols` exists in the model's current output.
- If the PR renames or removes a column that's in `check_cols`, flag it as HIGH severity.

### schema.yml / properties.yml
- Use Glob to find schema files for changed models (`**/*schema*.yml`, `**/*properties*.yml`).
- Compare documented column names against the model's actual SELECT output.
- Flag any columns documented but not in the model, or in the model but not documented.
- Check if dbt tests reference columns that no longer exist.

### Glue job ↔ base model
- If a Glue job changed, check the corresponding base model still matches
- If a base model changed, check the Glue job still extracts the right fields

### dbt test coverage
- For new columns: are tests defined? At minimum, `not_null` and `unique` for key columns
- For removed columns: are stale tests cleaned up?

### Airflow DAG dependencies
- If models were added or removed, check that DAG task dependencies reflect the data flow
- A model that `ref()`s another must have its Airflow task depend on that model's task

## Step 7: Quality Scan

Scan every changed file against [bug-patterns.md](bug-patterns.md). For each finding: exact `file:line`, the code, why it's a problem, a concrete fix.

**Lead with repo-specific concerns, in this order**:

1. **Style & convention adherence** (bug-patterns §1) — apply the guides you loaded in Step 2. Cite the source for every violation.
2. **Glue jobs — happy-path, on-point complexity** (bug-patterns §2). Flag over-engineering (defensive scaffolding, speculative abstractions, retry/caching layers) *and* under-specification (missing fields, skipped coercions). Compare against the nearest sibling Glue job.
3. **Airflow — follow Airflow best practices** (bug-patterns §3). No hardcoding of dates, credentials, paths, or env-specific values — use Connections, Variables, templated fields. Match DAG patterns already in the repo.
4. **Pipeline correctness** (bug-patterns §§4–10) — NULL handling, position-dependent INSERTs, JSON type guards, schema mismatches, snapshot check_cols, DAG dependency wiring, Glue↔base field drift.
5. **Athena v2 vs Trino syntax** — only when the PR is explicitly about engine portability. Not a default focus (see bug-patterns appendix).

## Step 8: Assess PR Description

Now that you know what the PR actually does, compare the description (from Step 1) against the reality you uncovered. Evaluate whether the description covers:

1. **Theme & intent** — does the description state the theme clearly? Does it match what you found?
2. **What changed** — which models/jobs/DAGs were modified and how.
3. **Why** — the business or technical motivation.
4. **Blast radius** — which downstream models are affected (cross-check against Step 5).
5. **Testing approach** — how the changes were validated.

If the description is thorough, acknowledge it briefly. If it's missing critical context, suggest specific additions — write a draft paragraph the author could paste in.

## Step 9: Save Review to Pulse Inbox

Write the rendered review to the Pulse inbox as a self-contained markdown file. Pulse ingests this on its own schedule and updates its internal structures; this skill does not talk to Pulse directly. Keep the coupling loose — the file is the interface.

1. **Determine the target directory**, in priority order:
   - `$PULSE_INBOX` environment variable, if set (check with `[ -n "$PULSE_INBOX" ]`).
   - `../oneone/data/inbox/` relative to the current repo, if that directory exists (`[ -d "../oneone/data/inbox" ]`).
   - Otherwise, **skip this step silently** — no Pulse available, no harm done.
2. **Derive the PR number** from `gh pr view $ARGUMENTS --json number -q .number`.
3. **Write the file** to `<target-dir>/pr-<number>-<UTC-timestamp>.md` (timestamp format `YYYYMMDDTHHMMSSZ`). Use the Write tool.
4. The file must begin with YAML frontmatter so Pulse's ingester has structured metadata without re-parsing the PR:

   ```yaml
   ---
   pr_url: <full URL from gh pr view>
   pr_number: <number>
   pr_title: <title>
   pr_author: <author login>
   base_ref: <baseRefName>
   head_ref: <headRefName>
   source_repo: <owner/repo from the URL>
   reviewed_at: <ISO 8601 UTC timestamp, e.g. 2026-04-14T11:30:00Z>
   reviewer: claude-code/review-pr
   ---
   ```

5. After the frontmatter, paste the **full review body** you rendered (everything from "Summary" through "Coaching" in the Output Format section).
6. Print a one-line confirmation: `Saved review to <full path>`.

Do not overwrite an existing file with the same name — the timestamp prevents collisions. If the user re-reviews the same PR, each review becomes its own file; Pulse can decide how to reconcile them.

## Step 10: Post as PR Comment (Optional)

After rendering the review in the terminal and (if applicable) saving to the Pulse inbox:

1. Ask the user: "Post this review as a PR comment?"
2. If yes:
   - Write the review to `/tmp/review-comment-$ARGUMENTS.md` (session-scoped path).
   - Run `gh pr comment $ARGUMENTS --body-file /tmp/review-comment-$ARGUMENTS.md`.
   - Report the returned comment URL.
3. If no: stop — the review stays in the terminal only.

Never post without explicit user confirmation — PR comments are visible to the whole team.

## Output Format

Structure your review output as follows. Every section is required unless explicitly noted as skippable.

---

## Summary

**PR**: [title] ([URL])
**Author**: [author]
**Changes**: [N files changed], [+additions], [-deletions]
**Type**: [schema change / new model / bug fix / refactor / mixed]
**Layers touched**: [list of pipeline layers]

## Theme & Intent

- **Theme**: [one-sentence description of what this PR is about]
- **Intent**: [the outcome the author is after]
- **Approach**: [the strategy the author took, as inferred from title/body/commits]

## Approach

- [What you analyzed]
- [Which tracing steps you performed]
- [How many downstream models you checked]

## Blast Radius

| Downstream Model | File | Connection | Risk | Impact |
|-----------------|------|------------|------|--------|
| [model_name] | [path] | `ref('changed_model')` at line N | Column `X` renamed | Hard failure — query will error |
| [model_name] | [path] | `SELECT *` from changed model | Schema change propagates | Soft failure — wrong column order |

If no downstream impact: state "No downstream consumers found" with evidence (grep results).

## Findings

Order by severity. For each finding:

### [SEVERITY] [Title]

**Location**: `file:line`

**Code**:
```sql
-- The problematic code
```

**Impact**: [What breaks or degrades]

**Fix**:
```sql
-- Suggested fix
```

If no findings: state "No issues found" — this is a valid outcome for clean PRs.

## PR Description

- **Covers well**: [what the description does well]
- **Missing**: [what should be added]
- **Suggested addition**: [draft text if needed]

Skip this section if the description is thorough.

## Coaching

- [1-3 actionable patterns to adopt or avoid for future PRs]
- [Repo-specific best practices relevant to this change]

Keep coaching concise and specific to what you observed — no generic advice.
