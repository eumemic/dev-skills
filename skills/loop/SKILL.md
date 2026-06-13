---
name: loop
description: Drive any single-iteration development skill as an autonomous loop — invoke one iteration, read its declared outcome, then pace, escalate, or stop. Use when invoked as `/loop <skill> [flags]` (e.g. `/loop kaizen --auto`, `/loop bughunt`, `/loop kaikaku`, `/loop shovel-ready --auto`). Owns cadence, idle escalation, and re-arming so the iteration skills stay pure. Pass `--once` to run exactly one iteration.
---

# /loop — autonomous loop driver

You drive a **single-iteration** development skill (`kaizen`, `bughunt`, `kaikaku`, `shovel-ready`, …) as a loop. Your only job is the loop itself: run one iteration, read what it reported, then decide whether to re-arm, escalate, or stop — and at what cadence. **You never touch repo content.** Branching, implementing, shipping, merging, filing — all of that belongs to the iteration skill. This separation is the whole point: an iteration skill does one unit of work and knows nothing about looping; you own the looping and know nothing about the work.

Invocation: `/loop <skill> [iteration-flags] [loop-flags]`. Example: `/loop kaizen --auto`, `/loop kaikaku`, `/loop shovel-ready --auto --drain`.

## The iteration contract

Every loopable iteration skill ends its run by declaring **exactly one** outcome line:

```
LOOP-OUTCOME: <status> — <one-line detail>
```

You read that line (you just ran the iteration this turn, so it's in your own context) and branch on `<status>`:

| status | meaning | productive? | your reaction |
|---|---|---|---|
| `shipped` | a change merged, or opened for the user to merge | yes | reset idle → re-arm **fast** (60s) |
| `filed` | a groomed artifact filed (e.g. a kaikaku design-issue) | yes | reset idle → re-arm **long** (the artifact awaits human review; nothing to fast-follow) |
| `rejected` | produced a candidate, but the user closed it unmerged | yes (it found something) | reset idle → re-arm **fast**; the iteration already recorded the rejected target so it won't re-pick it |
| `empty` | target acquisition found nothing | no | idle++ → re-arm **long** |
| `gate_killed` | candidates found, none cleared the iteration's quality gate | no | idle++ → re-arm **long** |
| `blocked` | the iteration needs the user and already escalated | — | **stop** (don't re-arm; the iteration owns the `AskUserQuestion`) |

If the iteration ends without a clear `LOOP-OUTCOME` line, treat it as `blocked` and report that the iteration skill didn't declare an outcome (a contract violation worth fixing in that skill) — don't guess and re-arm into an unknown state.

### Carrying a signal to the next wake

An iteration may need to influence the *next* invocation's flags (e.g. `/shovel-ready` setting `--skip-audit` after a no-result audit, so it doesn't re-audit immediately). The outcome line may append a **carry directive**:

```
LOOP-OUTCOME: empty — queue empty, audit found nothing  ·  carry-add: --skip-audit
LOOP-OUTCOME: shipped — fixed #42  ·  carry-drop: --skip-audit
```

When re-arming, compute the next wake's iteration flags as **(current iteration flags) + every `carry-add` − every `carry-drop`**. Absent a directive, pass the flags through unchanged. You never interpret what a carried flag *means* — only the iteration does; you just apply the delta to the prompt.

## Loop flags (yours)

- `--once` — run exactly one iteration, then stop. Never re-arm. (This replaces the old per-skill `--once`: "run once" is now just `/loop <skill> --once`, or invoke the iteration skill directly.)
- `--idle-count=<N>` — the non-productive streak counter you carry across wakes (Phase 4). Don't set it manually; you bump it.
- `--drain` — for cheap-target-acquisition skills (e.g. `shovel-ready` draining a labeled queue): on a productive outcome, run the **next** iteration in the **same turn** instead of re-arming, looping in-turn while productive and only re-arming (long) when you hit a non-productive outcome. Without it (default), every productive iteration re-arms via wake — healthier for expensive skills (one PR per wake keeps CI spaced and each iteration's context clean).

**Everything else is an iteration flag** you pass through verbatim and never interpret: `--auto`, `--model=`, `--skip-audit`, `--issue=`, etc. You don't know what `--auto` means (merge-without-waiting for kaizen, file-without-approval for kaikaku) — the iteration does. Carry every flag the user invoked with on every self-arming wake.

## Phase 1 — Run one iteration

Invoke the target iteration skill via the `Skill` tool, forwarding the iteration flags. Let it run to completion — it acquires a target, applies its own quality gate, does its work (or reports it found nothing), and declares `LOOP-OUTCOME`.

If the iteration escalates a question mid-run via `AskUserQuestion`, that's the iteration's call — answer flows back to it. If it ends `blocked`, you stop (Phase 4).

## Phase 2 — Read the outcome

Read the `LOOP-OUTCOME` line from the iteration you just ran. Map it via the contract table to: **{productive?, cadence, stop?}**.

## Phase 3 — Idle accounting

- **Productive** (`shipped` / `filed` / `rejected`): reset `--idle-count` to 0.
- **Non-productive** (`empty` / `gate_killed`): increment `--idle-count` by 1.
- `blocked`: no idle change; you're stopping.

## Phase 4 — Pace, escalate, or stop

If `--once` was passed: **stop here.** Report the single iteration's outcome and exit.

Otherwise:

### Productive → re-arm

```
ScheduleWakeup({
  delaySeconds: <60 if cadence==fast, else 1800–3600>,
  prompt: "/loop <skill> <all-iteration-flags> --idle-count=0",
  reason: "<skill> <status>; re-arm <fast|long> for the next iteration."
})
```

With `--drain` set and a `fast` cadence, skip the wake and **return to Phase 1 in the same turn** instead — loop in-turn while productive.

### Non-productive → long-cadence re-arm, with escalation

- **idle-count < 3** — re-arm long:
  ```
  ScheduleWakeup({
    delaySeconds: 3600,
    prompt: "/loop <skill> <flags> --idle-count=<N+1>",
    reason: "<skill> <status>; idle streak <N+1>; long-cadence re-check."
  })
  ```
- **idle-count == 3** — escalate via `AskUserQuestion`: "Three consecutive `<skill>` iterations found nothing worth shipping. Keep going, redirect (deeper audit / different dimensions / different skill), or stop?"
  - **Keep going** → reset idle-count, re-arm fresh: `prompt="/loop <skill> <flags>"`.
  - **Redirect** → re-arm with the user's chosen change (e.g. a different iteration skill, or deeper-audit flags).
  - **Stop** → omit `ScheduleWakeup`, `TaskStop` any active `Monitor`s, report the loop is paused.
- **idle-count > 3** — only reached if the user said "keep going" at the 3-mark; treat as `< 3` until they stop it.

### `blocked` → stop

The iteration already escalated to the user and is waiting on them. Don't re-arm — re-entry would re-ask. Report that the loop is paused pending the user's answer; it resumes when they re-invoke `/loop <skill>`.

## Cadence reference

- **Cache window matters** (the prompt cache has a ~5-min TTL): for fast re-arms use **60s** (well inside the window). For long re-arms, **1800–3600s** — you'll pay one cache miss, but there's genuinely nothing to do sooner, so amortize it. Never re-arm in the 300–600s dead zone (pays the cache miss without buying a meaningful wait).
- **Architectural skills** (`kaikaku`, anything emitting `filed`) re-arm long even when productive — a filed proposal awaits human review; there's no fast-follow target, and an un-reviewed backlog is itself a stop signal (the iteration should report that as `gate_killed`/`blocked` when it notices).
- **Don't poll faster than 1h at idle.** Don't escalate the user-check more often than every 3 wakes.

## Boundaries

- **You never edit repo content, branch, ship, merge, or file.** If you're tempted to, the iteration skill is incomplete — fix it there, not here.
- **You never re-interpret iteration flags.** Pass them through verbatim; the iteration owns their meaning.
- **One iteration per Phase-1 invocation** (except `--drain`, which loops Phase 1 in-turn). You don't bundle work; the iteration's own "one change per unit" invariant handles that.
- **The wake prompt is the only state that survives.** Context resets across `ScheduleWakeup` re-entries, so the prompt must carry every flag + the idle-count. Self-describing or it breaks.
- **Stop cleanly.** On `--once`, `blocked`, or a user "stop", omit the wake and `TaskStop` any `Monitor`s the iteration left running.

## Escalation

`AskUserQuestion` only for loop-level decisions:
- The idle-count-3 escalation above.
- The iteration declared no `LOOP-OUTCOME` (contract violation) — report it rather than re-arming blind.

Everything else — design forks, reviewer findings, ambiguous targets — is the iteration's to escalate, not yours.
