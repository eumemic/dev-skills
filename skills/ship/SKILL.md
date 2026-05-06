---
name: ship
description: Drive a code change from working tree to a green PR — runs project checks, /simplify, code-review subagent, commits, opens PR, monitors CI. Stops when CI is green; caller handles merge. Use when invoked as `/ship`, or when another skill (e.g., `/shovel-ready`, `/kaizen`, `/bughunt`) hands off a working tree of changes to be shipped. Project-agnostic; reads check commands from CLAUDE.md.
---

# /ship — drive a working-tree change to a green PR

You're driving a change that is **already implemented** on a non-default branch through the publish pipeline: project checks → structural-quality pass → code-review pass → commit → push → PR → green CI. You stop when CI is green. The caller decides whether to merge automatically (e.g., `/shovel-ready --auto`, `/kaizen --auto`) or wait for a human signal.

This skill is invoked with optional context from the caller — a one-line summary of what changed, a commit type/scope, and an issue number for `Closes #N`. If the caller doesn't pass them, infer from the diff and the conversation.

## Pre-conditions

- Working tree is on a non-default branch with the intended changes either staged or unstaged. If still on the default branch (master/main), refuse and tell the caller — branch creation is the caller's responsibility.
- Code is implemented and locally validated by the caller (i.e., they at minimum eyeballed the diff). /ship adds the publish-pipeline checks on top.

## Phase 1 — Pre-flight

Run these in parallel:
- `git status` — confirm branch + dirty state.
- `git diff` — see unstaged changes.
- `git diff --cached` — see staged changes.
- `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name` — confirm the default branch name (master vs main).

If `git status` shows the default branch, stop and tell the caller. Don't auto-branch — the caller's intent for branch naming and base is information you don't have.

If both staged and unstaged changes exist, ask the caller (via `AskUserQuestion`) whether to commit only staged or stage everything modified-tracked. Don't `git add -A` — that sweeps in runtime artifacts (`.logs/`, lockfiles, spool DBs).

## Phase 2 — Run all checks

Read `CLAUDE.md` (project root) for the project's check commands. The standard pattern is "do all three before every commit":
- A type-checker (mypy / pyright / tsc / ...)
- A linter + format-check (ruff / eslint+prettier / ...)
- A unit test runner (pytest / vitest / ...)

Run them in parallel where possible. If anything fails, **stop and report**. The caller may have an in-flight implementation issue you can't fix without their input. Don't paper over a real test failure.

If the project has e2e or integration tests gated on Docker or external services, **don't run them as part of /ship**. CI runs them on push. Local pre-push e2e gating is friction without payoff. (Exception: if the change touches migrations / alembic / DB schema, run `alembic heads` or equivalent locally to catch head-collision issues mypy can't see.)

## Phase 3 — Structural-quality pass (optional)

If a `/simplify` skill (or equivalent: "kaizen", "code-cleanup", "polish") is in the available-skills list, invoke it now via the `Skill` tool. It typically launches parallel review agents (reuse / quality / efficiency) and applies their findings inline.

If no `/simplify` skill exists, do a manual self-review: scan the diff for redundant state, copy-paste with slight variation, dead code paths, comments explaining WHAT instead of WHY, and the project's flagged anti-patterns from CLAUDE.md.

If the change is a one-line typo fix or trivial config bump, skip this phase — the overhead exceeds the value. Pass `--no-simplify` to skip explicitly.

## Phase 4 — Code-review pass (optional)

If a code-reviewer subagent is configured (e.g., `pr-review-toolkit:code-reviewer`, `feature-dev:code-reviewer`), launch it with:
- The diff (`git diff HEAD`)
- A brief description of the change's intent (caller-supplied or inferred)
- The relevant issue context if `--issue=N` was passed
- Project-specific severity guidance from CLAUDE.md or memory

The reviewer returns severity-scored findings. Apply actionable findings (typically sev ≥ 70) before opening the PR. Note sub-70 findings in the PR description for transparency — score triages but doesn't suppress.

If the project's memory says "Post all code review findings, don't filter by score" or similar, follow that instead. Don't filter findings without explicit guidance.

If no code-reviewer subagent exists, self-review the diff against the change's intent.

Pass `--no-review` to skip.

## Phase 5 — Commit

Stage explicit files — never `git add -A`. Use either:
- `git add -u` to stage all tracked modifications (safe — won't pick up new untracked files).
- Or list paths explicitly: `git add path/a path/b ...`.

Compose the commit message:

- **Title**: `<type>(<scope>): <imperative summary under 72 chars>`. Type from caller's `--commit-type=` or inferred from the diff (`refactor` for code-shape changes with no behavior change, `fix` for bug fixes, `feat` for new behavior). Scope from the topmost-changed module.
- **Body**: 2–6 lines explaining the *why*. Reference prior PRs/issues by number. If the caller passed `--issue=N`, include `Closes #N` on its own line at the end.
- **Co-Authored-By trailer**: if AI-authored, end with `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>`.

Pass the body via heredoc to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
type(scope): summary

Body explaining why.

Closes #N

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If a pre-commit hook runs additional checks and fails, **fix the underlying issue and create a NEW commit**. Do not `--amend` after a hook failure — the commit didn't happen, so amend would modify the *previous* commit and risk losing work. Re-stage and re-commit.

## Phase 6 — Push

`git push -u origin <branch>` to set upstream on first push.

If the branch has been pushed before and CI is currently running, **stop the active CI monitor first** before force-pushing. Force-push invalidates the running CI run; the existing monitor will see "all checks complete" on the canceled run. Re-arm the monitor after every push that resets CI.

## Phase 7 — Open the PR

```
gh pr create --title "<conventional title>" --body "$(cat <<'EOF'
## Summary

<2-3 bullets covering what changed and why>

## Test plan

- [x] Type-checker — clean
- [x] Linter + format-check — clean
- [x] Unit suite — N passed
- [ ] CI runs <e2e/integration suite>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Closes #<N>  <!-- only if --issue= was passed -->
EOF
)"
```

Title under 70 chars; details belong in the body. The `Closes #N` footer is load-bearing for GitHub's auto-close — `(#N)` syntax in commits references but doesn't close.

If the caller passed `--draft`, add `--draft` to `gh pr create`.

Return the PR URL to the caller.

## Phase 8 — Monitor CI

Use the `Monitor` primitive — wakes on stdout events without burning prompt cache while idle. The poll body emits one line per check transition out of pending and exits when all are terminal:

```bash
prev=""
while true; do
  s=$(gh pr checks <PR-N> --json name,bucket 2>/dev/null || echo "[]")
  cur=$(echo "$s" | jq -r '.[] | select(.bucket!="pending") | "\(.name): \(.bucket)"' | sort)
  comm -13 <(echo "$prev") <(echo "$cur")
  prev=$cur
  echo "$s" | jq -e 'length>0 and all(.bucket!="pending")' >/dev/null && break
  sleep 30
done
echo "ALL CHECKS COMPLETE"
```

Schedule a fallback `ScheduleWakeup` (~1500s) in case the Monitor misfires. The Monitor is the primary wake signal; the wakeup is the safety net.

When CI is green, **stop here**. Report:
- The PR URL
- The final CI status (all green, or which checks failed)
- Whether the caller should merge or wait

If CI fails, surface the failing check names and short error excerpts. Do **not** auto-fix CI failures unless the caller explicitly asked. If they do ask: pull the failing check log via `gh run view --log-failed`, diagnose, push a fix, and re-monitor.

## What /ship does NOT do

- **Does not merge.** Caller decides whether to `gh pr merge` or wait for a human.
- **Does not branch.** Caller creates the branch with their preferred naming/base.
- **Does not implement code.** Implementation exists in the working tree before /ship is invoked.
- **Does not pick targets.** Caller knows what's being shipped.
- **Does not loop.** A single invocation drives one PR to a green state.

## Boundaries

- **Don't `git add -A`.** Tracked-only (`-u`) or explicit paths.
- **Don't push to the default branch.** Always work on a feature branch.
- **Don't skip checks "because it's a tiny change".**
- **Don't `--amend` after a pre-commit hook failure.** New commit; preserves the integrity of the failed-then-recovered sequence.
- **Don't `--no-verify` on commits** unless the user explicitly asks; fix the hook failure rather than bypass it.
- **Don't run e2e locally.** CI runs them.
- **Don't assume the default branch is `master`.** Read it from `gh repo view`.

## Optional flags

- `--issue=<N>` — include `Closes #<N>` in the PR body (caller-supplied for issue-driven flows).
- `--commit-type=<type>` — pin the conventional-commit type (`fix`, `feat`, `refactor`, `chore`, `docs`, `test`).
- `--commit-scope=<scope>` — pin the scope.
- `--summary=<one-line>` — caller-provided one-line summary for the title; otherwise inferred from the diff.
- `--draft` — open the PR as draft.
- `--no-simplify` — skip Phase 3.
- `--no-review` — skip Phase 4.

## When to escalate

Use `AskUserQuestion` when:
- Tests fail and the fix isn't obvious from the diff alone.
- The reviewer raises a sev ≥ 80 finding that flips the design.
- The diff has both staged and unstaged changes and the intent is ambiguous.
- The caller passed `--issue=N` but the change clearly doesn't address that issue's acceptance criteria.

When in doubt, ask. The marginal cost of a question is small; the marginal cost of a bad PR landing on the user's queue is the rest of their afternoon.
