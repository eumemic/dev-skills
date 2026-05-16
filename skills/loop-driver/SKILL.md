---
name: loop-driver
description: Reference document shared by /shovel-ready, /kaizen, and /bughunt for their common autonomous-loop machinery (branching, /ship handoff, /retro pass, merge handling, idle escalation, flag passthrough). NOT user-invokable. Only load when one of the three loop specializations explicitly directs you here.
---

# loop-driver — shared reference for autonomous loop skills

> **Don't invoke this directly.** This skill is loaded by `/shovel-ready`, `/kaizen`, and `/bughunt` for the phases they share. If a user types `/loop-driver`, redirect them to one of the three specializations.

This document defines the common skeleton for an autonomous-development loop driver: take a target, ship it on a branch through `dev-skills:ship`, run a per-iteration `/retro` pass, merge, then either re-arm or escalate. Each specialization (`/shovel-ready`, `/kaizen`, `/bughunt`) plugs in its own **target acquisition** logic at Phase 1; everything else lives here.

## Phase model

| Phase | Owner | Description |
|---|---|---|
| 1 | Specialization | **Target acquisition.** How a candidate is found (issue queue, parallel-agent audit, etc.). |
| 1.5 | Specialization | **Quality gate.** Top candidate must clear a specialization-defined bar; otherwise the iteration ships nothing (a normal outcome). |
| 2 | loop-driver | Branch and plan. |
| 3 | Specialization | **Implement.** Specialization-specific discipline (TDD vs. structural-only vs. labeled-issue-feature work). |
| 4 | loop-driver | Hand off to `dev-skills:ship`. |
| 5 | loop-driver | `/retro` pass — iteration mode. |
| 6 | loop-driver | Merge (auto or wait-for-user). |
| 7 | loop-driver | Reset idle counter. |
| 8 | loop-driver | Loop control (re-arm or escalate). |

## Flags

All three specializations share these flags. Pass them through verbatim on every self-arming `ScheduleWakeup`.

- `--auto` — merge automatically once CI passes (`gh pr merge --squash --delete-branch`). Default: wait for the user to merge in the GitHub UI.
- `--once` — run a single iteration; do not `ScheduleWakeup` at the end. Default: loop indefinitely. (`/shovel-ready` doesn't expose `--once` because the queue's natural emptiness already provides a stop condition.)
- `--model=<sonnet|opus|haiku>` — pin spawned subagents to a specific Claude generation. Applied at every subagent spawn point this loop owns: the specialization's Phase 1 audit agents, and the code-reviewer subagent inside `/ship` (passed through verbatim via `--model=`). The main session keeps whatever model it started on; this flag only affects `Agent`-tool calls. Default (omitted): no `model:` parameter — subagents inherit the session's model.
- `--idle-count=<N>` — internal carry-over for the empty-iteration streak counter (Phase 8). Don't set manually.

Flags compose. Self-arming wakes preserve every flag the user originally invoked with, plus mutations to `--idle-count`.

## Shared invariants

- **One change per PR.** Even if you spot two adjacent candidates, ship them separately. Reviewer attention and revert blast-radius stay scoped.
- **No scope expansion mid-iteration.** Adjacent smells become *next* iteration's candidates. Trust the loop.
- **Pass through flags on every wake.** `ScheduleWakeup`-driven re-entries lose conversation context; the prompt itself must be self-describing.
- **Fail hard, no fallbacks.** Per the project's stance (read `CLAUDE.md`), the model sees raw errors and retries through the session log. Don't suppress.
- **Quality gate beats throughput.** Each specialization defines its own bar. "Found something but not worth shipping" is a normal iteration outcome — not a failure to be avoided. See Phase 1.5.

## Quality gate principle

**Every loop driver has a temptation to find SOMETHING every iteration to justify the wake.** Resist this.

The quality gate is the per-iteration version of the idle-escalation Phase 8 mechanism. Phase 8 catches "audit found nothing." Phase 1.5 catches "audit found things, but none clear the bar." Both are healthy outcomes; both feed into idle-count.

What "clearing the bar" means is specialization-specific:
- `/shovel-ready`: implicit — issues bearing the label have already cleared a human-applied bar. The gate barely applies in Execute mode. (It does apply during Audit mode, when deciding whether to label unlabeled candidates.)
- `/kaizen`: top candidate must be unambiguously accidental complexity, not borderline ("might be load-bearing"). A net-positive line count is a yellow flag.
- `/bughunt`: top candidate must have provability ≥ 0.6 AND severity-on-real-paths above triviality. A reachable bug that no one would ever notice isn't worth a PR.
- `/retro` (when invoked from the loop-driver Phase 5): findings must be codifiable as durable artifacts (memory, skill edit, script, repo issue) — not just session-local "huh, that was annoying."

If the top candidate doesn't clear the bar, **report what was found and why the bar wasn't met, then proceed to Phase 8 as if the iteration were empty.** Don't silently ship a borderline change because "we have to ship something."

## Phase 2 — Branch and plan

```bash
default=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
git fetch origin "$default"
git checkout -b <prefix>/<short-slug> "origin/$default"
```

Branch prefix and slug convention is specialization-specific:
- `/shovel-ready`: descriptive name including the issue number if convention allows.
- `/kaizen`: `kaizen/<3–5-word-kebab-case>` naming the **target** (e.g., `kaizen/collapse-defer-retry-wake`).
- `/bughunt`: `bughunt/<short-symptom-slug>` naming the **symptom**, not the fix (e.g., `bughunt/missing-roster-on-group-rename`).

Plan with `TaskCreate`: implement → /ship → /retro → merge → loop control. Mark `in_progress` as you start each step.

## Phase 4 — Hand off to `dev-skills:ship`

Invoke `dev-skills:ship` via the `Skill` tool. Pass:
- `--commit-type=<fix|feat|refactor>` — see specialization for default.
- `--commit-scope=<topmost-changed-module>`.
- `--summary=<one-line description>` — used as the PR title.
- `--issue=<N>` if applicable. Load-bearing for `/shovel-ready` (auto-close on merge); optional for `/bughunt` (only if the bug was already filed); not applicable to `/kaizen`.
- `--model=<value>` if `--model=` was passed to the loop. `/ship` applies it to its code-reviewer subagent.

`/ship` runs project checks, `/simplify` pass, code-review subagent (if configured), commits, pushes, opens the PR, and monitors CI until all checks are terminal. It returns when CI is green.

If `/ship` reports a failing check or a sev ≥ 80 review finding requiring design changes, treat as a blocker — diagnose, fix, re-invoke `/ship`. **Don't skip ahead to Phase 5.**

`/ship` does NOT merge. Merge is owned by Phase 6 of this driver.

## Phase 5 — `/retro` pass (iteration mode)

After `/ship` returns with CI green and **before** Phase 6 merge, invoke `dev-skills:retro` via the `Skill` tool with `--scope=iteration`. Retro reflects on this iteration's friction signals (not the whole session) and produces zero or more action items:

- **Memory entries** — feedback/project/reference notes capturing learnings.
- **Skill edits** — improvements to any of the loop drivers, `/ship`, `/retro` itself, or other invoked skills.
- **Scripts** — small utilities that would have made this iteration smoother.
- **Repo issues** — feature requests on the *target repo* (not on the skill repos) for groundwork that would have unblocked a faster fix.

The retro applies its own quality gate: most iterations should produce nothing. "No actionable findings this round" is the expected default. See `dev-skills:retro` for details.

If retro produces skill edits or scripts that affect the **current branch's repo**, commit and push them to the same PR. CI will re-run; wait for it green again before proceeding to Phase 6. If retro produces memory entries or repo issues that don't touch the current branch, those are lightweight side effects — proceed directly to Phase 6.

If retro escalates a question via `AskUserQuestion`, answer it before proceeding. Don't merge with retro work pending.

## Phase 6 — Merge

### If `--auto` was passed

```bash
gh pr merge <N> --squash --delete-branch
```

Verify: `gh pr view <N> --json state,mergedAt`. If the merge succeeds on GitHub but the local checkout fails (e.g., "master is already used by another worktree"), that's harmless — the merge is on the server. Proceed to Phase 7.

For `/shovel-ready` only: also verify auto-close with `gh issue view <N> --json state` (should be CLOSED). If still OPEN, the PR body is missing a `Closes #N` line; fix with `gh pr edit <N>` and re-check.

### Otherwise (default — wait for the user to merge)

Persistent `Monitor` polling `gh pr view`:

```bash
PR=<pr-number>
prev=""
while true; do
  state=$(gh pr view "$PR" --json state --jq .state 2>/dev/null || echo "ERR")
  if [ "$state" != "$prev" ]; then
    echo "PR #$PR state: $state"
    prev="$state"
  fi
  case "$state" in
    MERGED|CLOSED) break ;;
  esac
  sleep 60
done
```

`persistent: true`. Optionally `PushNotification` once CI is green (and again post-retro): `"PR #N green and ready for your merge."`

When `MERGED`: report and proceed to Phase 7.

When `CLOSED` without merge: the user rejected the change.
- Acknowledge; don't argue.
- Specialization-specific cleanup:
  - `/shovel-ready`: comment on the originating issue explaining what was attempted, so the next iteration doesn't re-pick it without context.
  - `/kaizen`: record the rejected target in conversation context so the next iteration's audit doesn't re-pick it.
  - `/bughunt`: record the rejected symptom so the audit doesn't re-discover it.
- Proceed to Phase 7.

## Phase 7 — Reset idle counter

A productive iteration (PR merged or closed-without-merge after a real attempt) means we found *something* this round. Reset `--idle-count` to 0 for the next wake.

A non-productive iteration (Phase 1.5 quality gate killed the iteration before branching) does NOT reset the counter — see Phase 8.

## Phase 8 — Loop control

If `--once` was passed, **stop here.** Report the iteration outcome and exit.

Otherwise, decide cadence and re-arm:

### If this iteration shipped a candidate (productive)

Re-arm immediately:

```
ScheduleWakeup({
  delaySeconds: 60,
  prompt: "/<skill><pass-through flags>",
  reason: "<skill> iteration complete; immediately scanning for next candidate."
})
```

(Pass through `--auto` if set; never set `--once` in self-arms.)

### If the audit was empty OR the quality gate killed the iteration (non-productive)

Increment `--idle-count` by 1.

- **idle-count < 3** — long-cadence re-arm (codebase looks clean for now; check back in an hour):
  ```
  ScheduleWakeup({
    delaySeconds: 3600,
    prompt: "/<skill> --idle-count=<N+1><other-flags>",
    reason: "<skill>: no candidate cleared the bar; idle streak <N+1>; long-cadence re-audit."
  })
  ```

- **idle-count == 3** — escalate via `AskUserQuestion`:
  - "Three consecutive iterations found no candidate worth shipping. Keep going, redirect (deeper audit / different dimensions), or stop?"
  - On "Keep going": reset idle-count, re-arm fresh: `prompt="/<skill><other-flags>"`.
  - On "Stop": omit `ScheduleWakeup`, stop any active `Monitor`s, report the loop is paused.

- **idle-count > 3** — only happens if the user said "Keep going" at the 3-mark; treat the same as < 3 from there.

### Pacing notes

- Don't poll faster than 1h at idle. At 1h cadence the prompt-cache cost is amortized; faster polls just waste tokens.
- Don't escalate the user-check more often than every 3 wakes — frequent confirmation prompts outweigh their value.
- Only `/shovel-ready` Wait Mode and `/bughunt`/`/kaizen` empty audits use the long cadence. Productive iterations re-arm at 60s.

## Shared boundaries

- **Don't bundle.** One target per PR. Adjacent candidates wait for the next iteration.
- **Don't skip the structural-quality + code-review passes.** Both `/simplify` and the code-review subagent (if configured) catch issues of different types. If neither is configured, do them manually before opening the PR.
- **Don't merge without CI green.** Even if the diff is "obviously safe", let CI confirm.
- **Don't push to default branch directly.** Always work on a branch and open a PR.
- **Don't force-push to a PR after CI started without re-arming the monitor.** Force-push invalidates the running CI run; the existing monitor will see "all checks complete" on the canceled run.
- **Don't speculate in the PR body.** Describe what changed and why. File follow-ups separately.

## Shared escalation triggers

`AskUserQuestion` mid-iteration when:
- Two top candidates are within ROI noise of each other (let the user pick).
- A reviewer raises a sev ≥ 80 finding mid-`/ship` that flips the design.
- A test fails for a reason you didn't expect.
- The fix would change a public-facing API or wire format.
- Three consecutive iterations exit non-productive (Phase 8 escalation).

When in doubt, ask. The marginal cost of a question is small; the marginal cost of a bad PR is the rest of the loop's credibility.
