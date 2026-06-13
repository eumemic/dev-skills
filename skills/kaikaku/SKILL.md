---
name: kaikaku
description: Continuous architectural simplification and correctness-by-construction — whole-system recon via parallel agents to find high-leverage opportunities to simplify-and-beautify a structure OR foreclose a whole defect-class by making an illegal state unrepresentable, at the module/layer/concept altitude; steelman each candidate against Chesterton's Fence, then file groomed design-issue(s) for every candidate that clears the gate — autonomously, labeled by validation (no build, no in-session approval gate). Use when invoked as `/kaikaku` (runs one iteration); use `/loop kaikaku` to run it as an autonomous loop. The architectural-altitude sibling of `/kaizen`. Project-agnostic; reads simplicity + beauty + correctness stance from CLAUDE.md and project memory.
---

# /kaikaku — one architectural-simplification iteration

Kaizen (改善) and kaikaku (改革) are the two halves of the lean-improvement vocabulary. Kaizen is *continuous incremental improvement* — many small, local, low-risk fixes; that's `/kaizen`. Kaikaku is *radical, transformative change* — rethinking a structure so a whole **class** dissolves. This is that loop, at architectural altitude.

The class that dissolves can be **complexity** (an over-built structure simplified away) or a **defect-class** (an illegal state made *impossible to write* — the [poka-yoke](https://en.wikipedia.org/wiki/Poka-yoke) / mistake-proofing move). Kaikaku owns **both** architectural cells: simplify-and-beautify *and* correctness-by-construction. They are one recon stream, not two skills — the unifying test is **leverage** (a whole class goes away), *not* whether the diff is net-shorter. A correctness foreclosure that *adds* a sum type or a constraint still clears the bar if it makes a class of bug unrepresentable. (If a finding is a single instance to fix, it's a `/bughunt` target; if it's a few lines to tidy, it's `/kaizen`. Kaikaku is the structural, whole-class altitude for either axis.)

You run **one iteration**: perform whole-system architectural reconnaissance, find the highest-**leverage** opportunity to simplify *and* beautify the architecture (or to foreclose a defect-class), **steelman** it against the possibility that the current shape is load-bearing — the complexity is earned, or the defect-class isn't actually reachable — and, if it survives — author and file a groomed **design-issue** for **every** candidate that clears the gate, then declare the outcome. Kaikaku **does not build** and **does not stop for approval** — it files autonomously: its deliverables are decision-ready proposals, not PRs, and a round routinely yields more than one. `/loop kaikaku` runs this on an autonomous (long-cadence) loop.

Kaikaku is a `/loop`-driven iteration, but it does **not** use the `build-cycle` skeleton — it grooms a proposal instead of building, so it has no branch / implement / `/ship` / merge. It owns its whole iteration and ends by declaring its `LOOP-OUTCOME` to `/loop`. This file owns:

- **Phase 1** — architectural reconnaissance (parallel-agent whole-system recon + adversarial steelman).
- **Phase 1.5** — the kaikaku-specific quality gate (survives the fence; high leverage; uncontested decision-readiness).
- **Phase 4′** — author and file the design-issue(s) (kaikaku's whole build-equivalent — there is no branch/implement/ship/merge).
- **Phase 7** — declare the `LOOP-OUTCOME` (`filed` / `empty` / `gate_killed` / `blocked`) that `/loop` paces on.

## Why kaikaku doesn't use the build-cycle skeleton

`/kaizen` and `/bughunt` plug a target-acquisition step into `build-cycle`'s build spine and ship a PR every productive iteration. Kaikaku changes three things that ripple through the whole iteration:

| | build loops (kaizen/bughunt) | kaikaku |
|---|---|---|
| **Altitude** | lines, functions, single files | modules, layers, data models, seams, concepts |
| **Deliverable** | a merged PR | groomed design-issue(s), filed autonomously (the human/fleet builds them) |
| **Detection** | grep + pattern audit | comprehension — trace dependencies, map a concept's representations |
| **Gate** | "is this accidental complexity?" | "does the complexity survive a Chesterton's-Fence steelman?" |
| **Cadence** | one an hour | one a week — "nothing worth proposing" is the *normal* outcome |

If a finding could be fixed by extracting a helper or deleting a few lines, **it's a `/kaizen` candidate, not a kaikaku one.** Route it there and move on.

## Specialization-specific invariants

(The shared iteration invariants — one target at a time, no scope expansion, fail hard, quality-gate-beats-throughput — apply here too; see `build-cycle`. These are *additional*.)

- **Architectural altitude only.** A valid candidate changes a *structure*: it relocates a responsibility to its right layer, unifies how a concept is represented, redraws a dependency boundary, replaces an invariant-held-by-vigilance with a correct-by-construction mechanism, collapses a speculative extension point, or gives a smeared concept a single home. The payoff is that a whole class dissolves — of downstream complexity, or of latent defect (an illegal state made unrepresentable) — not that one file got shorter.
- **Chesterton's Fence is the core discipline.** Architectural complexity usually *earns its keep*. Every candidate MUST survive an adversarial steelman that reads the actual code, comments, and issue-history to find the reason the current shape exists. The bar is: *"I tried hard to justify keeping it and couldn't."* A false "simplification" that proposes tearing out correct complexity is the worst possible output — worse than an empty iteration.
- **Groom, don't build.** Kaikaku's job ends at decision-ready proposals, filed. The human triages and directs on the tracker; the fleet/human implements (this mirrors the `/shovel-ready` grooming model and the dispatch boundary). Do not open implementation PRs from this loop.
- **Simplify *and* beautify.** "Beautiful" is a real, weighted criterion, not decoration: conceptual integrity (one concept → one representation), orthogonality (independent things independent), correct-by-construction (invariants enforced by types/structure, not vigilance), symmetry (parallel concepts get parallel structure), minimal concept count, legibility (the code reveals the design). A change that's LOC-neutral but restores symmetry or makes an invariant structural can still clear the gate on leverage alone.
- **File every candidate that clears the gate (multi-output).** Recon usually surfaces several real opportunities; groom and file *all* that survive the fence — don't throw away validated runners-up or make the user approve them one at a time. Each gets its own issue, labeled by its own steelman outcome. The per-candidate quality gate (Phase 1.5), not an artificial per-iteration cap, is the throttle. (One iteration can legitimately file zero, one, or several issues.)
- **Respect the project's stance.** Read `CLAUDE.md` (especially "Architecture", "Key invariants", "How to approach changes") and any `project_*` / `feedback_*` memory before recon. The project's named invariants and stated architecture are the highest-signal map of where complexity is *earned* vs *accidental*.

## Phase 1 — Architectural reconnaissance

Comprehend the architecture, don't grep it. Run a **whole-system recon** as parallel agents along architectural **axes** (axes, not subsystems — each agent sees cross-cutting patterns), then an adversarial **steelman** pass on the top candidates. Under ultracode, author this as a single `Workflow` (recon fan-out → rank → steelman pipeline); otherwise dispatch parallel `Agent` calls and run the steelman as a second round.

### Read the architecture stance first

Read `CLAUDE.md`'s Architecture + Key-invariants sections and the relevant `project_*` memory. Pass the load-bearing-complexity examples into every agent's prompt as Chesterton's-Fence calibration (e.g. for aios: the two-pool split, the credential-free workflow host, the surface-attenuation lattice, monotonic context). The agents must know what *earned* complexity looks like in *this* system before they propose removing any.

### The five recon axes

1. **Layering & dependency integrity** — dependency direction across layers; boundary violations (lower layer importing upper; a service reaching into query internals; routers holding business logic); circular/near-circular imports and the lazy-import shims that paper over them (the cycle is the smell); god-modules imported by ~everything; reach-arounds (a seam everyone bypasses).
2. **Concept multiplicity / missing homes** — enumerate the core domain concepts; find ones represented multiple ways or carried by parallel-but-divergent machinery (the prime example: a duality whose two halves are maintained by copy-paste-with-drift). A concept smeared across many files with no single home wants one home or one parameterized representation.
3. **Correctness by construction — invariants held by vigilance, illegal states left representable.** This is the *defect-class* lens (the poka-yoke axis). For each invariant the project names, determine whether it's enforced *structurally* (a type, a single chokepoint, a DB constraint, a single-source generation) or by *developer vigilance* ("this MUST come before that"; "mirrors the logic in <other file>"; "keep these N predicates byte-for-byte in sync"). Then hunt the duals the project *hasn't* named: **illegal states a type still admits** (optional fields that must co-occur; primitive obsession admitting out-of-range values; a loose `dict`/`Any` where a pinned type belongs); **validate-don't-parse boundaries** (data crossing as a loose shape and re-checked ad-hoc at each use, so one path can skip the check); **stale derived state** (caches / denormalized columns / "echo" copies with nothing structurally tying them to their source). The fix proposes the chokepoint / type / constraint / parse-at-the-boundary / single-source generation that makes the bad state **impossible to write** — and it may *add* structure (a sum type, a smart constructor, a constraint) even when that isn't net-simpler, **provided it dissolves a whole class** (correctness leverage, not LOC). Recurring `/kaizen` or `/bughunt` patches of the same shape across N sites are a *symptom* — find the structural fix. (See the anti-guard gate in Phase 1.5: a *foreclosure* makes the state unrepresentable; a *guard* merely re-checks a state that stays writable — the latter is the belt-and-suspenders the project forbids, not a kaikaku candidate.)
4. **Leaky abstractions & speculative generality** — abstractions everyone reaches around (seam in the wrong place); extension points / flags / parameters used 0–1× (collapse to the concrete case); impedance mismatches (a representation converted back and forth) where one canonical form removes a class of translation code.
5. **Broken symmetry & vocabulary** — parallel concepts that lack parallel structure (so understanding doesn't transfer); per-resource modules that drift; domain vocabulary where one concept has several names or one name means two things. Either unify behind one spine, or make the divergence intentional and named.

### Each recon agent's prompt should include

- The CLAUDE.md architecture/invariants excerpts and the project's *earned-complexity* examples (Chesterton calibration).
- The **altitude test**: "if this could be fixed by extracting a helper or deleting a few lines, it's too low — discard it."
- A mandatory `steelman_for_status_quo` field per observation (the strongest honest case the current shape is load-bearing — for a correctness candidate, that the defect-class isn't actually reachable — grounded in the code).
- A rubric: rank by `(architectural_leverage × confidence) ÷ (blast × risk)`; return the top 2–4; quality over quantity; an empty list is a valid honest result.
- A structured deliverable per observation: `title, locus, current_shape, radiated_complexity, proposed_shape, beauty_gains[], steelman_for_status_quo, blast_radius, migration_cost, risk, architectural_leverage, confidence` — plus, for a correctness/defect-class candidate, `reachability_evidence` (live instance | historical drift | concrete future-edit path) and `foreclosure_not_guard` (the type/chokepoint/constraint that makes the illegal state unrepresentable, and why it passes the unaware-developer test rather than merely re-checking a still-writable state).

Pass `--model=<value>` through to every agent if it was passed (see `build-cycle` flags); otherwise omit `model:` and inherit the session's model.

### Steelman the top candidates (the fence gate)

For the top ~5 ranked observations, spawn an **adversarial fence-defender** per candidate whose job is to *prove the current shape is load-bearing*: read the actual code, comments, docstrings, and any issue references at the locus to find why it exists. For a **complexity** candidate that means proving the complexity earns its keep. For a **correctness** candidate it means the inverse — proving the defect-class is *not actually reachable* (already prevented elsewhere, or an illegal state no real path produces), so the foreclosure would be unnecessary belt-and-suspenders. A correctness candidate therefore only survives if it carries **reachability evidence**: a live instance, historical drift (the invariant has already been violated and patched — git archaeology), or a concrete edit a plausible future change makes that introduces an instance. Each returns a verdict — `load_bearing` (kill: the complexity is earned, or the defect-class isn't reachable), `partially_removable` (narrow), or `genuinely_removable` (survives) — plus the strongest counter-argument, hidden migration costs, and a refined framing. Only candidates that **survive their own steelman** advance.

## Phase 1.5 — kaikaku quality gate

Apply the quality-gate principle (don't lower the bar to file *something* — non-productive is a normal, healthy outcome; see `/loop`'s contract). Apply the gate to **each** surviving candidate independently and **file every one that clears it**. A candidate clears only if **all** are true:

1. **Survives the fence.** The steelman verdict is `genuinely_removable` or `partially_removable` with a real simplification left after narrowing. `load_bearing` fails.
2. **High leverage.** The change dissolves a *class* of downstream complexity (or makes a load-bearing invariant structural) — not a one-off tidy. If the only payoff is a handful of fewer lines, it's a `/kaizen` candidate; fail the gate and route it there.
3. **Decision-ready, not contentious-without-data.** The proposal is concrete enough that a human can approve the *direction* from the issue alone. If the right answer genuinely depends on data you don't have (a benchmark, a usage count, a product decision), the deliverable is a *spike/investigation* framing, not a confident proposal — say so.
4. **Honest migration cost.** The migration plan and blast/risk are stated plainly. An `epic`-blast proposal is allowed (it's a design-issue, not a PR) but must say so.
5. **Foreclosure, not a guard** *(for any candidate whose payoff is correctness — making a defect-class impossible)*. The proposed mechanism must make the illegal state **unrepresentable**, not merely re-checked. Apply the **unaware-developer test**: *after this change, every edit a future developer could make that reintroduces one instance of the class is caught by the type-checker, the compiler, a DB constraint, or a single-source generation that has no second site to edit.* If such an edit could still compile / type-check / pass CI, the proposal is a **runtime guard against a state that stays writable** — the belt-and-suspenders the project forbids; fail the gate (it's a `/bughunt` instance-fix at best). The test keys on *where the check sits relative to the write path*, not on the word "check": a **build-time / CI drift-guard** that asserts the single source generates all its consumers **passes** (the copies can no longer be authored independently); a runtime validator that ships even when wrong **fails**. A live bug found while proving the class is reachable is *evidence* for the issue, not something to fix in this loop.

If nothing clears the gate: report what was found and why each was rejected (especially fence kills — those are valuable architectural *confirmations* that the complexity is earned), then declare `LOOP-OUTCOME: gate_killed` (a non-productive iteration; `/loop` handles idle/cadence). Filing a weak architectural issue is worse than filing none — it erodes trust in the queue.

## Phase 4′ — Author and file the design-issue(s)

This is kaikaku's whole build-equivalent — there is no branch/implement/ship/merge. The deliverables are groomed architectural design-issues — **one per candidate that clears the gate**, filed autonomously.

### Write the proposal

Structure the issue body as a steelmanned RFC:

- **Current shape** — what the architecture does now, with the concrete loci (`path:concept`).
- **Radiated complexity** — the downstream cost this shape forces across the system (the *why-this-matters*; quantify with site counts / file spread where you can).
- **Proposed shape** — the simpler/more-beautiful architecture, concretely.
- **Beauty / simplicity / correctness gains** — which of conceptual-integrity / orthogonality / correct-by-construction / symmetry / minimal-concepts / legibility this buys; for a defect-class foreclosure, name the class of bug that becomes *unrepresentable*.
- **Steelman (Chesterton's Fence)** — the strongest case for the status quo, and why the proposal survives it (or how it was narrowed). This section is mandatory — it's what distinguishes a kaikaku proposal from a naive rewrite. For a correctness foreclosure it also carries the **reachability evidence** (the live instance / historical drift / future-edit path that proves the defect-class is real, not theoretical) and the **foreclosure-not-guard** rationale (why the fix makes the illegal state unrepresentable — it passes the unaware-developer test — rather than re-checking a state that stays writable).
- **Migration plan** — the staged path (expand/contract, stacked PRs, data backfill if any), and what stays back-compatible across the window.
- **Blast radius & risk** — files/modules touched, the reversibility, and the failure modes.
- **Open questions** — anything that needs a human/product call before implementation.

### Label by the steelman's outcome, not by default

The deliverable's label depends on **how the fence pass resolved**, not a fixed default. Don't reflexively stamp every proposal `needs-design`:

- **Steelman *converged*** — it landed on a concrete plan with no open architectural forks (a standard/obvious mechanism, a staged migration, and only *empirical assumptions* left to check). This is a groomed **spec**, not a design question. **Verify the steelman's key empirical assumptions yourself first** (e.g. "no non-pool connection reads this type" — go confirm it), then label it `shovel-ready` (or the repo's build-queue label). It can enter the build queue; a human glance is welcome but not a gate.
- **Steelman left *open forks*** — it narrowed the candidate but a real decision/trade-off remains (`partially_removable` with a genuine choice of shape, competing migrations, or a load-bearing concern you couldn't fully resolve). Label it `needs-design` and gate on human direction. Frame the open fork explicitly in the issue.

The failure mode to avoid: labeling a verified, converged, standard-pattern change `needs-design` and parking it behind a human review it doesn't need. (Precedent: the first kaikaku run filed the JSONB-codec proposal `needs-design`, but its steelman had fully converged — the only open item was verifying the pool-scope assumption; once verified it was reclassified `shovel-ready`.)

### File autonomously — no approval gate

Kaikaku files without stopping for in-session approval; it must be able to run unattended. For **every** candidate that clears Phase 1.5, file an issue (`gh issue create`) labeled by its own steelman outcome (see below). The steelman gate — fence + anti-guard + reachability, applied per candidate — is what makes unattended filing safe; the human reviews the filed issues on the tracker, like PRs. Don't present a draft for sign-off, don't pick one and discard the rest, and don't ask which to file: file them all, labeled. Before filing, dup-check the tracker (`gh issue list --search`) and record each filed target so the next recon excludes it. Report the filed URLs and the count.

## Phase 7 — Declare the outcome

End the iteration with the `LOOP-OUTCOME` line; `/loop` owns everything after.

- `filed` — one or more design-issues were filed (Phase 4′; report the count and URLs). `/loop` treats it as productive and re-arms **long** (filed proposals await human review on the tracker; there's no fast-follow). This is the architectural cadence — nothing to do at 60s.
- `gate_killed` — recon found candidates but none survived the fence + gate (Phase 1.5). Non-productive; `/loop` idles. A fence-kill is a valuable *confirmation* that the complexity is earned — report it.
- `empty` — recon surfaced nothing at architectural altitude. Non-productive.
- `blocked` — a genuine inability to proceed (e.g. the issue tracker is unreachable). Kaikaku does **not** block for approval or to choose among candidates — it files autonomously and labels human-decision cases `needs-design` (see Escalation). `blocked` should be rare.

Autonomous filing is the **default** — there is no approval mode to opt out of, so `--auto` is redundant (accepted as a harmless no-op for back-compat / `/loop` passthrough; it never meant "merge" — there is no merge). The loop flags (`--once`, `--idle-count`, `--drain`) are `/loop`'s.

**The per-candidate quality gate is the throttle — not a per-iteration cap, not a human approval step.** File everything that clears it; withholding a validated candidate to avoid "filing too much" is the throw-away-work failure mode. But the bar is real and stays high: don't file a candidate that fails the fence / anti-guard / reachability test just to produce output, and never re-surface a pending or rejected target. If `needs-design` proposals genuinely pile up unreviewed, that's the *human's* cue to triage the tracker — not kaikaku's cue to stop surfacing validated structural problems.

## Boundaries

(Additive to build-cycle's shared boundaries.)

- **Don't build.** No implementation PRs from this loop. If a surviving candidate is small and contained enough to *want* to build, it was probably a `/kaizen` target — route it there.
- **Don't propose a rewrite as a simplification.** "Rewrite subsystem X" is not a kaikaku proposal unless the steelman shows the current shape is genuinely accidental *and* the migration is staged and bounded.
- **Don't re-surface a pending or rejected proposal.** Track filed/rejected targets so recon excludes them until they're resolved.
- **Don't speculate on future flexibility.** Kaikaku proposes present-tense structural integrity, not extensibility for hypothetical futures (that's the speculative-generality smell it *removes*).

## Escalation — label, don't block

Kaikaku runs unattended, so it does **not** stop to ask the user. The cases that would once have been an in-session question become a **label + explicit framing in the issue**, for the human to resolve on the tracker:

- A candidate's steelman is genuinely split (`partially_removable` with a load-bearing core), or the proposal touches a public API / wire format / data model other systems depend on → file it `needs-design`, with the open fork / blast stated plainly in the Open-questions section. Don't decide it, and don't withhold it.
- Multiple candidates survive → file them all (the multi-output default); there is no "pick one" to escalate.
- Recon keeps surfacing the same fence-kill → that's an earned-complexity confirmation; note it in the run log and move on. Re-confirming the project's documented rationale isn't a per-iteration question.

The only genuine `blocked` is an inability to act at all (tracker unreachable, auth failure). When a candidate is uncertain, the move is **file it `needs-design`** — not ask. A labeled issue the human triages costs less than a stalled loop, and never throws away the work.
