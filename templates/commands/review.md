---
description: Review the implemented code against spec.md, plan.md, tasks.md, and constitution.md. Writes review.md with HIGH-SIGNAL findings only. Pass `--comment` (and optionally `--pr <N>`) to post inline GitHub PR comments.
handoffs:
  - label: Fix Review Findings
    agent: speckit.implement
    prompt: Address review findings in review.md (start with CRITICAL/HIGH)
    send: true
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

Recognized flags inside `$ARGUMENTS`:

- `--pr <N>` — review PR number `N` instead of local diff
- `--comment` — post review findings to the PR as inline comments (requires `--pr` or a current branch with an open PR)
- Any other free-form text → use as scope hints (e.g., a diff range like `HEAD~5..HEAD`, or a focus area like `security only`)

## Pre-Execution Checks

**Check for extension hooks (before review)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_review` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Goal.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Goal

Produce a HIGH-SIGNAL code review of the implemented code against the feature's `spec.md`, `plan.md`, `tasks.md`, and `/memory/constitution.md`. The review writes findings to `review.md` inside the feature directory and may optionally post inline comments on a GitHub PR.

This command MUST run only after `__SPECKIT_COMMAND_IMPLEMENT__` has produced code changes (committed, staged, or working-tree).

## Operating Constraints

- **Source code is READ-ONLY**: this command writes only `review.md` (and optional GitHub comments via `--comment`). It does not modify source files.
- **Constitution is non-negotiable**: violations of MUST principles in `/memory/constitution.md` are automatically CRITICAL.
- **HIGH-SIGNAL ONLY**: false positives erode trust. When in doubt, drop the finding.
- **No exploratory tool calls**: every tool invocation must serve a concrete review step. Do not "test" tools.

## Execution Steps

### 1. Initialize Review Context

Run `{SCRIPT}` once from repo root and parse JSON for FEATURE_DIR and AVAILABLE_DOCS. Derive absolute paths:

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md
- REVIEW = FEATURE_DIR/review.md
- CONSTITUTION = `/memory/constitution.md` (may be absent)

Abort if SPEC, PLAN, or TASKS is missing (instruct user to run the missing prerequisite command).

For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

### 2. Determine Review Scope

Resolution order:

1. If `--pr <N>` in `$ARGUMENTS`: fetch the PR with `gh pr view <N> --json number,title,body,state,isDraft,headRefOid,baseRefName,headRefName,url`. Use `gh pr diff <N>` for the diff. Capture `headRefOid` as `FULL_SHA` for inline-comment links.
2. Else if user provided an explicit ref/range (e.g., `HEAD~5..HEAD`, branch name, commit SHA), use it. `FULL_SHA = git rev-parse HEAD`.
3. Else if a feature branch is detected (from `.specify/feature.json` or current branch matching the feature dir pattern), diff against merge-base with the default branch (`main`/`master`). `FULL_SHA = git rev-parse HEAD`.
4. Else, diff working-tree + staged changes against `HEAD`. `FULL_SHA = git rev-parse HEAD`.

Capture:

- **DIFF**: full unified diff
- **CHANGED_FILES**: paths grouped by area (src, tests, config, docs)
- **NEW_FILES** / **MODIFIED_FILES** / **DELETED_FILES**
- **PR_TITLE** and **PR_BODY** (if `--pr`); otherwise commit subject of HEAD

### 3. Pre-flight Gate (skip conditions)

Evaluate in order. If any fires, exit cleanly without writing `review.md` (other than as noted) and report the reason to the user.

1. **Empty diff** — write one-line `review.md`: "No changes to review at <scope>." Exit.
2. **PR closed/merged** (only when `--pr`) — `state` is `CLOSED` or `MERGED`. Skip.
3. **Draft / WIP** (only when `--pr`) — `isDraft` is true, or title contains `[WIP]` / `[DRAFT]`. Skip.
4. **Already reviewed** — if `review.md` exists and its `**Scope**` matches the current scope/SHA, ask the user before overwriting (default: skip).
5. **Already commented by speckit** (only when `--comment` and `--pr`) — `gh pr view <N> --comments --json comments` shows a prior `Code review` summary from this command at the same `headRefOid`. Skip the comment step (still allow local re-review if user confirms).
6. **Trivial / generated-only diff** — diff is exclusively in:
   - lockfiles: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `Gemfile.lock`, `composer.lock`, `go.sum`
   - files marked `linguist-generated` in `.gitattributes`
   - vendor directories: `vendor/`, `third_party/`, `node_modules/`
   Skip with message "No reviewable source changes."

### 4. Load Artifacts (Progressive Disclosure)

Load only what each pass needs:

**From spec.md**: Functional Requirements (FR-###), Success Criteria (SC-###), User Stories with acceptance scenarios, Edge Cases.
**From plan.md**: stack/deps, Technical Constraints, Project Structure, contract references.
**From tasks.md**: completed task IDs, descriptions, file paths.
**From constitution**: principle names, MUST/SHOULD statements, scoping notes.
**From code**: read changed files in full when small; for large files, load the diff hunks plus enough surrounding context to understand reachability.

### 5. Review Passes — SIGNAL DISCIPLINE

**Read this before flagging anything.**

Only flag a finding when at least one of the following is true:

- The code will fail to compile, parse, or type-check (syntax error, missing import, undefined symbol, type error)
- The code will definitely produce wrong results regardless of input — a clear logic error provable from the diff alone (off-by-one with no guard, inverted condition, wrong operator, swapped arguments)
- A constitution **MUST** principle is unambiguously violated, AND you can quote the exact rule + cite the violating line
- A `spec.md` FR-### or acceptance scenario is materially unmet by the implementation
- A clear security flaw exists in the changed code (injection, missing auth/authz check, hardcoded secret, unsafe deserialization, path traversal, SSRF, open redirect, broken crypto)

**Do NOT flag:**

- Code style, formatting, or naming preferences
- "Could be more readable" / "consider refactoring" / general code-quality concerns
- Potential issues conditional on inputs or state outside the diff
- Pre-existing issues not touched by this change
- Issues a linter would catch (do **not** run linters to verify)
- Missing test coverage **unless** required by the constitution or by an unmet spec acceptance scenario
- Speculative future-proofing
- Concerns that require context outside the diff that you cannot validate
- Issues already silenced explicitly (e.g., a permitted lint-ignore comment per constitution)

**If you are not certain a finding is real, drop it.**

Now run the passes. Cite `path/to/file.ext:LN` and quote the offending line.

#### A. Requirements Coverage

- Each FR-### claimed implemented (per `tasks.md` mark-offs): is it actually realized in code?
- Flag FR-### with no observable supporting code change.
- SC-### that need buildable evidence (perf budgets, scale tests) but lack it.

#### B. Plan Adherence

- Stack/dependencies match plan's Technical Context (no unsanctioned libs).
- File layout follows the chosen Project Structure.
- Public contracts in `contracts/` honored (signatures, schemas, status codes).

#### C. Constitution Compliance

- For each MUST/SHOULD principle: pass / fail with exact quote of the rule and citation of the violating line.
- Scoping: a principle scoped to certain paths only applies to files matching that scope. Do not flag files outside scope.
- TDD principle (if present): tests exist for new behavior; failing-first evidence is discoverable from history (test commit precedes code commit, or commit message records RED→GREEN). If test + code share a single squashed commit with no evidence of failure, flag MEDIUM (not CRITICAL) — best-effort only.

#### D. Correctness & Bugs

- Off-by-one, null/undefined deref, wrong operator (`<` vs `<=`), inverted condition, swapped argument order
- Error paths swallowing exceptions, leaking resources, returning success on failure
- Concurrency: race, shared mutable state, missing await/lock
- Edge cases from spec: handled?

#### E. Security

- Input validation at boundaries (user input, external APIs)
- Injection (SQL, command, XSS, path traversal)
- AuthN/AuthZ check present where required
- Secrets/credentials not committed
- Unsafe deserialization, SSRF, open redirect, weak crypto

#### F. Tests (only if required by constitution or spec)

- New behavior has tests; tests assert behavior, not implementation
- Acceptance scenarios from spec covered
- Mocked-out boundaries that hide real failures

#### G. Severity Assignment

- **CRITICAL**: constitution MUST violation; missing FR; security flaw; data-loss risk; broken core flow
- **HIGH**: definite bug; spec acceptance scenario unmet; significant perf regression; missing tests when required
- **MEDIUM**: partial edge-case coverage; constitution SHOULD violation; weak TDD evidence
- **LOW**: only emit if it still meets the signal bar above (rare — most LOW-tier concerns belong in the Do-Not-Flag list)

### 6. Validation Pass

For **every** finding produced by step 5, re-validate before writing it to `review.md`. Drop any that fail.

- **Bug / correctness**: re-read the cited code in full (not just the diff hunk). Confirm execution actually reaches the cited line, the bug holds for realistic inputs, and surrounding code does not already handle it.
- **Constitution**: re-read the principle text and its scoping. Confirm the rule applies to this file path. Confirm the diff actually breaks it. Confirm no permitted opt-out is present.
- **Requirements coverage**: confirm no other part of the diff satisfies the FR/scenario. Cross-check `tasks.md` task→file mapping.
- **Security**: confirm input is reachable from an untrusted source and the sink is exploitable as written. Drop if reachability requires assumptions outside the diff.

Renumber IDs (`R1, R2, ...`) after drops so the published list is contiguous.

### 7. Write `review.md`

Use this structure:

```markdown
# Code Review: [FEATURE NAME]

**Date**: [ISO date]
**Scope**: [diff range — e.g., `main...HEAD` or `gh pr <N>`]
**Head SHA**: [FULL_SHA]
**Files changed**: [count] ([N new], [M modified], [D deleted])
**Reviewer**: speckit.review

## Verdict

[APPROVE | APPROVE WITH NITS | REQUEST CHANGES | BLOCK]

[1–2 sentence rationale]

## Findings

| ID | Severity | Category | Location | Summary | Suggested Fix |
|----|----------|----------|----------|---------|---------------|
| R1 | CRITICAL | Security | src/auth.ts:42 | SQL string-concat in login | Use parameterized query |

(Stable IDs `R1, R2, ...`. Group rows by severity desc. Skip empty severity buckets.)

## Requirements Coverage

| Requirement | Status | Evidence | Notes |
|-------------|--------|----------|-------|
| FR-001 | Covered | src/foo.ts:10-40, tests/foo.test.ts | |
| FR-002 | Partial | src/bar.ts:55 | Edge case from spec missing |
| FR-003 | Missing | — | No code change observed |

## Constitution Compliance

| Principle | Status | Notes |
|-----------|--------|-------|
| [Principle 1] | Pass | |
| [Principle 2 — TDD] | Fail | Tests + code in one squash commit; no failing-first evidence |

## Test Assessment

- New tests added: [count]
- Spec acceptance scenarios covered: [N/total]
- Gaps: [list]

## Metrics

- Total findings: [N]
- CRITICAL: [N], HIGH: [N], MEDIUM: [N], LOW: [N]
- Requirements covered: [N/total] ([%])
- Files reviewed: [count]
- Findings dropped at validation: [N]

## Recommended Next Actions

1. [Highest-priority fix]
2. ...

Run `__SPECKIT_COMMAND_IMPLEMENT__` to apply fixes, then re-run `__SPECKIT_COMMAND_REVIEW__`.
```

Cap findings table at 50 rows. If overflow, append an "## Overflow" subsection with counts by category.

### 8. Report to User

Print to chat:

- REVIEW path
- Verdict
- Counts by severity
- Top 3 findings (ID + severity + 1-line summary)
- Suggested next command

### 9. Optional: Post GitHub PR Comments (`--comment` only)

Run **only if** `--comment` is in `$ARGUMENTS`.

1. **Resolve PR**: if `--pr <N>` provided, use it. Otherwise `gh pr view --json number,headRefOid,url` to detect the PR for the current branch. Abort with a message if none found.
2. **Capture FULL_SHA** = the PR's `headRefOid`. All inline-comment code links MUST use this exact SHA.
3. **No findings**: post a single summary comment via `gh pr comment <N> --body "..."`:
   ```
   # Code review

   No issues found. Checked spec coverage, plan adherence, constitution compliance, correctness, and security.
   ```
   Stop.
4. **Findings present**: post one **inline comment per unique finding** using `mcp__github_inline_comment__create_inline_comment` (with `confirmed: true`). If MCP tool unavailable, fall back to `gh api` PR review comments. Never post duplicates.

   Each comment body:
   - Brief problem description
   - Citation: link to constitution principle / quote FR-### / spec section
   - Suggested fix:
     - **Small fix** (≤5 lines, self-contained, fully resolves the issue) → include a committable ` ```suggestion ` block
     - **Larger fix** (6+ lines, structural, multi-location, or requires follow-up) → describe the fix, **no** suggestion block
   - **Never** post a committable suggestion unless committing it fixes the issue completely.

5. **Code links** — when referencing other locations from a comment body, use this **exact** format (otherwise the GitHub Markdown preview will not render):

   ```
   https://github.com/<owner>/<repo>/blob/<FULL_SHA>/<path>#L<start>-L<end>
   ```

   Rules:
   - **Full commit SHA** — never `HEAD`, never a branch name, never the result of inline `$(git rev-parse HEAD)`
   - Repo owner/name **must match the PR's repo** (use `gh repo view --json owner,name`)
   - `<path>` is the repo-relative file path
   - `#L<start>-L<end>` placed after the path
   - Provide at least **1 line of context before and after** the cited line. If commenting on lines 5–6, link `#L4-L7`.

6. **Pre-post audit** (do not write this list anywhere — internal check only): mentally enumerate every comment you intend to post. Confirm each is unique, cites real evidence, and would be welcomed by a senior reviewer. Drop any you would not personally defend.

If `--comment` is not provided, do not touch the PR. Local `review.md` is the sole artifact.

### 10. Check for extension hooks

After reporting (and after any optional comment posting), check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.after_review` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Operating Principles

### Review Quality

- **Cite, don't summarize**: every finding has `path:line` and a quote of the offending line.
- **Actionable fixes**: each finding includes a concrete fix, not "consider improving X".
- **HIGH-SIGNAL ONLY**: respect the Do-Not-Flag list in step 5. Drop on doubt.
- **Constitution-scoped**: a principle only applies to files inside its declared scope.
- **No source edits**: this command writes only `review.md` (+ optional GitHub comments). Code changes are deferred to `__SPECKIT_COMMAND_IMPLEMENT__`.
- **Idempotent**: re-running on the same SHA produces stable finding IDs and counts.

### Output Discipline

- Cap findings at 50 rows; overflow goes to an "Overflow" subsection with category counts.
- Skip categories with zero findings (no empty tables).
- Empty diff → one-line `review.md` and exit.

### Parallelism (executor's choice)

The passes in step 5 (A–F) are independent and may be run in parallel by the executing agent (e.g., subagents per pass). Validation in step 6 may also fan out per finding. This is a recommendation, not a requirement; sequential execution is acceptable when the agent has no subagent capability.

## Context

{ARGS}
