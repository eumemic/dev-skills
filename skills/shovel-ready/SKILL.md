---
name: shovel-ready
description: One iteration against a GitHub `shovel-ready` issue queue — if non-empty, take the highest-ROI issue (TDD, /ship, /retro, merge); if empty, audit closed work and unlabeled candidates to refill it; declare the outcome. Use when the user invokes `/shovel-ready`, asks to "work the shovel-ready queue", "drain shovel-ready", "drive issues to merge", "autopilot on labeled issues"; use `/loop shovel-ready --auto --drain` to clear the queue autonomously. GitHub-specific (uses `gh` CLI); skip for GitLab/Jira.
---

# /shovel-ready — one iteration against a labeled issue queue

You run **one iteration** against a GitHub repository's `shovel-ready` label queue, routed by queue state: if it has issues, take one to merge; if it's empty, audit to refill it. `/loop shovel-ready` runs this on a loop; `/loop shovel-ready --drain` clears the queue back-to-back in one turn.

This skill is a thin specialization over the shared **build-cycle** skeleton. **Read `dev-skills:build-cycle` for the shared iteration phases (branching, /ship handoff, /retro, merge, declaring the outcome).** Looping — long-cadence waiting, idle escalation, re-arming — is owned by `/loop`. This file owns:

- **Phase 1** — target acquisition (queue state detection + mode routing).
- **Audit Mode** — closure audit + unlabeled-candidate audit to refill an empty queue.
- **Phase 3** — the shovel-ready-specific implementation discipline (TDD per issue).

Everything else — branching, `/ship` invocation, `/retro` pass, merge handling, the closing `LOOP-OUTCOME` — lives in `build-cycle`.

`Monitor` wakes on CI completion (build-cycle Phase 6). Within one iteration, in-turn transitions (audit→execute) chain in the same turn — never `ScheduleWakeup` between them; that's a self-imposed delay. Cross-iteration waiting is `/loop`'s job, not yours.

## Specialization-specific flags

(See build-cycle for the shared `--auto` / `--model=`. The loop flags — `--idle-count`, `--once`, `--drain` — belong to `/loop`, which passes the ones below through to you.)

- `--auto` — merge automatically once CI passes; skip the wait-for-user-merge step. Use for autonomous queue clearing (`/loop shovel-ready --auto`).
- `--skip-audit` — a one-shot signal from the previous iteration: it just audited an empty queue with no result, so skip Audit Mode this once (don't re-ask the user immediately) and declare `empty`. You **consume** it — clear it via `carry-drop: --skip-audit` so the iteration after re-audits. You set it via `carry-add: --skip-audit` after your own no-result audit (see Empty queue after audit).

## Specialization-specific invariants

(See build-cycle for the shared invariants. These are *additional*.)

- **Only `shovel-ready`-labeled issues are eligible.** Don't infer readiness from issue body or your own judgment. The label is a deliberate readiness gate; respect it. If you find a well-scoped issue that lacks the label, surface it in the audit pass — don't silently work it.
- **TDD discipline is mandatory.** For bug fixes: write a failing repro test, watch it fail with the right error, then fix. For features: write tests asserting the new behavior, watch them fail, then implement. Don't claim a fix without seeing the test go red→green.
- **Auto-close via `Closes #N` in PR body.** GitHub doesn't auto-link from `(#N)` commit syntax — only "Closes / Fixes / Resolves" keywords in the PR body close the issue. Always include one (and `--issue=<N>` to `/ship` ensures this).

## Phase 1 — Detect queue state and route

```
gh issue list --label shovel-ready --state open --json number,title,body,labels
```

Routing:

- **Non-empty queue** → **Execute Mode** (below): take one issue to merge.
- **Empty queue, no `--skip-audit`** → **Audit Mode** (below): try to refill, then re-route in-turn.
- **Empty queue, `--skip-audit` set** → declare `LOOP-OUTCOME: empty — queue empty (audit skipped this cycle) · carry-drop: --skip-audit`. `/loop` handles the long-cadence wait; clearing the one-shot flag means the iteration after this one re-audits.

The "just audited, nothing found" signal can't live in conversation context — it resets across `/loop`'s re-entries. It travels as the `--skip-audit` flag, set and cleared via the `carry-add`/`carry-drop` outcome directives (see `dev-skills:loop`).

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

If no issue clears even this light gate: rare, but declare `LOOP-OUTCOME: gate_killed`. (The gate failing here likely means stale labels — flag for audit.)

### Branch and plan

Per build-cycle Phase 2. Use a descriptive branch name that includes the issue number if convention allows (e.g., `fix/handle-empty-roster-42` for issue #42). Don't reuse an existing branch — start fresh per issue.

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

When invoking `dev-skills:ship` (build-cycle Phase 4):
- `--commit-type=<fix|feat|...>` — usually `fix` for bug-shaped issues, `feat` for new behavior.
- `--issue=<N>` — load-bearing; `/ship` includes `Closes #<N>` in the PR body so GitHub auto-closes on merge.

### build-cycle Phase 5 (`/retro`) and Phase 6 (merge)

Per build-cycle, with one shovel-ready-specific verification after merge:

After `--auto` merge OR user-merge, verify auto-close: `gh issue view <N> --json state` should be CLOSED. If still OPEN, the PR body is missing a `Closes #N` line — fix with `gh pr edit <N>` and re-check.

### Declare the outcome

After merge (build-cycle Phase 6), declare `LOOP-OUTCOME: shipped — #<N>` (or `rejected` if the user closed the PR; `blocked` if you escalated). Then stop — `/loop` decides whether to run the next iteration. Clearing a full queue back-to-back in one turn is `/loop --drain`'s job (target acquisition here is cheap — `gh issue list`, not parallel agents — which is exactly when `--drain` pays off), not an in-skill loop.

## Audit Mode (queue empty, audit not yet run)

When the queue is empty, this iteration tries to refill it before declaring `empty`. Two sub-audits, each with a confirmation gate.

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

When done, return to Phase 1 **in the same turn** — do NOT `ScheduleWakeup` at the audit→execute boundary. If even one issue got labeled, the queue is non-empty and Execute Mode runs immediately; if not, fall through to the empty-queue outcome below. Putting a wake between audit and execute is a self-imposed delay with no upside.

## Empty queue after audit

When the queue is empty and Audit Mode found nothing to label or close, declare:

```
LOOP-OUTCOME: empty — shovel-ready queue empty; audit found nothing to label · carry-add: --skip-audit
```

The `carry-add: --skip-audit` tells `/loop` to skip the audit on the *immediately* following iteration (so the user isn't re-asked the audit questions every wake); that iteration consumes and clears it (`carry-drop`), so auditing resumes the wake after. Everything else — the long-cadence wait, the idle-count streak, and the "still want me watching this queue?" escalation at idle-count 3 — is owned by `/loop`. You declare the outcome; `/loop` paces the wait.

## Specialization-specific boundaries

(Additive to build-cycle's shared boundaries.)

- **Don't auto-close issues without confirmation.** Even strong-signal closure candidates get an `AskUserQuestion` first — closure is communicative, not mechanical.
- **Don't auto-label issues without confirmation.** Same reason. Tier-1 candidates still get a "label these four?" gate.
- **Don't bundle.** One issue per PR. If you find two issues that need the same fix, close one as duplicate of the other and ship one PR.
- **Don't skip TDD.** No exceptions. If you can't write a failing test, the change isn't well-defined enough to ship.
- **Don't overlook `Closes #N`.** Without it, the issue stays open after merge and the queue silently fills with done work.

## Specialization-specific escalation

`AskUserQuestion` when (in addition to build-cycle's shared triggers):
- The reviewer raises a sev ≥ 80 finding that flips the design (silent-clear semantics, missing FK guards, wrong abstraction). Apply the fix and verify with the user before proceeding to PR.
- Two reasonable implementations of the issue have meaningfully different blast radii. Don't pick silently.
- A test fails for a reason you didn't expect. Investigate before forcing it green; the wrong test passing is worse than the right test failing.

When in doubt, ask. The marginal cost of a question is small; the marginal cost of a bad PR is the rest of your session.
