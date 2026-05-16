---
name: shovel-ready
description: Drive a GitHub `shovel-ready` issue queue end-to-end — pick highest-ROI, TDD, /ship, /retro, merge; when empty, audit closed work and unlabeled candidates to refill it; otherwise wait for new labels. Use when the user invokes `/shovel-ready`, asks to "work the shovel-ready queue", "drain shovel-ready", "drive issues to merge", "autopilot on labeled issues", "issue-label queue", or sets up an autonomous-development loop on a GitHub-labeled issue tracker. GitHub-specific (uses `gh` CLI); skip for GitLab/Jira.
---

# /shovel-ready — work a labeled issue queue to completion

You're driving a GitHub repository's `shovel-ready` label queue from open to merged through three modes, routed by queue state at each iteration. The same prompt re-enters this skill on every wake; the mode is a function of `gh issue list --label shovel-ready --state open`, not anything the prompt carries.

This skill is a thin specialization over the shared loop-driver framework. **Read `dev-skills:loop-driver` for the shared phases (branching, /ship handoff, /retro, merge, idle escalation, flag passthrough).** This file owns:

- **Phase 1** — target acquisition (queue state detection + mode routing).
- **Audit Mode** — closure audit + unlabeled-candidate audit (replaces some of loop-driver Phase 8 for the empty-queue case).
- **Wait Mode** — specializes loop-driver Phase 8 with shovel-ready's own `--skip-audit` interleave.
- **Phase 3** — the shovel-ready-specific implementation discipline (TDD per issue).

Everything else — flags (mostly), branching, `/ship` invocation, `/retro` pass, merge handling — lives in `loop-driver`.

This is a long-running loop. `ScheduleWakeup` is for **Wait Mode** and **CI fallback** only — never for in-turn phase transitions like audit→execute or merge→pick-next, which chain in the same turn. `Monitor` wakes on CI completion. Don't sleep-poll or hold the conversation open.

## Specialization-specific flags

(See loop-driver for shared flags — `--auto`, `--model=`. `/shovel-ready` does NOT expose `--once` because the queue's natural emptiness already provides a stop condition. `--model=` is forwarded to `/ship`; shovel-ready itself has no direct subagent spawns.)

- `/shovel-ready` — default; audit if queue is empty. Default merge behavior: wait for the user to merge in the GitHub UI.
- `/shovel-ready --auto` — merge automatically once CI passes; skip the wait-for-user-merge step. Use for autonomous queue clearing.
- `/shovel-ready --skip-audit` — Wait Mode directly; the previous iteration just audited with no result, no point re-asking the user immediately.
- `/shovel-ready --idle-count=N` — Wait Mode with the streak counter (per loop-driver Phase 8).

Flags compose: e.g., `/shovel-ready --auto --skip-audit --idle-count=2` is a wait-mode wake on the autonomous-merge variant. Pass through whatever flags were set on every self-arming `ScheduleWakeup`.

## Specialization-specific invariants

(See loop-driver for the shared invariants. These are *additional*.)

- **Only `shovel-ready`-labeled issues are eligible.** Don't infer readiness from issue body or your own judgment. The label is a deliberate readiness gate; respect it. If you find a well-scoped issue that lacks the label, surface it in the audit pass — don't silently work it.
- **TDD discipline is mandatory.** For bug fixes: write a failing repro test, watch it fail with the right error, then fix. For features: write tests asserting the new behavior, watch them fail, then implement. Don't claim a fix without seeing the test go red→green.
- **Auto-close via `Closes #N` in PR body.** GitHub doesn't auto-link from `(#N)` commit syntax — only "Closes / Fixes / Resolves" keywords in the PR body close the issue. Always include one (and `--issue=<N>` to `/ship` ensures this).

## Phase 1 — Detect queue state and route

```
gh issue list --label shovel-ready --state open --json number,title,body,labels
```

Routing:

- **Non-empty queue** → **Execute Mode** (below). Reset `--idle-count` and drop `--skip-audit` on the next wake's prompt.
- **Empty queue, no `--skip-audit`** → **Audit Mode** (below).
- **Empty queue, `--skip-audit` set** → **Wait Mode** (below).

The "audit attempted this session" signal does NOT live in conversation context — context resets across `ScheduleWakeup` re-entries (especially at long cadence). Carry the signal via the wake prompt itself: after a no-result audit, re-arm with `prompt="/shovel-ready --skip-audit"`. After the next iteration does *anything* productive, drop the flag.

## Execute Mode (queue has 1+ issues)

### Pick highest-ROI

ROI ≈ (operator/user pain × confidence-this-fixes-it) ÷ effort. Concretely:

- Smaller scope wins ties.
- Bugs with wire-level repros beat speculative refactors.
- Issues whose acceptance criteria are fully written beat ones with open questions.
- "I worked around this in production" issues outrank "nice to have" issues.

If you genuinely can't rank two, ask the user via `AskUserQuestion`. Don't toss a coin.

### Phase 1.5 — Quality gate (light)

For Execute Mode, the gate is implicit: the label IS the gate. Issues bearing `shovel-ready` have already cleared a human-applied bar. The only Phase 1.5 check here:

- **Issue still actionable?** Body hasn't been updated to "blocked on X", scope hasn't ballooned since labeling, no comment from another contributor saying "actually let's not." If the issue should no longer be shovel-ready, remove the label with a comment explaining what changed and pick a different one.

If no issue clears even this light gate: rare, but proceed to loop-driver Phase 8 as if Audit Mode found nothing. (The gate failing here likely means stale labels — flag for audit.)

### Branch and plan

Per loop-driver Phase 2. Use a descriptive branch name that includes the issue number if convention allows (e.g., `fix/handle-empty-roster-42` for issue #42). Don't reuse an existing branch — start fresh per issue.

Plan with `TaskCreate`: investigate → write failing tests → implement → /ship → /retro → merge. Mark each in_progress as you start.

### Investigate

Read the issue body fully. If it has open questions (sections labeled "Open question", lines ending with `?`, design forks the author flagged), use `AskUserQuestion` BEFORE writing code. Don't assume the answer; the issue author flagged the question for a reason.

Find code touchpoints via grep. Read enough surrounding code to understand the existing patterns the fix should mirror. Aim to mirror, not invent.

### Phase 3 — Implement

Write failing tests first (TDD discipline above). For bug fixes: replicate the broken behavior. The test should fail with the exact symptom the issue describes (KeyError, wrong return value, missing event, etc.). If the test fails for some other reason, the test is wrong before the code is. For features: tests assert the new behavior. They should fail with `AttributeError: 'X' object has no attribute 'Y'` or similar — proving the feature doesn't exist yet.

Run the test. Confirm it fails for the right reason. Only then move to implementation.

If the change has no observable behavior to test (pure CI tweak, dependency bump, doc-only change), the issue probably isn't shovel-ready as the gate intends — kick it back to audit by removing the label with a comment explaining what's missing, rather than fabricating a test to satisfy the rule.

Make the minimal change needed to pass the failing tests. Don't refactor adjacent code unless it's load-bearing for the fix.

### `/ship` defaults

When invoking `dev-skills:ship` (loop-driver Phase 4):
- `--commit-type=<fix|feat|...>` — usually `fix` for bug-shaped issues, `feat` for new behavior.
- `--issue=<N>` — load-bearing; `/ship` includes `Closes #<N>` in the PR body so GitHub auto-closes on merge.

### Loop-driver Phase 5 (`/retro`) and Phase 6 (merge)

Per loop-driver, with one shovel-ready-specific verification after merge:

After `--auto` merge OR user-merge, verify auto-close: `gh issue view <N> --json state` should be CLOSED. If still OPEN, the PR body is missing a `Closes #N` line — fix with `gh pr edit <N>` and re-check.

### Pick next

Return to Phase 1 in-turn. Productive iteration → re-arm at 60s per loop-driver Phase 8 productive branch. (Note: shovel-ready in Execute Mode often picks the next issue in the SAME turn rather than re-arming, since target-acquisition is fast — `gh issue list` rather than parallel agents. If you do this, skip the 60s wake; just loop Phase 1 again.)

## Audit Mode (queue empty, audit not yet run)

When the queue is empty, the loop should fill it before settling into Wait Mode. Two sub-audits, each with a confirmation gate.

### Audit closure (issues that should be closed because PRs landed)

GitHub's auto-link only fires when commits/PRs use "Closes / Fixes / Resolves" syntax. Many repos use `(#N)` instead, which references but doesn't close. So issues stay open after their work shipped.

Procedure:
1. List all open issues. List recent merged PRs (`gh pr list --state merged --limit 30`).
2. For each open issue, check `gh issue view <N> --json closedByPullRequestsReferences`. If non-empty and merged, the issue can be closed cleanly.
3. For issues with empty `closedByPullRequestsReferences` but recent PR work in their area, manually cross-reference. Look for `(#issue)` in commit messages, "Phase X" trackers shipped over multiple PRs, etc.
4. Group findings by tier:
   - **Strong**: PR body or commits explicitly cite this issue's full scope as delivered.
   - **Probably superseded**: design issue replaced by a later design issue.
   - **Skip**: not enough signal.
5. Present the strong + probably-superseded list to the user via `AskUserQuestion`. Don't unilaterally close.
6. On confirmation, close each with a comment that cites the merging PR(s) and the scope coverage. Use `gh issue close <N> --comment "..."`.

### Audit shovel-readiness (unlabeled candidates)

Some issues are practically shovel-ready but lack the label — usually because they were filed as follow-ups during a different PR's roll-up and never triaged. Find them.

For each unlabeled open issue, classify by gap:
- **Tier 1** — design settled, scope clear, files/lines identified, full acceptance criteria, no open questions. Just needs the label. (Clears the audit-mode quality gate.)
- **Tier 2** — 1–2 small decisions remain (e.g., "should this work on update too?", "scope fix to provider X only?"). Resolvable with `AskUserQuestion`. (Clears the gate after resolution.)
- **Tier 3** — multiple decisions in design space, no recommendation in the body, or the author explicitly says "not shovel-ready / placeholder". (Fails the gate.)
- **Tier 4** — investigation/discovery needed before implementation. (Fails the gate.)
- **Tier 5** — explicitly placeholder (issue body says so). (Fails the gate.)

Present the tiered list to the user. For Tier 1, ask "label all?"; for Tier 2, ask the small decisions one at a time and comment the resolved decision on the issue before labeling — quote the user's resolution verbatim in the comment so it becomes the rationale future readers see when re-opening the issue. Skip Tier 3+.

When done, return to Phase 1 **in the same turn** — do NOT `ScheduleWakeup` at the audit→execute boundary. If even one issue got labeled, the queue is non-empty and Execute Mode runs immediately; if not, fall through to Wait Mode. Putting a wake between audit and execute is a self-imposed delay with no upside.

## Wait Mode (queue empty after audit)

This is shovel-ready's specialization of loop-driver Phase 8. Same machinery (long-cadence re-arm, idle-count escalation), with the shovel-ready-specific addition of the `--skip-audit` interleave on "Keep going."

Schedule a long-cadence `ScheduleWakeup` (~3600s, the runtime maximum). The skill re-enters on wake; if the queue is now non-empty (someone labeled a new issue), Execute Mode runs; otherwise it falls back to Wait Mode again.

The wake prompt carries an idle-streak counter so consecutive empty waits escalate. Read `--idle-count=N` from the invocation; if absent, treat as 0.

Always pass through the `--auto` flag if it was set on the original invocation — every self-arming wake should preserve it.

- **idle-count < 3** — re-arm with `ScheduleWakeup` and bump the counter:
  ```
  prompt="/shovel-ready --skip-audit --idle-count=<N+1><pass-through --auto>"
  delaySeconds=3600
  reason="No shovel-ready issues, idle streak <N+1>; long-cadence poll."
  ```
- **idle-count == 3** (≈ 3 hours of nothing) — `AskUserQuestion`: "still want me watching the shovel-ready queue?"
  - If yes → reset counter, re-arm fresh: `prompt="/shovel-ready<pass-through --auto>"`, no `--skip-audit` (re-audit on next iteration to catch issues that may have been added).
  - If no → omit `ScheduleWakeup`, stop any active `Monitor`s, report the loop is paused.
- **idle-count > 3** — only happens if the user said yes at the 3-mark; treat the same as < 3 from there until the user explicitly stops the loop.

Don't poll faster than 1h — at 1h cadence the prompt-cache cost is amortized over a full re-entry, and labelling latency rarely matters in minutes. Don't escalate the user-check more often than every 3 wakes.

To stop the loop manually at any time: omit the `ScheduleWakeup` and TaskStop any active `Monitor`s.

## Specialization-specific boundaries

(Additive to loop-driver's shared boundaries.)

- **Don't auto-close issues without confirmation.** Even strong-signal closure candidates get an `AskUserQuestion` first — closure is communicative, not mechanical.
- **Don't auto-label issues without confirmation.** Same reason. Tier-1 candidates still get a "label these four?" gate.
- **Don't bundle.** One issue per PR. If you find two issues that need the same fix, close one as duplicate of the other and ship one PR.
- **Don't skip TDD.** No exceptions. If you can't write a failing test, the change isn't well-defined enough to ship.
- **Don't overlook `Closes #N`.** Without it, the issue stays open after merge and the queue silently fills with done work.

## Specialization-specific escalation

`AskUserQuestion` when (in addition to loop-driver's shared triggers):
- The reviewer raises a sev ≥ 80 finding that flips the design (silent-clear semantics, missing FK guards, wrong abstraction). Apply the fix and verify with the user before proceeding to PR.
- Two reasonable implementations of the issue have meaningfully different blast radii. Don't pick silently.
- A test fails for a reason you didn't expect. Investigate before forcing it green; the wrong test passing is worse than the right test failing.

When in doubt, ask. The marginal cost of a question is small; the marginal cost of a bad PR is the rest of your session.
