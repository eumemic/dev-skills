---
name: bughunt
description: Continuous speculative bug-hunting — audit codebase against project invariants and risk patterns via parallel agents, form hypothesis, prove with failing test, fix at root cause, hand off to /ship, run /retro, then loop. Use when invoked as `/bughunt`. Loops by default; pass `--once` for a single iteration. Pass `--auto` to merge automatically once CI is green; default is to wait for the user to merge in the GitHub UI. Project-agnostic; reads risk dimensions from CLAUDE.md key-invariants and project memory.
---

# /bughunt — continuous hypothesis-driven bug discovery loop

You're driving an autonomous loop that scans the codebase for risk patterns, forms specific bug hypotheses, proves them with red tests, fixes at root cause, ships, retros, then either merges or waits.

This skill is a thin specialization over the shared loop-driver framework. **Read `dev-skills:loop-driver` for the shared phases (branching, /ship handoff, /retro, merge, idle escalation, flag passthrough).** This file owns:

- **Phase 1** — target acquisition (parallel-agent risk-pattern audit and hypothesis formation).
- **Phase 1.5** — the bughunt-specific quality gate (provability + reachability + impact).
- **Phase 3** — the bughunt-specific implementation discipline (red→green TDD, root-cause fix).

Everything else — flags, branching, `/ship` invocation, `/retro` pass, merge handling, idle escalation — lives in `loop-driver`.

## Specialization-specific invariants

(See loop-driver for the shared invariants. These are *additional* to those.)

- **TDD is mandatory and load-bearing.** No bug ships without a red→green test capturing the exact symptom. The test must fail with the precise error your hypothesis predicts, and the fix must be what flips it.
- **Skip unprovable hypotheses silently.** If you can't write a failing test that captures the bug, the hypothesis is too vague or the bug doesn't exist. Drop it and pick the next candidate. Don't ship "I think there might be a problem here" speculation. Don't open a tracking issue — wrong hypotheses are routine and silent dispatch is cheaper than noise.
- **Root-cause fixes only.** Once the test is red, find and fix the underlying cause — don't add a guard that papers over the symptom. If the only fix is symptom-suppression, the hypothesis is incomplete.
- **Respect the project's invariants.** Read `CLAUDE.md` (especially "Key invariants" / "How to approach changes") and project memory before scoping risk patterns. Project-stated invariants are usually the highest-yield audit dimensions because the project has already named them.

## Phase 1 — Parallel-agent audit

Dispatch **multiple agents in parallel** (single message, multiple `Agent` tool calls), each scanning a distinct risk dimension. Each agent returns ranked candidate hypotheses; you synthesize.

### Read the project's stated invariants first

Read `CLAUDE.md` for "Key invariants", "Invariants", "Correctness rules", or similar. Read project memory for `feedback_*` and `project_*` files mentioning past bugs (e.g., "phase 2 lessons", "incident postmortem"). Past bugs cluster — the same shape often recurs in adjacent code. Pass these excerpts into each agent's prompt.

### Dispatch agents (4–6 dimensions)

The standard set (pick what fits the codebase):

1. **Project-invariant violations** — for each invariant in CLAUDE.md, search for code that fails to maintain it at any of its sites.
2. **Concurrency** — shared mutable state without locks; awaits between read-modify-write of the same row; ordering across async tasks; `gather()` losing partial results on one failure.
3. **Resource lifecycles** — connections/files/locks acquired without paired release; `try` without `finally`; tasks created without registration in the cleanup registry.
4. **Type confusion at `Any` boundaries** — places where mypy infers `Any` (dict accesses on JSON, raw fetches, untyped third-party returns) and downstream code assumes a specific shape.
5. **Error swallowing** — `except Exception:` blocks that don't re-raise; `try/except` around code where a failure should propagate; broad excepts in projects with a "fail hard" stance.
6. **Boundary cases** — empty list / single-element / max-int handling; pagination edges; range loops with computed upper bounds; slice indices.
7. **Stale derived state** — caches not invalidated on write; computed fields not recomputed on dependent change; "echo" patterns drifting from source.
8. **Time and timezone** — naive vs aware datetimes; DST; `now()` called inside a transaction returning a different value than elsewhere.
9. **Query construction** — SQL string interpolation; missing parameterization; case-folding traps.
10. **Idempotency** — retry paths that assume the operation didn't already succeed; webhook handlers without dedup.
11. **Test-coverage gaps** — boundary cases not tested; mocks that diverge from real-system behavior.

### Each agent's prompt should include

- The relevant `CLAUDE.md` invariants and `feedback_*` memory excerpts.
- A reachability filter: don't surface hypotheses that require pathological inputs no real call path produces.
- A provability filter: each hypothesis must come with a sketch of "what test would prove it" — if the agent can't sketch the test, drop the hypothesis.
- A length cap (under 400 words per response).
- A clear deliverable: ranked list with `(reachability × severity × provability) ÷ fix-scope` scores, file:line references, and the test sketch per finding.

Use `subagent_type: general-purpose`. Run all agents parallel via a single message with multiple `Agent` tool calls. If the loop was invoked with `--model=<value>` (see loop-driver flags), include `model: "<value>"` on each `Agent` call so every audit subagent runs on the chosen Claude generation; otherwise omit `model:` and let them inherit the session's model.

### Synthesize

Collect ranked lists. Cross-agent corroboration amplifies score. **Provability is a hard filter** at this stage — anything below ~0.6 doesn't make the synthesis cut. Speculation without a path to a red test is wasted iteration.

If the audit produces no hypothesis with provability ≥ 0.6, fall through to loop-driver Phase 8's empty-iteration branch.

## Phase 1.5 — Bughunt quality gate

Apply the loop-driver's quality-gate principle. For bughunt, the bar is:

A candidate clears the gate only if **all** are true:

1. **Provable** — provability ≥ 0.6 (already filtered in Phase 1, but re-check on the chosen candidate after reading the actual code).
2. **Reachable** — the input that triggers the bug is producible by some real call path (user action, scheduled job, API request). "Theoretically possible if every guard fails" doesn't qualify.
3. **Material** — the symptom matters. Data corruption, lost messages, security issues, user-visible incorrect behavior, silent failures masking real errors — all material. A typo in a log message that no one will ever read is not.
4. **Single-issue scope** — the fix touches one root cause, not "while we're here, fix these adjacent things."

If a candidate is borderline ("input feels reachable but I'd need to confirm"), present 2–3 candidates to the user via `AskUserQuestion` with reachability evidence for each.

If no candidate clears the gate: report what was found and why each was rejected, then proceed to loop-driver Phase 8 as if the iteration were empty. Don't ship a speculative fix to a theoretically-reachable but practically-impossible bug.

## Phase 3 — Implement

### Write the failing test

Write a test that reaches the suspect code path with the input that triggers the hypothesis, and asserts the *correct* outcome. Run it. It must fail.

**Confirm the failure mode is the predicted one.** A test that fails for the wrong reason is worse than no test:
- `ImportError` → test is broken; fix the test.
- Unexpected `KeyError` upstream of the bug → input shape is wrong; adjust until it reaches the suspect line.
- Predicted symptom (assertion comparing actual to expected) → hypothesis confirmed; proceed to fix.

**If the test passes** — the bug doesn't exist (or the hypothesis was wrong). Discard the candidate. **Skip silently** and pick the next one from the synthesis. Don't write a tracking issue, don't surface to the user.

If you've burned through three candidates this iteration without a red test, the audit is mis-calibrated. Surface via `AskUserQuestion`: "Three candidates this iteration didn't produce red tests — keep going at deeper audit, switch dimensions, or stop?"

### Fix at root cause

With the test red for the right reason, find the smallest fix that flips it green without rewriting unrelated logic.

- **Root-cause means root-cause.** Don't catch the bad case downstream of where it originated. The fix should be at the site introducing the invariant violation.
- **Don't expand scope.** Adjacent smells become *next* iteration's `/kaizen` candidates.
- **Don't add defensive guards beyond the fix.** Per "fail hard, no fallbacks" stances, future regressions should remain visible.

Re-run the failing test. Confirm it passes. Run the full test suite to confirm no other tests went red.

If the fix breaks other tests, **investigate before adjusting them.** A test relying on the buggy behavior is itself a bug; fix it properly.

### `/ship` defaults

When invoking `dev-skills:ship` (loop-driver Phase 4):
- `--commit-type=fix` (almost always; `refactor` only if the fix is structural with no observable behavior delta).
- `--issue=<N>` if the bug was already filed (search first: `gh issue list --search "<key symptom>"`).

## Specialization-specific boundaries

(Additive to loop-driver's shared boundaries.)

- **Don't claim a bug without a red test.**
- **Don't over-instrument tests.** Smallest input, sharpest assertion.
- **Don't open the PR while the test is still red.**

## Specialization-specific escalation

`AskUserQuestion` when (in addition to loop-driver's shared triggers):
- Three candidates this iteration produced no red test (audit mis-calibration; switch dimensions or stop).
- The bug crosses architectural lines and the fix could legitimately live in N places.
- The bug is "by design" per a comment / memory entry — confirm before fixing.
- A reviewer challenges the *correctness* of the fix at sev ≥ 80 mid-`/ship`. (Possible deeper root cause; revisit Phase 3 fix step.)

When in doubt, ask. A wrong fix is worse than a missed bug.
