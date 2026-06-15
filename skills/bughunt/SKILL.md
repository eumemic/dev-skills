---
name: bughunt
description: One iteration of speculative bug-hunting — audit codebase against project invariants and risk patterns via parallel agents, form hypothesis, prove with failing test, fix at root cause, hand off to /ship, run /retro, merge, declare the outcome. Use when invoked as `/bughunt` (runs one iteration); use `/loop bughunt` to run it as an autonomous loop. Pass `--auto` to merge automatically once CI is green; default is to wait for the user to merge in the GitHub UI. Project-agnostic; reads risk dimensions from CLAUDE.md key-invariants and project memory.
---

# /bughunt — one hypothesis-driven bug-discovery iteration

You run **one iteration**: scan the codebase for risk patterns, form a specific bug hypothesis, prove it with a red test, fix at root cause, ship, retro, merge (or hand to the user), and declare the outcome. `/loop bughunt` runs this on an autonomous loop.

This skill is a thin specialization over the shared **build-cycle** skeleton. **Read `dev-skills:build-cycle` for the shared iteration phases (branching, /ship handoff, /retro, merge, declaring the outcome).** Looping — cadence, idle escalation, re-arming — is owned by `/loop`. This file owns:

- **Phase 1** — target acquisition (parallel-agent risk-pattern audit and hypothesis formation).
- **Phase 1.5** — the bughunt-specific quality gate (provability + reachability + impact).
- **Phase 3** — the bughunt-specific implementation discipline (red→green TDD, root-cause fix).

Everything else — flags, branching, `/ship` invocation, `/retro` pass, merge handling, and the closing `LOOP-OUTCOME` — lives in `build-cycle`; the loop that re-runs this iteration lives in `/loop`.

## Specialization-specific invariants

(See build-cycle for the shared invariants. These are *additional* to those.)

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
11. **Test-coverage gaps** — boundary cases not tested; mocks that diverge from real-system behavior (a unit test that mocks the very function whose logic is wrong passes forever and hides the bug — check that the layer asserting a behavior actually exercises the code that implements it).
12. **Recently-merged feature code** — `git log` the last ~10–15 `feat:` commits and audit each newly-added code path for feature-logic errors: wrong field defaults, validation gaps, account-scoping omitted on a new endpoint, request/response-model drift, a partial / "stage N" migration leaving a mixed encode-decode state, a stale assumption a later commit silently falsified. Have the finder `git show` the actual merge commit, not just the file. **On a mature, heavily-swept codebase this is usually the highest-yield dimension** — the invariant/risk-pattern dimensions above are the most-audited code in the tree, so the freshest un-audited surface is the recent diff and the peripheral subsystems (connectors, CLI, MCP, sandbox internals) that get less bughunt attention than the core.
13. **Sibling-path guard gaps** — when a validation, guard, constraint, state-transition helper, or contract is enforced at one entry-point, enumerate the **sibling** entry-points that reach the same downstream and check whether each enforces it too. The guard's own existence proves the invariant matters; the bug is the path that forgot it. High-recurrence shapes: a request-bound applied to a query param but **not** the opaque-cursor / forged-token path that carries the same value into the same SQL; a "clean up then fail" helper called on most terminal paths but bypassed by one early-return; a cap / advisory-lock taken by the granular add path but not the bulk-replace path; account-scoping applied on `attach` but not on a sibling `bind`. Method: grep for the guard/helper/constant, then read **every** caller and every sibling that *should* route through it but open-codes a partial version instead. This pairs naturally with dimension 12 — a guard added in a recent commit is the prime candidate to have missed a sibling.

### Each agent's prompt should include

- The relevant `CLAUDE.md` invariants and `feedback_*` memory excerpts.
- A reachability filter: don't surface hypotheses that require pathological inputs no real call path produces.
- A provability filter: each hypothesis must come with a sketch of "what test would prove it" — if the agent can't sketch the test, drop the hypothesis.
- A length cap (under 400 words per response).
- A clear deliverable: ranked list with `(reachability × severity × provability) ÷ fix-scope` scores, file:line references, and the test sketch per finding.

Use `subagent_type: general-purpose`. Run all agents parallel via a single message with multiple `Agent` tool calls. If invoked with `--model=<value>` (see build-cycle flags), include `model: "<value>"` on each `Agent` call so every audit subagent runs on the chosen Claude generation; otherwise omit `model:` and let them inherit the session's model.

### Synthesize

Collect ranked lists. Cross-agent corroboration amplifies score. **Provability is a hard filter** at this stage — anything below ~0.6 doesn't make the synthesis cut. Speculation without a path to a red test is wasted iteration.

**If the invariant/risk-pattern dimensions (1–11) come up empty on a mature codebase, pivot to the recently-merged-feature surface (dimension 12) and the peripheral subsystems before declaring `empty`.** An empty invariant sweep on a heavily-audited core is a true negative *about the wrong surface*, not about the codebase — the bug, if there is one, is most likely in code merged since the last sweep. Run a second audit round targeted there (have finders surface their single strongest lead even when sub-threshold, so the synthesis sees near-misses rather than a bare empty set) before you conclude there's nothing to ship.

If the audit — after that pivot — produces no hypothesis with provability ≥ 0.6, declare `LOOP-OUTCOME: empty` (build-cycle Phase 7) and end the iteration.

## Phase 1.5 — Bughunt quality gate

Apply build-cycle's quality-gate principle. For bughunt, the bar is:

A candidate clears the gate only if **all** are true:

1. **Provable** — provability ≥ 0.6 (already filtered in Phase 1, but re-check on the chosen candidate after reading the actual code).
2. **Reachable** — the input that triggers the bug is producible by some real call path (user action, scheduled job, API request). "Theoretically possible if every guard fails" doesn't qualify.
3. **Material** — the symptom matters. Data corruption, lost messages, security issues, user-visible incorrect behavior, silent failures masking real errors — all material. A typo in a log message that no one will ever read is not.
4. **Single-issue scope** — the fix touches one root cause, not "while we're here, fix these adjacent things."

If a candidate is borderline ("input feels reachable but I'd need to confirm"), present 2–3 candidates to the user via `AskUserQuestion` with reachability evidence for each.

If no candidate clears the gate: report what was found and why each was rejected, then declare `LOOP-OUTCOME: gate_killed`. Don't ship a speculative fix to a theoretically-reachable but practically-impossible bug.

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

When invoking `dev-skills:ship` (build-cycle Phase 4):
- `--commit-type=fix` (almost always; `refactor` only if the fix is structural with no observable behavior delta).
- `--issue=<N>` if the bug was already filed (search first: `gh issue list --search "<key symptom>"`).

## Specialization-specific boundaries

(Additive to build-cycle's shared boundaries.)

- **Don't claim a bug without a red test.**
- **Don't over-instrument tests.** Smallest input, sharpest assertion.
- **Don't open the PR while the test is still red.**

## Specialization-specific escalation

`AskUserQuestion` when (in addition to build-cycle's shared triggers):
- Three candidates this iteration produced no red test (audit mis-calibration; switch dimensions or stop).
- The bug crosses architectural lines and the fix could legitimately live in N places.
- The bug is "by design" per a comment / memory entry — confirm before fixing.
- A reviewer challenges the *correctness* of the fix at sev ≥ 80 mid-`/ship`. (Possible deeper root cause; revisit Phase 3 fix step.)

When in doubt, ask. A wrong fix is worse than a missed bug.
