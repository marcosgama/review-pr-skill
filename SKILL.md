---
name: review-pr
description: Deep, repo-aware PR review for clever-datalake-workflows. Traces upstream sources, downstream consumers, cross-references schemas, and detects real bugs with code evidence. Use when reviewing PRs that touch dbt models, Glue jobs, Airflow DAGs, or pipeline DDL.
argument-hint: <PR-URL-or-number>
disable-model-invocation: true
allowed-tools: Bash(gh *) Read Grep Glob
---

# /review-pr — Deep Pipeline PR Review

You are reviewing a pull request in the **clever-datalake-workflows** repository. This is a data pipeline repo using dbt (Athena/Trino SQL), Airflow DAGs, AWS Glue ETL jobs, and a layered architecture (landing → raw → base → staging → intermediate → refined → mart).

The PR to review is: `$ARGUMENTS`

## Pre-Flight Checks

Before starting the review, validate the environment:

1. **Verify gh is authenticated**: Run `gh auth status`. If it fails, tell the user to run `gh auth login` and stop.
2. **Verify the PR exists**: Run `gh pr view $ARGUMENTS --json number,title,state`. If it fails, report that the PR URL or number is invalid and stop.
3. **Check for changes**: Run `gh pr view $ARGUMENTS --json files`. If the PR has zero changed files, report "No code changes to review" and stop.

## Step 1: Fetch and Understand the PR

Fetch PR metadata and read all changed files:

1. Run `gh pr view $ARGUMENTS --json number,title,body,author,commits,files,additions,deletions,baseRefName,headRefName` to get the full PR context.
2. Run `gh pr diff $ARGUMENTS` to get the complete diff.
3. For **every** changed file, read the full file (not just the diff). Use the Read tool on each file path. This is critical — diff hunks miss surrounding context.
4. Classify each file by pipeline layer using the rules in [pipeline-layers.md](pipeline-layers.md). Record the layer for each file.
5. Identify the change type for each file:
   - **Schema change**: Column additions, removals, renames, type changes
   - **New model**: File didn't exist before
   - **Bug fix**: Fixes logic errors, null handling, type casting
   - **Refactor**: Structural changes without behavior change
   - **Config change**: dbt project config, source definitions, test definitions

**For files classified as "Other"** (docs, CI, tooling): Review for general quality but skip pipeline-specific tracing steps below.

## Step 2: Trace Upstream

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

## Step 3: Trace Downstream

For each changed model, find every consumer:

1. Extract the model name from the file name (e.g., `staging_orders.sql` → `staging_orders`)
2. Grep the entire repo for `ref('model_name')` and `ref("model_name")`:
   ```
   grep -r "ref('staging_orders')" --include="*.sql" .
   grep -r 'ref("staging_orders")' --include="*.sql" .
   ```
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

## Step 4: Cross-Reference Consistency

Check these cross-cutting concerns:

### Snapshot check_cols
- Find any snapshots that reference changed models: grep for `ref('model_name')` in `**/snapshots/**/*.sql`
- Read the snapshot config and extract `check_cols`
- Verify every column in `check_cols` exists in the model's current output
- If the PR renames or removes a column that's in `check_cols`, flag it as HIGH severity

### schema.yml / properties.yml
- Find schema files for changed models: look for the model name in `**/*schema*.yml` or `**/*properties*.yml`
- Compare documented column names against the model's actual SELECT output
- Flag any columns documented but not in the model, or in the model but not documented
- Check if dbt tests reference columns that no longer exist

### Glue job ↔ base model
- If a Glue job changed, check the corresponding base model still matches
- If a base model changed, check the Glue job still extracts the right fields

### dbt test coverage
- For new columns: are tests defined? At minimum, `not_null` and `unique` for key columns
- For removed columns: are stale tests cleaned up?

### Airflow DAG dependencies
- If models were added or removed, check that DAG task dependencies reflect the data flow
- A model that `ref()`s another must have its Airflow task depend on that model's task

## Step 5: Find Bugs

Scan all changed files for the bug patterns defined in [bug-patterns.md](bug-patterns.md). For each pattern found:

1. Show the exact code at `file:line`
2. Explain why it's a problem
3. Provide a concrete suggested fix

Focus especially on:
- SQL function compatibility between Athena v2 and Trino
- NULL handling gaps (empty string → NULL before casting)
- Position-dependent INSERT INTO without column lists
- JSON extraction on polymorphic fields without type guards
- Schema mismatches between adjacent layers

## Step 6: Assess PR Description

Read the PR description (from Step 1 metadata). Evaluate whether it covers:

1. **What changed**: Which models/jobs/DAGs were modified and how
2. **Why**: The business or technical motivation
3. **Blast radius**: Which downstream models are affected
4. **Testing approach**: How the changes were validated

If the description is thorough, acknowledge it briefly. If it's missing critical context, suggest specific additions — write a draft paragraph the author could add.

## Step 7: Post as PR Comment (Optional)

After completing the review:

1. Ask the user: "Would you like me to post this review as a PR comment?"
2. If yes:
   - Write the review to a temporary file
   - Run `gh pr comment $ARGUMENTS --body-file /tmp/review-comment.md`
   - Report the comment URL
3. If no: The review stays in the terminal only.

## Output Format

Structure your review output as follows. Every section is required unless explicitly noted as skippable.

---

## Summary

**PR**: [title] ([URL])
**Author**: [author]
**Changes**: [N files changed], [+additions], [-deletions]
**Type**: [schema change / new model / bug fix / refactor / mixed]
**Layers touched**: [list of pipeline layers]

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
