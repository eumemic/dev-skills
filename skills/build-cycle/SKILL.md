---
name: build-cycle
description: Reference document shared by the build-iteration skills (/shovel-ready, /kaizen, /bughunt) for their common single-iteration machinery — branch, sync-with-master, /ship handoff, /retro pass, merge, and declaring the loop outcome. NOT user-invokable, and NOT the loop. The loop lives in `/loop`; this is one iteration's build cycle. Only load when a build-iteration skill directs you here.
---

# build-cycle — shared single-iteration build skeleton

> **Don't invoke this directly, and don't loop from here.** This is loaded by `/shovel-ready`, `/kaizen`, and `/bughunt` for the build steps they share within **one** iteration. Looping (cadence, idle escalation, re-arming) is owned by `/loop` — see that skill. If a user types `/build-cycle`, redirect them to a specialization, or to `/loop <specialization>` to run it as a loop.

This document defines the common skeleton for **one iteration** of a build-type development skill: take a target, ship it on a branch through `dev-skills:ship`, run a per-iteration `/retro` pass, merge, then **declare the outcome** that `/loop` reads to decide what happens next. Each specialization (`/shovel-ready`, `/kaizen`, `/bughunt`) plugs in its own **target acquisition** (Phase 1) and **implementation discipline** (Phase 3); everything else lives here.

`kaikaku` does **not** use this skeleton — it grooms an architectural proposal instead of building, so it owns its whole iteration and only shares the outcome contract.

## The outcome contract

An iteration's last act is to declare its result so `/loop` can pace the loop:

```
LOOP-OUTCOME: <status> — <one-line detail>
```

Build iterations emit one of: `shipped` (a PR merged or opened for the user), `rejected` (a candidate the user closed unmerged), `empty` (target acquisition found nothing), `gate_killed` (candidates found, none cleared Phase 1.5), or `blocked` (escalated to the user mid-iteration). They never emit `filed` (that's kaikaku's). See `dev-skills:loop` for the full table and what `/loop` does with each. **Always end with this line** — a missing outcome is a contract violation that strands the loop.

## Phase model

| Phase | Owner | Description |
|---|---|---|
| 1 | Specialization | **Target acquisition.** How a candidate is found (issue queue, parallel-agent audit, etc.). Empty → declare `empty` and stop the iteration. |
| 1.5 | Specialization | **Quality gate.** The top candidate must clear a specialization-defined bar; otherwise declare `gate_killed` and stop the iteration (both are normal, non-productive outcomes). |
| 2 | build-cycle | Branch and plan. |
| 3 | Specialization | **Implement.** Specialization-specific discipline (TDD vs. structural-only vs. labeled-issue work). |
| 3.5 | build-cycle | Sync with master before `/ship` (rebase if behind). |
| 4 | build-cycle | Hand off to `dev-skills:ship`. |
| 5 | build-cycle | `/retro` pass — iteration mode. |
| 6 | build-cycle | Merge (auto or wait-for-user). |
| 7 | build-cycle | Declare `LOOP-OUTCOME`. |

There is no idle-counter, no re-arm, and no cadence logic here — those moved to `/loop`. One invocation of a specialization = one iteration that ends at Phase 7.

## Flags this cycle interprets

(`/loop` owns the loop flags — `--once`, `--idle-count`, `--drain` — and passes these through to the specialization verbatim.)

- `--auto` — merge automatically once CI passes (`gh pr merge --squash --delete-branch`). Default: wait for the user to merge in the GitHub UI.
- `--model=<sonnet|opus|haiku>` — pin spawned subagents (the specialization's Phase 1 audit agents, and the code-reviewer inside `/ship`, forwarded verbatim via `--model=`) to a specific Claude generation. The main session keeps its own model; this only affects `Agent`-tool calls. Default (omitted): subagents inherit the session's model.
- `--issue=<N>` — include `Closes #<N>` in the PR body (load-bearing for `/shovel-ready` auto-close; optional for `/bughunt`; n/a for `/kaizen`).

## Shared invariants

- **One change per iteration.** Even if you spot two adjacent candidates, ship one. The other is the next iteration's target. Reviewer attention and revert blast-radius stay scoped.
- **No scope expansion mid-iteration.** Adjacent smells become *next* iteration's candidates. Trust the loop.
- **Fail hard, no fallbacks.** Per the project's stance (read `CLAUDE.md`), the model sees raw errors and retries through the session log. Don't suppress.
- **Quality gate beats throughput.** Each specialization defines its own bar (Phase 1.5). "Found something but not worth shipping" is a normal, healthy outcome — declare `gate_killed` and let `/loop` pace it. Don't ship a borderline change because "we have to ship something."

## Quality gate principle (Phase 1.5)

**Every build skill has a temptation to find SOMETHING every iteration to justify the run.** Resist it. If the top candidate doesn't clear the specialization's bar, **report what was found and why the bar wasn't met, then declare `gate_killed`** — don't silently ship a borderline change.

What "clearing the bar" means is specialization-specific:
- `/shovel-ready`: implicit — issues bearing the label have already cleared a human-applied bar (the gate mainly applies in Audit Mode, when deciding whether to *label* candidates).
- `/kaizen`: the top candidate must be unambiguously accidental complexity, not borderline ("might be load-bearing"). A net-positive line count is a yellow flag.
- `/bughunt`: provability ≥ 0.6 AND severity-on-real-paths above triviality.

## Phase 2 — Branch and plan

```bash
default=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
git fetch origin "$default"
git checkout -b <prefix>/<short-slug> "origin/$default"
```

Branch prefix and slug convention is specialization-specific:
- `/shovel-ready`: descriptive name including the issue number if convention allows.
- `/kaizen`: `kaizen/<3–5-word-kebab-case>` naming the **target**.
- `/bughunt`: `bughunt/<short-symptom-slug>` naming the **symptom**, not the fix.

Plan with `TaskCreate`: implement → sync-with-master → /ship → /retro → merge → declare outcome.

## Phase 3.5 — Sync with master before `/ship`

Master may have advanced while this iteration was in flight. The PR's CI runs against the merge of your branch into current master, so a stale base can fail on tests, schemas, or interfaces master updated mid-iteration. Rebase forward before handing off:

```bash
default=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
git fetch origin "$default"
behind=$(git rev-list HEAD..origin/$default --count)
if [ "$behind" -gt 0 ]; then
    git rebase "origin/$default"
fi
```

If the rebase conflicts, surface via `AskUserQuestion` (declare `blocked`) — don't auto-resolve. After a successful rebase, re-run the project's standard checks since the base shifted; a clean git merge can still interact semantically with your diff. If checks now fail, diagnose whether the rebase caused it (a master change conflicting with your diff) or an unrelated issue, and decide whether to fix-forward or abandon.

## Phase 4 — Hand off to `dev-skills:ship`

Invoke `dev-skills:ship` via the `Skill` tool. Pass:
- `--commit-type=<fix|feat|refactor>` — see specialization for default.
- `--commit-scope=<topmost-changed-module>`.
- `--summary=<one-line description>` — the PR title.
- `--issue=<N>` if applicable.
- `--model=<value>` if it was passed (applied to `/ship`'s code-reviewer subagent).

`/ship` runs project checks, `/simplify`, the code-review subagent (if configured), commits, pushes, opens the PR, and monitors CI until terminal. It returns when CI is green. It does **not** merge — that's Phase 6.

If `/ship` reports a failing check or a sev ≥ 80 review finding requiring design changes, treat as a blocker: diagnose, fix, re-invoke `/ship`. Don't skip ahead.

## Phase 5 — `/retro` pass (iteration mode)

After `/ship` returns CI-green and **before** Phase 6, invoke `dev-skills:retro` with `--scope=iteration`. It reflects on this iteration's friction signals and produces zero or more durable action items (memory entries, skill edits, scripts, repo issues). It applies its own quality gate — most iterations produce nothing, which is the expected default.

If retro produces skill edits or scripts that touch the **current branch's repo**, commit and push them to the same PR and wait for CI green again. Memory entries / repo issues that don't touch the branch are lightweight side effects — proceed. If retro escalates via `AskUserQuestion`, answer it before merging.

## Phase 6 — Merge

### If `--auto`

```bash
gh pr merge <N> --squash --delete-branch
```

Verify `gh pr view <N> --json state,mergedAt`. A server-side success with a local "master is already used by another worktree" failure is harmless. For `/shovel-ready`: also verify auto-close (`gh issue view <N> --json state` → CLOSED); if still OPEN, the PR body lacks `Closes #N` — fix with `gh pr edit <N>`.

→ Phase 7 with `shipped`.

### Otherwise (wait for the user)

Persistent `Monitor` polling `gh pr view <N> --json state` until `MERGED` or `CLOSED`. Optionally `PushNotification` once CI is green: "PR #N green and ready for your merge."

- `MERGED` → Phase 7 with `shipped`.
- `CLOSED` without merge → the user rejected the change. Acknowledge; don't argue. Record the rejected target so the next iteration's acquisition doesn't re-pick it (specialization-specific: `/shovel-ready` comments on the issue; `/kaizen` records the target; `/bughunt` records the symptom). → Phase 7 with `rejected`.

## Phase 7 — Declare the outcome

End the iteration with the `LOOP-OUTCOME` line (see the contract above). `shipped` / `rejected` from Phase 6; `empty` from Phase 1; `gate_killed` from Phase 1.5; `blocked` if you escalated and are waiting on the user. `/loop` reads this and owns everything that happens next.

## Shared boundaries

- **Don't loop from here.** No `ScheduleWakeup`, no idle-counter, no re-arm. End at Phase 7; `/loop` decides the rest.
- **Don't bundle.** One target per PR.
- **Don't skip the structural-quality + code-review passes.** Both `/simplify` and the code-review subagent catch different issue types. If neither is configured, do them manually before opening the PR.
- **Don't merge without CI green.** Even if the diff is "obviously safe."
- **Don't push to the default branch directly.** Always a branch + PR.
- **Don't force-push to a PR after CI started without re-arming the monitor.** Force-push invalidates the running CI run.
- **Don't speculate in the PR body.** Describe what changed and why; file follow-ups separately.

## Shared escalation triggers

`AskUserQuestion` (and declare `blocked`) mid-iteration when:
- Two top candidates are within ROI noise of each other (let the user pick).
- A reviewer raises a sev ≥ 80 finding mid-`/ship` that flips the design.
- A test fails for a reason you didn't expect.
- The fix would change a public-facing API or wire format.
- A Phase 3.5 rebase conflicts.

When in doubt, ask. The marginal cost of a question is small; the marginal cost of a bad PR is the loop's credibility.
