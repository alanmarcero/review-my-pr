# review-my-pr

A Claude Code skill that reviews a PR against its Jira ticket before human reviewers see it. It reads the ticket first, maps each requirement to the diff, runs the `/code-quality` and `/reduce-my-diff` skills, checks prior human and Cursor Bugbot comments against current code, and runs the tests. It does not read the implementation or test plans, so it can catch gaps they missed.

## Install

Copy `SKILL.md` into your skills directory:

```sh
mkdir -p ~/.claude/skills/review-my-pr
cp SKILL.md ~/.claude/skills/review-my-pr/
```

## Requirements

- The `code-quality` and `reduce-my-diff` skills, installed in `~/.claude/skills/`. Phases 3 and 4 run them as written. Without `code-quality`, the review falls back to `/simplify` and says so in the report.
- The `gh` CLI, logged in with read access to the repo.
- Jira access through the Atlassian MCP server or the Jira API, to read the ticket.

## Usage

```
/review-my-pr <PR-url>
/review-my-pr <PR-url> <ticket-id>
/review-my-pr <branch-name>
```

If you do not give a ticket ID, the skill takes it from the branch name (`PROJ-NNNNN` at the start). Run `/review-my-pr --help` for the full usage text.

## Fix modes

- `commit-fixes` (default): for small, focused PRs. The skill commits each fix separately with a `quality:` prefix.
- `draft-only`: for large PRs (more than about 20 files or 1000 lines), spikes, or PRs with many judgment calls. The skill commits nothing and numbers each finding so you can pick which ones to apply.

In both modes, the skill does not push and does not post comments to GitHub.

## Report

The report has a fixed set of sections: ticket context, requirement coverage, gaps, implicit requirements, code quality, diff reduction, unsafe access, tests, scope creep, review notes, and a summary.

The summary always ends with one table of every finding, in priority order:

| Column | Values |
|--------|--------|
| Priority | Critical, Medium, Low |
| Type | Functionality, Code quality, Diff reduction |
| Issue | The consequence, in one sentence |
| Location | `file:line` |
| Human flagged? | Whether a human reviewer raised it, and whether this review missed it |
| Cursor-bot? | Whether Cursor Bugbot raised it |

The table is present even when there are no findings.
