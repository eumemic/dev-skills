---
name: kaizen
description: Continuous accidental-complexity reduction — audit codebase against craftsmanship principles via parallel agents, pick highest-ROI target, simplify, hand off to /ship, run /retro, then loop. Use when invoked as `/kaizen`. Loops by default; pass `--once` for a single iteration. Pass `--auto` to merge automatically once CI is green; default is to wait for the user to merge in the GitHub UI. Project-agnostic; reads simplicity heuristics from CLAUDE.md and project memory.
---

# /kaizen — continuous audit-and-simplify loop

You're driving an autonomous loop that scours the codebase for accidental complexity, ships the highest-ROI simplification, runs `/retro`, then either merges or waits.

This skill is a thin specialization over the shared loop-driver framework. **Read `dev-skills:loop-driver` for the shared phases (branching, /ship handoff, /retro, merge, idle escalation, flag passthrough).** This file owns:

- **Phase 1** — target acquisition (parallel-agent codebase audit).
- **Phase 1.5** — the kaizen-specific quality gate.
- **Phase 3** — the kaizen-specific implementation discipline.

Everything else — flags, branching, `/ship` invocation, `/retro` pass, merge handling, idle escalation — lives in `loop-driver`.

## Specialization-specific invariants

(See loop-driver for the shared invariants — one PR per change, no scope expansion, fail hard, etc. These are *additional* to those.)

- **Accidental complexity only** — defensive guards for model mistakes, fallbacks, retry loops that hide errors, multi-stage corrective pipelines, backwards-compat shims, near-duplicate code paths, state machines encoded across log scans. Don't rewrite legitimately complex domain logic that earns its lines.
- **No new features, no behavior changes** — kaizen is structural. If the audit surfaces a real bug, surface it as a `/bughunt` candidate and move on. Don't fix it under a kaizen PR.
- **No TDD** — there's no behavior change to test for. Existing tests must continue to pass; that's the regression bar.
- **Respect the project's stance** — read `CLAUDE.md` and any `feedback_*` memories before picking. Project-specific anti-patterns weight your search heavily.

## Phase 1 — Parallel-agent audit

Dispatch **multiple agents in parallel** (single message, multiple `Agent` tool calls) covering different dimensions of craftsmanship. Each agent returns ranked findings; you synthesize.

### Read the simplicity stance first

Before launching agents, read `CLAUDE.md` (project root) and any `feedback_*` / `project_*` memory files about complexity, style, simplicity priorities. Pass the relevant excerpts into each agent's prompt so their findings are calibrated to *this* project's bar.

### Dispatch agents

Pick 4–6 dimensions appropriate to the codebase. The standard set:

1. **Project invariants** — read `CLAUDE.md`'s invariants/anti-patterns sections. Find code that maintains the same invariant differently across sites, or violates it subtly.
2. **Structural duplication** — search for near-twin function names (`process_X` + `process_X_v2`, `defer_X` + `defer_retry_X`), multi-site inlined state machines, copy-paste with slight variation.
3. **File-size + smell-words** — top-N largest files; grep for `fallback`, `legacy`, `compat_`, `TODO`, `XXX`, `HACK`, `FIXME`, `workaround`, `recover`, `repair`, `reconcile`. Each match is a candidate, not a verdict.
4. **Broad excepts and error swallowing** — `except Exception:` blocks that don't re-raise; symptom-suppression guards; "fail soft" handlers in places where "fail hard" is the project's stance.
5. **Multi-stage corrective pipelines** — modules whose docstrings advertise "replaces X + Y + Z with a single function". Sometimes load-bearing for crash recovery; sometimes a state machine fighting the architecture.
6. **Craftsmanship principles** — assess against: Clean (dead code, unclear purpose), Simple (over-engineering, clever solutions), Comprehensible (poor naming, high cognitive load), Well-Factored (mixed responsibilities, large functions), Single Responsibility (multi-purpose units), Testable (hard-to-test code, side effects), DRY (missed abstractions).

### Each agent's prompt should include

- The relevant `CLAUDE.md` excerpts and `feedback_*` memory excerpts so calibration matches the project.
- A rubric: rank findings by `(impact × confidence) ÷ blast-radius`, return top 3–5.
- The instruction: "Don't fix anything; just identify and rank."
- A length cap (under 400 words per response) so synthesis stays manageable.
- A clear deliverable: ranked list with file:line references and a one-line rationale per finding.

Use `subagent_type: general-purpose` for the audit agents. They run parallel via a single message with multiple `Agent` tool calls.

### Synthesize

Collect the ranked lists. Merge into a single global ranking — same target appearing across multiple agents amplifies its score. Rank by: cross-agent corroboration first (multiple dimensions flagging it = signal), then per-dimension severity, then blast radius (smaller wins ties).

### Read the top candidate's file in detail

Before committing to a target, **read the actual file** (not just the agent's excerpt). Confirm:
- Does this complexity earn its lines, or is it accidental?
- Is there a near-twin elsewhere that could be unified?
- Does the same logic appear inline in N call sites without a helper?
- Does CLAUDE.md explicitly forbid the pattern?

If the audit produces no candidate at the depth applied, fall through to loop-driver Phase 8's empty-iteration branch.

## Phase 1.5 — Kaizen quality gate

Apply the loop-driver's quality-gate principle. For kaizen, the bar is:

A candidate clears the gate only if **all** are true:

1. **Unambiguously accidental.** The complexity does NOT earn its lines on second read. "Maybe load-bearing for crash recovery" or "might be intentional, hard to tell" fails the gate.
2. **Net deletion expected.** A kaizen PR that adds more lines than it removes is a yellow flag — it's probably an abstraction posing as a simplification. Borderline net-positive cases need explicit justification or they fail the gate.
3. **Not contentious.** If two equally-reasonable people would disagree on whether the code is better after the change, fail the gate. Kaizen ships uncontroversial wins.
4. **Bounded blast radius.** A simplification that touches > ~5 files OR crosses module boundaries fails the gate unless the user has explicitly approved a deeper-cut iteration.

If the top candidate fails the gate but a borderline candidate exists ("might be load-bearing"), present 2–3 options to the user via `AskUserQuestion` with rationale. Don't gamble on a contentious refactor.

If no candidate clears the gate: report what was found and why each was rejected, then proceed to loop-driver Phase 8 as if the iteration were empty (the audit wasn't empty, but the iteration is non-productive — that's still an `--idle-count` increment). Don't ship a borderline change just because the audit produced output.

## Phase 3 — Implement

Make the focused simplification. Keep it tight:

- **Delete more than you add.** A net-positive line count for kaizen is a yellow flag — surface it and justify in the PR body.
- **Don't expand scope mid-implementation.** Adjacent smells become *next* iteration's candidates.
- **Update tests in lockstep.** Fossil tests (signature-equality checks for now-deleted functions) get deleted, not retrofitted.
- **Update doc comments** that reference now-changed code paths.
- **Don't add behavior.** If a "simplification" requires changing what the code does observably, it's not a kaizen — re-route to `/bughunt` (if it's a hidden bug) or back to the user (if it's a feature).

### `/ship` defaults

When invoking `dev-skills:ship` (loop-driver Phase 4):
- `--commit-type=refactor`
- `--issue=<N>` not applicable.

## Specialization-specific boundaries

(Additive to loop-driver's shared boundaries.)

- **Don't pick a target the user has already rejected this session** (a CLOSED PR within memory).
- **Don't deep-refactor on the first iteration** of a fresh codebase audit. Smaller wins build calibration trust.
- **Don't speculate on "future flexibility"** in PR bodies. Kaizen ships present-tense simplicity.

## Specialization-specific escalation

`AskUserQuestion` when (in addition to loop-driver's shared triggers):
- Two top candidates are within ROI noise of each other.
- The target straddles real-vs-accidental complexity (e.g., a corrective pipeline that *might* be needed for crash recovery).

When in doubt, ask. A wrong-target kaizen PR teaches the user to distrust the loop.
