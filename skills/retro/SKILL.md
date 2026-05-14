---
name: retro
description: Reflect on a slice of the current session (a single iteration, the whole session, or a natural pause point) to identify durable, codifiable learnings about workflow OR dev infrastructure — and ship them only if they clear a quality bar. Most invocations produce nothing; that's the point. Use when the user asks to "run a retro", "do a retrospective", "reflect on the session", "what did you learn", "improve skills based on this session", "codify learnings"; when invoked from a loop driver (`/shovel-ready`, `/kaizen`, `/bughunt`) with `--scope=iteration`; or **proactively at pause points** when the agent is waiting on async work (CI, autodev runs, monitor events) and has cycles to think meta.
---

# /retro — quality-gated retrospective

Reflect on a slice of the conversation, **identify only durable codifiable learnings, and apply a quality gate** before proposing or executing changes. Most retros — especially per-iteration ones — should produce nothing actionable, and that's the healthy default.

**Scope is broad: workflow AND dev infrastructure.** A retro should consider not just "did the agent's skills handle this well" but also "could the underlying tooling have been better" — autodev pipeline gaps, aios memory features, eumemic-ops audit checks, missing CLI affordances, etc. The action item types section below makes this explicit (constellation-wide issues, not just target-repo issues).

## Modes

`/retro` runs in one of three scopes:

- **`--scope=iteration`** — invoked by the loop drivers (`/shovel-ready`, `/kaizen`, `/bughunt`) at Phase 5, between `/ship` CI-green and merge. Analyzes only this iteration's events (since branch creation). Action items that touch the current branch's repo get committed to the same PR; everything else is a side effect.
- **`--scope=session`** (default) — invoked standalone by the user. Analyzes the full conversation history. Action items always go to whichever repo they belong to, separately from any in-flight work.
- **`--scope=pause`** — agent-initiated at a natural pause point (waiting on autodev/CI/monitor, mid-coffee in the conversation flow). Analyzes the slice since the last retro or, if none this session, since the conversation start. Behaves like `--scope=session` but with a stricter quality gate: pause-point retros run frequently, so noise is more costly.

If neither flag is passed and there's no loop-driver caller, default to `session`. If invoked from a loop driver without `--scope`, that's a bug in the caller — assume `iteration` and continue.

## When to self-invoke (pause mode)

The agent should proactively trigger `/retro --scope=pause` when **all** of these hold:

1. There's a natural pause: waiting on async work (autodev job, CI run, long Monitor task), or the user has just confirmed a milestone and there's no immediate next user message expected.
2. The session has produced concrete events since the last retro: filings, failures, fixes, user corrections, surprising results.
3. The agent has cycles — i.e., the pause is long enough that running a retro doesn't compete with reactive work.

Self-invocation is signaled by saying "running /retro at this pause point" and proceeding through the phases. **The quality gate still applies** — most pause-point retros produce nothing, and that remains the healthy default. Self-invocation is permission to *consider*, not permission to *ship*.

When in doubt, skip. A missed pause-point retro is invisible; a noisy one drains the user's attention budget.

## The bias to resist

The temptation in a retro is to find SOMETHING worth changing every time. Resist it. **Wrong skill changes — over-eager codifications, fossil instructions, premature abstractions in skill files — actively make future sessions worse.** A skill edit that "feels good" in the moment but doesn't capture a recurring pattern becomes a stale instruction that biases future agents incorrectly.

The right outcome on most iterations is: "looked at the iteration's friction signals; nothing rose to the bar of being worth codifying; proceeding."

## Quality gate

A finding clears the bar only if **all three** are true:

1. **Recurring** — would a future session benefit from this guidance, or is it a one-time discovery? One-time learnings go in code comments or commit messages, not in skills or memory.
2. **Codifiable** — can this be expressed as a durable artifact (a feedback memory, a skill section, a script, a repo issue)? Vague "this was annoying" doesn't qualify; a concrete remedy does.
3. **Worth its weight** — would adding this to the relevant skill / memory / repo make agents act *differently* in the future, or is it noise that another reader would skim past? Skills are read on every load; every line costs attention.

If a finding fails any of the three, drop it. **Do not** force-fit it into a skill edit just because the retro was invoked.

If no finding clears all three, the retro produces no actionable output. Report "no actionable findings this round" and return.

## Action item types

Open-ended; pick the one that fits each finding:

- **Memory entries** — feedback / project / reference / user types per the auto-memory schema. Best for: stable preferences, decisions with long horizons, pointers to external resources.
- **Skill edits** — modifications to `loop-driver`, the three loop specializations, `/ship`, `/retro` itself, or any other skill that was active. Best for: workflow improvements that recur across iterations or sessions.
- **Scripts** — small CLI utilities or one-liners committed to the repo (or to `~/.claude/scripts/`) that automate a friction point. Best for: repeated multi-step shell sequences.
- **Constellation issues** — feature requests filed on **any repo in the constellation**, not just the loop driver's target: aios, eumemic-ops, ant-proxy, autodev, dev-skills, aios-web. Best for: missing dev-infrastructure that, if it existed, would have made this iteration (or future iterations) faster. Examples worth filing:
  - autodev pipeline gaps surfaced during a real run (retry CLI, label hygiene, forward-reference handling).
  - aios primitives that would simplify common agent workflows (memory durability, attachment validation, etc.).
  - eumemic-ops audit checks for newly observed invariants.
  - Missing skills or skill clarifications surfaced by friction in this session.

The action item doesn't have to be in the repo you're working in. **"Could this have been easier if upstream X were different?" is always a legitimate question.**

A single retro can produce a mix. Most retros produce zero. Some produce one. A retro producing three or more is suspicious — apply the quality gate harder.

## Phase 1 — Scope the analysis

Identify the time window for this retro:
- `--scope=iteration` — events since the current branch was created. Read with `git log <default>..HEAD` to bound it temporally; use the conversation since the matching `/shovel-ready` / `/kaizen` / `/bughunt` Phase 2 invocation as the conversation slice.
- `--scope=session` — the whole conversation history visible.

If invoked from a loop driver, also note **which skills were involved** in this iteration (loop specialization + `/ship` + any sub-skills). Action items will be attributed to those skills.

## Phase 2 — Read existing artifacts

Before proposing changes, read what's already there:
- The active skills' SKILL.md files (so a proposed edit doesn't duplicate or contradict existing content).
- `MEMORY.md` and the relevant memory entries (so a proposed memory doesn't duplicate).
- Recent merged PRs in the target repo (so a proposed issue isn't already filed or solved).

Skip this step if the analysis window has produced no friction signals — Phase 3 will exit early anyway.

## Phase 3 — Identify candidate findings

Scan the analysis window for:

- **User corrections** — places where the user redirected the agent ("don't do X", "stop doing Y", "use Z instead"). High signal: the user is actively communicating a preference.
- **User confirmations of non-obvious choices** — places where the agent did something unusual and the user accepted it without pushback. Lower volume but equally valuable; failed retros only mine corrections, drift on confirmations.
- **Recurring friction** — the same problem hit two or three times in the window. Once is a fluke; twice is a pattern.
- **Discoverability gaps** — skills that should have triggered but didn't, or commands the agent didn't know existed.
- **Wasted work** — investigations that could have been short-circuited by a tool, script, or piece of context the agent didn't have.
- **Infra papercuts** — things that worked but were rougher than they needed to be: a CLI gap, a missing audit check, an autodev pipeline phase that demanded manual recovery, an aios feature that *would* exist in a more-mature constellation. The kind of "I had to do X manually but a script/CLI/feature should have done it" friction. **These are the highest-leverage findings for `--scope=pause` retros**, since they accumulate across sessions before being captured.

Each candidate finding gets a one-line summary and a proposed action item type. **Don't fix anything yet.**

## Phase 4 — Apply the quality gate

For each candidate, check the three conditions:
- Recurring? (Would a future session benefit?)
- Codifiable? (Can this be a concrete artifact?)
- Worth its weight? (Would the artifact change behavior?)

Drop everything that fails. **It is correct for Phase 4 to drop everything in many retros — even most retros.**

If after gating no findings remain:

```
No actionable findings this round.
```

Return. (For iteration-mode invocations, this is the success path; the loop driver continues to merge.)

## Phase 5 — Present surviving findings for approval

If 1+ findings survived Phase 4, present a short structured summary via `AskUserQuestion`:

```
Retro found N action items worth codifying:

1. [Type] [Target] — [one-line description]
   Rationale: [why this clears the bar]

2. ...

Approve all / approve subset / discard all?
```

Default to "approve subset" semantics — let the user opt in per item if there are multiple. **Don't auto-execute** even high-confidence findings; the user is the final filter.

## Phase 6 — Execute approved items

Per item:

- **Memory entries**: write the file under `~/.claude/projects/<project>/memory/` and update `MEMORY.md`.
- **Skill edits**: edit the relevant SKILL.md. If invoked from a loop driver and the skill repo is the same as the current branch's repo, commit and push to the existing PR. Otherwise, the user can review the diff in the skill repo separately.
- **Scripts**: create the file with executable permissions; if it belongs to the current branch's repo, commit it; otherwise stash under `~/.claude/scripts/` (or wherever the user keeps personal automation).
- **Repo issues**: `gh issue create` on the target repo with a descriptive body; surface the issue URL.

After execution, report what was done. For iteration-mode invocations, return control to the loop driver so it can wait for CI re-run (if a commit was added to the current PR) and proceed to its Phase 6 (merge).

## Phase 7 — Surface dropped findings (optional, sparingly)

If you dropped 3+ candidates in Phase 4, consider mentioning them as a brief footnote ("3 candidate findings did not clear the quality gate: <one-liner each>"). This is **optional** and only useful when the volume of dropped candidates suggests a calibration issue worth the user noticing. Don't list dropped findings to look thorough.

For iteration-mode retros, skip Phase 7 unless the user has explicitly asked for verbose retros — the loop driver wants minimal narration.

## Boundaries

- **Don't write skill edits that are session-specific.** "When working on aios this week, X" is a memory entry, not a skill edit.
- **Don't propose tracking issues on the *skill* repos** (dev-skills, etc.) for things that should be skill edits. If the change belongs in a skill, edit the skill. (Constellation-issue filings on dev-skills are still legitimate for tooling gaps in the skill *infrastructure*, e.g., skill-discovery, plugin loading — not for "rewrite this skill.")
- **Don't expand the analysis window.** Iteration mode means iteration; pause mode means since-last-retro; session mode means whole session.
- **Don't skip the quality gate even when the user said "run a retro".** The user expects a retrospective; the user does NOT necessarily expect changes. "Nothing actionable this round" is a complete and successful retro.
- **Don't argue with rejections.** If the user rejects a Phase 5 proposal, the finding is dropped. Don't re-propose it later in the same session.
- **Don't auto-invoke `--scope=pause` more than once per ~5 user messages of substantive work.** Pause-mode is most valuable when meaningful events have accumulated; running it on every brief wait turns it into noise.

## When to escalate

`AskUserQuestion` mid-retro when:
- A finding is high-impact but borderline on the quality gate (the user's call on whether to codify).
- A proposed skill edit would conflict with another skill's existing guidance.
- A proposed memory would contradict an existing memory entry that's still relevant — confirm intent before overwriting.

When in doubt about whether a finding clears the bar, **drop it.** Wrong skill edits compound; missed retros don't.

## Reference files

- **`references/skill-criteria.md`** — what skills ARE and ARE NOT for, with examples. Useful when deciding between a skill edit and a memory entry.
- **`references/analysis-framework.md`** — detailed candidate-finding categories and proposal templates. Useful for session-mode retros that want a deeper structure than Phase 3 provides.
