---
name: review-my-pr
description: Review a PR against its Jira ticket requirements and code quality before sending to human reviewers. Catches gaps missed by implementation and test plans. Use when a PR is ready for review or when asked to "review my PR".
---

# Review My PR

## Constraints

- **Do not read the implementation or test plans.** Stay independent of them.
- **Pick a fix mode before Phase 3.** See Fix Mode Decision.
- **Do not push.**
- **Do not post comments to GitHub.** Draft findings only; the user posts in their own voice.

## Handle --help Flag

If the user passes `--help`, `-h`, or `help`, display this and stop:

```
/review-my-pr - Review a PR against ticket requirements + code quality

USAGE:
  /review-my-pr <PR-url>
  /review-my-pr <PR-url> <ticket-id>
  /review-my-pr <branch-name>

ARGUMENTS:
  PR URL      Full GitHub PR URL (e.g., https://github.com/org/repo/pull/123)
  ticket-id   Jira ticket ID (optional, derived from branch name if not provided)
  branch      Branch name if no PR exists yet

EXAMPLES:
  /review-my-pr https://github.com/org/repo/pull/548
  /review-my-pr https://github.com/org/repo/pull/548 ABC-123
  /review-my-pr feature-branch
```

## Startup

1. Read CLAUDE.md and project memory (MEMORY.md) from the working directory.
2. **Read the `/code-quality` and `/reduce-my-diff` skills in full before anything else**, and confirm you read both. Phases 3 and 4 execute them.
3. Verify `gh auth status`.

## Parse Arguments

1. For a PR URL, extract org/repo/number and get the branch with `gh pr view <number> --json headRefName`. For a branch name, use it directly.
2. Derive the ticket ID from the branch name (pattern `PROJ-NNNNN` at the start), unless one is given.
3. **Pull the latest before reading any code.**

   ```bash
   git fetch origin <branch> main
   git checkout <branch>
   git reset --hard origin/<branch>     # or `git pull --ff-only` to protect local work
   git log -1 --format='%h %cI %s'      # record this SHA; every finding is anchored to it
   ```

   State the head SHA in the report. Re-fetch and compare SHAs before writing the report and at the start of **every** follow-up turn that re-examines findings. If the SHA moved, `git show` each new commit (`git log --format='%h %cI %s' <old-sha>..origin/<branch>`) and re-verify every open finding against the new head. Never assume a commit resolved a batch.

4. If a PR exists, fetch **all** prior feedback from **three separate API surfaces**. A finding in one does not appear in the others, and `gh pr view --json comments` returns issue comments only:

   ```bash
   # 1. Issue comments (the PR conversation tab)
   gh api repos/<org>/<repo>/issues/<number>/comments \
     --jq '.[] | "[\(.user.login)] \(.created_at)\n\(.body)\n---"'

   # 2. Review bodies (the text a reviewer writes when submitting a review)
   gh api repos/<org>/<repo>/pulls/<number>/reviews \
     --jq '.[] | "[\(.user.login)] state=\(.state) \(.submitted_at)\nBODY: \(.body)\n---"'

   # 3. Inline review comments (anchored to file:line)
   gh api repos/<org>/<repo>/pulls/<number>/comments \
     --jq '.[] | "[\(.user.login)] \(.path):\(.line // .original_line)\n\(.body)\n---"'
   ```

   Split findings into two buckets:
   - **Bot findings** (login matching `cursor`, e.g. `cursor[bot]`): file:line and a one-line summary each.
   - **Human findings**: every named human, including findings they attribute to their own AI review. Record author, file:line, summary, and timestamp.

   Verify each prior finding against current code.

5. **Order prior findings against the commit log.** Compare `git log --format='%h %cI %s' origin/main..HEAD` timestamps to each review's `submitted_at`. A finding may predate its fix, and an odd constant may be a reviewer's request.

---

## Phase 1: Requirement Extraction

**Read the ticket BEFORE the diff**, so you notice what is absent.

### Sources (in priority order)

1. **Acceptance criteria**: each becomes a tracking-table row (Jira API or Atlassian MCP).
2. **Description body**: "must", "always", "never", "required", example messages, reference tables. Requirements often never reached the AC list.
3. **Example messages / mockups**: their formatting (blank-line separators, exact text) is a requirement.
4. **Linked tickets and parent epic**: context, dependencies, prior decisions.
5. **Comments**: scope clarifications and later requirement changes.
6. **Sub-tasks**: each may carry a delegated requirement.

### Output

1. **Ticket Context**: issue type (story/spike/bug), whether ACs exist and how many, parent epic and its status, and any Jira comments that change scope.
2. **D-requirements** (D1, D2...): from the description, examples, and linked context.
3. **AC-requirements** (AC1, AC2...): from the acceptance criteria.

Keep D and AC separate. A conflict between them is a finding.

**Spike-style tickets** (no ACs, bullets like "revisit X, define Y"): say so in Ticket Context, skip strict gap-scoring, and judge whether what was built is sound. Confirm intent with the user before marking a D-requirement NOT MET.

---

## Phase 2: Requirement Mapping

Go **requirement by requirement**, not file by file, recording:
- **Status**: MET, PARTIALLY MET, NOT MET, or NOT IN SCOPE
- **Evidence**: file, line, code
- **Gap**: what is missing, if not fully met

**NOT IN SCOPE requires proof** that it is handled elsewhere or explicitly deferred.

### Catching Gaps the Plans Missed

Run all five:

1. **Ticket before diff**: done in Phase 1.
2. **Negative space**: list requirements with zero evidence in the diff. For each: pre-existing, another PR, or missing?
3. **PR description audit**: mark each claim in the PR body *delivered*, *partial*, or *missing in diff*. Examples: "adds X schema + editor form" (is there a form?), "wires Y into Z" (does Z reference Y?), "introduces DLQ handling" (is a DLQ handler exported?).
4. **Removals in the commit log**: `git log main..HEAD --oneline | grep -iE "remove|delete|rip out|drop"`. Grep for leftovers of each removed surface: type names, JSDoc mentions, case branches, imports.
5. **Repo context**: flag a parallel mechanism where the PR should extend an established pattern.

### Implicit Requirements

Check these even if no AC mentions them:

- **Schema sync**: a new field on one schema appears in every related schema (Zod, Typebox, OpenAPI, GraphQL).
- **N parallel schemas drift**: when one concept lives in several schema systems, compare them side by side. Watch for strict `z.enum(...)` vs `z.string()`, `z.any()` vs `z.unknown()`, required vs optional, and missing fields. The classic symptom: "could save any string to the DB and fail at execution."
- **Gate/conditional completeness**: UI that renders based on which fields exist includes the new field.
- **Existing pattern adherence**: new code matches similar existing code.
- **Constant/copy accuracy**: character-compare constants against exact text in the ticket (legal, regulatory).
- **Test coverage for new code paths**: both branches of new conditionals.
- **Missing lifecycle coverage**: a new resource with an `init`/`dispose` pair (dispatcher, subscription, worker, connection) has a test for disposal.
- **Backwards compatibility**: especially visibility changes (a constructor going `private`, a class moving to `getInstance()`). Verify every call site, including tests.
- **Dead code from earlier refactors** (see technique 4): stale JSDoc, `switch` branches for removed types, comments about removed concepts. File:line each.
- **Broken imports / undefined symbols**: for every `import { X } from "./module"` in the diff (especially tests), verify the module exports `X`.

### Scope Creep

Trace every changed file to a specific ticket requirement.

- **In scope**: "gravity" changes that follow from the primary change: mirrored schema updates, test adaptations, barrel exports, seed data.
- **Out of scope**: surrounding refactors, features not in the ticket, "while I'm here" cleanups.
- **Exception**: a fix for a pre-existing bug found along the way is not flagged negatively.

---

## Fix Mode Decision

- **`commit-fixes`** (default): small, focused PRs. Apply fixes as each skill directs. Commit them separately from feature commits, with messages like `quality: fix [description]`, and report per commit.
- **`draft-only`**: large or judgment-heavy PRs. Run both skills report-only. Commit nothing; number every finding for the user to pick.

Use `draft-only` if **any** of these is true, or if it is borderline:
- The PR touches > ~20 files or > ~1000 lines
- More than ~5 findings need judgment calls ("is this scope creep?", "is this duplication worth unifying?")
- Memory says the user prefers to curate fixes
- The ticket is a spike with no ACs

State the mode in the report. Compute the diff against the **merge base**, after a fetch, so a stale local main does not bloat it: `git fetch origin main && git diff origin/main...HEAD --stat`.

---

## Phase 3: Code Quality

Execute the `/code-quality` skill on the PR's branch, exactly as written. It is the only source for what this step checks, fixes, and reports. If it is unavailable, use `/simplify` or the model's equivalent review tool, and say so in the report.

## Phase 4: Reduce My Diff

Execute the `/reduce-my-diff` skill on the PR, exactly as written, under the same rule. Its approval rules still hold in `commit-fixes` mode.

---

## Phase 5: Review Checks

Reconcile prior feedback before fixing anything.

### Prior-Feedback Reconciliation

#### Human reviews

Mark each human finding **addressed**, **still present**, **stale**, or **open thread** (an unanswered question, or a value under negotiation). Look for:

1. **Findings you missed.** Add them to the Issues table and credit the human, not yourself.
2. **Findings the author already acted on.** Match `submitted_at` to commit timestamps. A request to "lower the retry limit" followed by a commit changing the constant is addressed, not an unexplained divergence.
3. **Live disagreements.** When two humans pull a value opposite ways, or a question is unanswered, say so and point at the thread. Do not advocate a third position.

**Never re-litigate a settled thread.** If the author answered it, note the resolution and move on.

#### Cursor-bot

Bot comments stay pinned to their original commit, so rebased branches collect stale ones. For each, check the current branch:

- Does the file still exist? (`ls <path>`)
- Does the symbol still exist? (grep)
- Did a later commit fix it? (scan `git log main..HEAD --oneline` for the file or an obvious "Fix ..." message)
- Is the issue observable at the flagged file:line now?

Reconciliation output:

```
Humans: N findings from <names>
- H1 (<author>, <file:line>) addressed in <commit-hash>, verified against current code
- H2 (<author>, <file:line>) still present, carry forward as item #X
- H3 (<author>, <file:line>) open thread, author has not answered; do not take a side
- H4 (<author>, <file:line>) MISSED BY THIS REVIEW, add to Issues table, credit the human

Cursor-bot: N findings across the PR's lifetime
- N1 addressed in <commit-hash> (verified, <file:line> no longer has the issue)
- N2 stale, flagged file was removed in <commit-hash>
- N3 still present (<file:line>), carry forward to the Issues table
```

### Unsafe Access Sweep

Run on every diff. Grep the changed files, then rule on each hit by tracing where the value comes from:

```
grep -nE '\[[0-9]+\]|\.find\(|\.at\(|\)!|\]!|payload\.|/ ' <changed files>
```

- **Array index or non-null assertion** (`rows[0]!`): safe only when the producer guarantees the element (a SQL scalar aggregate with no `group by`, a `.map` over a non-empty literal). A `group by`, `limit`, filter, or caller-supplied index guarantees nothing.
- **`.find()` result**: safe only when the next line guards it, or it ends in `?.x ?? fallback`.
- **Callback parameter in `.map` / `.forEach`**: undefined only in sparse arrays; loader or API JSON never is. A `?.` here is style; say so.
- **Chart-library render callbacks** (`payload`, `active`, `viewBox` in recharts): undefined during animation and on empty data; each needs an early return or `?.`.
- **Index into a label or config map** (`LABELS[key]`): safe for a literal union the map covers, or an index from mapping the same array; unsafe for a key from the database or a URL param.
- **Division**: a zero denominator yields `NaN` (`NaN%`, a collapsed bar). Check whether zero is reachable.

Severity: a crash the user can reach is Critical. Rendering `undefined` or `NaN` is Medium. An access the type system proves safe is not a finding, and neither is a `?.` requested on top of it.

When the sweep clears a guard a human asked for, name the producer ("`results` maps over a non-empty `as const` array, so index 0 always exists"), not "looks fine".

### Tests: per-package breakdown

In monorepos, run tests per workspace.

- Find affected packages: `git diff origin/main...HEAD --name-only | awk -F/ '{print $1 "/" $2}' | sort -u`.
- Run each one's test command with the repo's runner (bun, jest, vitest).
- **Set test-only env before giving up.** On env validation errors like "DATABASE_URL required", find the test env the repo documents (its CI workflow, a `.env.test`, or the README) and re-run with it.
- Passes on `main` but fails on HEAD is Critical; fails on both is pre-existing.
- `TypeError: X is not a function` or `X is undefined` means a broken import. Report it as **Critical**: the suite fails CI.

---

## Phase 6: Report

Fixed section order. Every section is mandatory; write "None" rather than dropping a header.

```
## PR Review: PR #<number> (<branch-name>), <ticket-id>

### Ticket Context

<Ticket title>. <Issue type>, <parent epic key> (<epic status>). <One sentence on ACs: "No explicit ACs, spike-style bullet description" or "4 ACs" or "Bug with reproduction steps, no ACs">. <One sentence on Jira comments that change scope, if any>.

### Requirement Coverage

| ID | Requirement | Status | Evidence |
|----|-------------|--------|----------|
| AC1 | ... | MET | file:line |
| AC2 | ... | NOT MET | missing: ... |
| D1 | ... | PARTIAL | file:line, notes |

**For AC-free tickets**: write "Ticket has no ACs; D-requirements are the reviewer's reading of the bullets" before the table.

**Ticket verdict** (1 short paragraph): does the PR deliver what the ticket asked? For spikes, call out mismatches between the ticket's framing (e.g. "reporting dashboard") and the PR's scope (e.g. "a generic event export"), and suggest confirming intent when the gap is interpretation, not quality.

### Gaps Found

Each gap with file:line and why it matters, including PR description vs diff mismatches, undefined symbols in tests, and init/dispose pairs with no disposal test.

If none: "None."

### Implicit Requirement Findings

Per Phase 2 Implicit Requirements category that applies, with file:line: each schema location and its divergence, each stale reference, each compat break with its call sites.

If none: "None."

### Code Quality

The `/code-quality` report, in the format that skill defines. State the fix mode, and list each `quality:` commit, or "None. Drafted findings only, awaiting user triage." in `draft-only` mode.

### Diff Reduction

The `/reduce-my-diff` report, in the format that skill defines.

### Unsafe Access

One line per access the sweep cleared, plus an Issues row for each it did not. Required even when all is safe.

- `file:line`: `<expression>`, safe: `<the producer that guarantees it>`
- Or `Clean: N sites examined (index access, .find, non-null assertions, render callbacks, division)`

List separately any human-requested guard the sweep cleared, with the trace.

### Tests

Per package:
- `packages/foo`: N pass, M fail. If M > 0, name the first failing test and its error.
- `packages/bar`: N pass, 0 fail.

Include type-check per package if run. A TS error the PR caused is Critical, even in a test file the diff did not touch.

### Scope Creep

Each out-of-ticket change with file path and why it does not trace to a requirement. Gravity changes are not scope creep.

If none: "None."

### Review Notes (no action needed)

Lower-signal observations not worth a fix or comment, with file:line.

### Summary

**Stats**
- Requirements: X/Y met (or "N/A, spike ticket")
- Gaps: N found
- Quality fixes: N applied (or `0, draft-only mode`)
- Scope creep: `none` | `N items flagged`
- Recommendation: one of
  - `ready for human review`
  - `needs attention on [specific items by #]`
  - `blockers present, see items #X, #Y` (if any Critical item exists)

**Issues & Concerns (priority order)**

List **every** finding from every phase, even if an earlier section already lists it. This is the canonical triage list. Number top to bottom by priority:

1. **Critical**: blocking and high priority are one tier. Missing AC, failing CI, a test that can never pass (undefined imports), TS compile error, security issue, data loss risk, resource leak, clear bug, requirement gap, behavior-changing scope creep, unsafe parallel-schema divergence. Every row blocks the merge.
2. **Medium**: performance work (unbounded or repeated query, avoidable scan, hot-path allocation), code-quality findings, implicit requirement misses, duplicated construction that will drift, stale references to removed concepts.
3. **Low**: minor duplication, review notes worth mentioning.

**Performance findings are Medium**, however large the speedup, unless the cost takes the service down or loses data; then Critical, and the row says why.

**Code style is not a finding.** Naming, formatting, idiom, and comment wording go in Review Notes, unless they cause a defect; then rank on the defect.

**Test-only findings cap at Medium.** A finding whose blast radius stops at tests, evals, fixtures, or harness code is Medium or Low however severe (a false-passing suite, an assertion that can never fire, missing branch coverage). Deployed code that only runs tests (a scheduled test job, a test CLI, a test-only API route) counts as test-only. Two exceptions keep their real severity:

- A broken test that fails CI. The red gate blocks the merge.
- Code shared by tests and production (a constant, prompt, schema, or helper both paths read). Rank it on the production consequence, and name the production path.

When the ceiling is doing work, state it in the row: `Medium (test-only)`.

Format:

| # | Priority | Type | Issue | Location | Human flagged? | Cursor-bot? |
|---|----------|------|-------|----------|----------------|-------------|
| 1 | Critical | Functionality | <one sentence of consequence, not just pattern> | src/api/client.ts:42 | No | No |
| 2 | Critical | Functionality | Null deref possible when `user.profile` is undefined | src/user.ts:42 | Yes, @reviewer | Yes, flagged |
| 3 | Medium | Code quality | Builds the same request config in three handlers, so a header change must land three times | src/api/handlers.ts:88 | Yes, @reviewer (missed by this review) | No |
| 4 | Medium | Code quality | Re-runs an unbounded table scan on every page load | src/reports/query.ts:12 | No | No |
| 5 | Low | Diff reduction | Reformats 40 untouched lines in a file the ticket does not need | src/routes.ts:10 | No | No |

**This table is mandatory in every report.** Never replace it with prose or a bullet list. With zero findings, keep the header and write one row: `| - | - | - | None | - | - | - |`.

**Type values** (exactly one per row; if a finding fits two, pick the first in this order):
- `Functionality`: the fix changes what deployed code does. Requirement gaps, bugs, unsafe access that crashes or renders `undefined` or `NaN`, failing tests or CI, schema drift that lets bad data through, and scope creep that changes behavior.
- `Code quality`: works, but harder to read, change, or run than needed. Every `/code-quality` finding, plus dead code, stale references, duplication, missing test coverage, and performance work.
- `Diff reduction`: larger than the task needs. Every `/reduce-my-diff` finding, plus scope creep that does not change behavior.

**Human flagged? values:**
- `No`: this review found it independently.
- `Yes, @name`: a human raised it and this review found it too.
- `Yes, @name (missed by this review)`: a human raised it and this review did **not** find it. Mark these honestly.
- `Yes, @name (open thread)`: unresolved between humans. Report the thread's state; do not pick a side.

Human-flagged rows rank **above** equal-severity rows nobody raised.

Write the **consequence**, not the pattern. "Leaks a database connection on every request" beats "pool client not released". "Could save any string to the DB and fail at execution" beats "status schema is z.string()".

**Human review reconciliation** (short paragraph after the table)

- `<N> humans have reviewed: @a raised X findings on <date>, @b raised Y. Z addressed in commit <hash>. W still present, items #A, #B. V remain open threads.`
- In its own sentence, name each human finding this review did **not** find independently, with the count.
- An AI-generated human review ("I ran a Claude review") still counts as human-raised; say so.
- If none: `Humans: no review comments on PR.`

**Cursor-bot reconciliation** (short paragraph after the human one)

- `Cursor-bot flagged N issues across the PR's lifetime. X addressed in commit <hash>. Y stale (file/symbol removed). Z still present, carried forward as items #A, #B.`
- If it raised something this review did not: `Cursor-bot also noted: <summary>` with your verdict.
- No PR yet: `Cursor-bot: N/A (branch only, no PR).` No comments: `Cursor-bot: no comments on PR.`

**Closing line** (draft-only mode only)

"I did not commit any changes. Point to item numbers above and I'll stage them as a single `quality:` commit."
```
