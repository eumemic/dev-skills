---
name: kaikaku
description: Continuous architectural simplification — whole-system recon via parallel agents to find high-leverage simplify-and-beautify opportunities at the module/layer/concept altitude, steelman the top candidate against Chesterton's Fence, then author a groomed architectural design-issue and hand off (no build). Use when invoked as `/kaikaku` (runs one iteration); use `/loop kaikaku` to run it as an autonomous loop. The architectural-altitude sibling of `/kaizen`. Project-agnostic; reads simplicity + beauty stance from CLAUDE.md and project memory.
---

# /kaikaku — one architectural-simplification iteration

Kaizen (改善) and kaikaku (改革) are the two halves of the lean-improvement vocabulary. Kaizen is *continuous incremental improvement* — many small, local, low-risk fixes; that's `/kaizen`. Kaikaku is *radical, transformative change* — rethinking a structure so a whole **class** of complexity dissolves. This is that loop, at architectural altitude.

You run **one iteration**: perform whole-system architectural reconnaissance, find the highest-**leverage** opportunity to simplify *and* beautify the architecture, **steelman** it against the possibility that the complexity is load-bearing, and — if it survives — author a groomed architectural **design-issue** and hand it off, then declare the outcome. Kaikaku **does not build**: its deliverable is a decision-ready proposal, not a PR. `/loop kaikaku` runs this on an autonomous (long-cadence) loop.

Kaikaku is a `/loop`-driven iteration, but it does **not** use the `build-cycle` skeleton — it grooms a proposal instead of building, so it has no branch / implement / `/ship` / merge. It owns its whole iteration and ends by declaring its `LOOP-OUTCOME` to `/loop`. This file owns:

- **Phase 1** — architectural reconnaissance (parallel-agent whole-system recon + adversarial steelman).
- **Phase 1.5** — the kaikaku-specific quality gate (survives the fence; high leverage; uncontested decision-readiness).
- **Phase 4′** — author the design-issue and hand off (kaikaku's whole build-equivalent — there is no branch/implement/ship/merge).
- **Phase 7** — declare the `LOOP-OUTCOME` (`filed` / `empty` / `gate_killed` / `blocked`) that `/loop` paces on.

## Why kaikaku doesn't use the build-cycle skeleton

`/kaizen` and `/bughunt` plug a target-acquisition step into `build-cycle`'s build spine and ship a PR every productive iteration. Kaikaku changes three things that ripple through the whole iteration:

| | build loops (kaizen/bughunt) | kaikaku |
|---|---|---|
| **Altitude** | lines, functions, single files | modules, layers, data models, seams, concepts |
| **Deliverable** | a merged PR | a groomed architectural design-issue (the human/fleet builds it) |
| **Detection** | grep + pattern audit | comprehension — trace dependencies, map a concept's representations |
| **Gate** | "is this accidental complexity?" | "does the complexity survive a Chesterton's-Fence steelman?" |
| **Cadence** | one an hour | one a week — "nothing worth proposing" is the *normal* outcome |

If a finding could be fixed by extracting a helper or deleting a few lines, **it's a `/kaizen` candidate, not a kaikaku one.** Route it there and move on.

## Specialization-specific invariants

(The shared iteration invariants — one target at a time, no scope expansion, fail hard, quality-gate-beats-throughput — apply here too; see `build-cycle`. These are *additional*.)

- **Architectural altitude only.** A valid candidate changes a *structure*: it relocates a responsibility to its right layer, unifies how a concept is represented, redraws a dependency boundary, replaces an invariant-held-by-vigilance with a correct-by-construction mechanism, collapses a speculative extension point, or gives a smeared concept a single home. The payoff is that a whole class of downstream complexity dissolves — not that one file got shorter.
- **Chesterton's Fence is the core discipline.** Architectural complexity usually *earns its keep*. Every candidate MUST survive an adversarial steelman that reads the actual code, comments, and issue-history to find the reason the current shape exists. The bar is: *"I tried hard to justify keeping it and couldn't."* A false "simplification" that proposes tearing out correct complexity is the worst possible output — worse than an empty iteration.
- **Groom, don't build.** Kaikaku's job ends at a decision-ready proposal. The human approves direction; the fleet/human implements (this mirrors the `/shovel-ready` grooming model and the dispatch boundary). Do not open implementation PRs from this loop.
- **Simplify *and* beautify.** "Beautiful" is a real, weighted criterion, not decoration: conceptual integrity (one concept → one representation), orthogonality (independent things independent), correct-by-construction (invariants enforced by types/structure, not vigilance), symmetry (parallel concepts get parallel structure), minimal concept count, legibility (the code reveals the design). A change that's LOC-neutral but restores symmetry or makes an invariant structural can still clear the gate on leverage alone.
- **One proposal per iteration.** Even if recon surfaces two architectural opportunities, groom the single highest-leverage one. The others are next iteration's candidates.
- **Respect the project's stance.** Read `CLAUDE.md` (especially "Architecture", "Key invariants", "How to approach changes") and any `project_*` / `feedback_*` memory before recon. The project's named invariants and stated architecture are the highest-signal map of where complexity is *earned* vs *accidental*.

## Phase 1 — Architectural reconnaissance

Comprehend the architecture, don't grep it. Run a **whole-system recon** as parallel agents along architectural **axes** (axes, not subsystems — each agent sees cross-cutting patterns), then an adversarial **steelman** pass on the top candidates. Under ultracode, author this as a single `Workflow` (recon fan-out → rank → steelman pipeline); otherwise dispatch parallel `Agent` calls and run the steelman as a second round.

### Read the architecture stance first

Read `CLAUDE.md`'s Architecture + Key-invariants sections and the relevant `project_*` memory. Pass the load-bearing-complexity examples into every agent's prompt as Chesterton's-Fence calibration (e.g. for aios: the two-pool split, the credential-free workflow host, the surface-attenuation lattice, monotonic context). The agents must know what *earned* complexity looks like in *this* system before they propose removing any.

### The five recon axes

1. **Layering & dependency integrity** — dependency direction across layers; boundary violations (lower layer importing upper; a service reaching into query internals; routers holding business logic); circular/near-circular imports and the lazy-import shims that paper over them (the cycle is the smell); god-modules imported by ~everything; reach-arounds (a seam everyone bypasses).
2. **Concept multiplicity / missing homes** — enumerate the core domain concepts; find ones represented multiple ways or carried by parallel-but-divergent machinery (the prime example: a duality whose two halves are maintained by copy-paste-with-drift). A concept smeared across many files with no single home wants one home or one parameterized representation.
3. **Invariants held by vigilance** — for each invariant the project names, determine whether it's enforced *structurally* (a type, a single chokepoint, a DB constraint) or by *developer vigilance* ("this MUST come before that"; "mirrors the logic in <other file>"). The vigilance ones are prime targets: propose the chokepoint/type/constraint that makes the invariant impossible to violate. (Recurring kaizen patches of the same invariant across N sites are a *symptom* — find the structural fix.)
4. **Leaky abstractions & speculative generality** — abstractions everyone reaches around (seam in the wrong place); extension points / flags / parameters used 0–1× (collapse to the concrete case); impedance mismatches (a representation converted back and forth) where one canonical form removes a class of translation code.
5. **Broken symmetry & vocabulary** — parallel concepts that lack parallel structure (so understanding doesn't transfer); per-resource modules that drift; domain vocabulary where one concept has several names or one name means two things. Either unify behind one spine, or make the divergence intentional and named.

### Each recon agent's prompt should include

- The CLAUDE.md architecture/invariants excerpts and the project's *earned-complexity* examples (Chesterton calibration).
- The **altitude test**: "if this could be fixed by extracting a helper or deleting a few lines, it's too low — discard it."
- A mandatory `steelman_for_status_quo` field per observation (the strongest honest case the complexity is load-bearing, grounded in the code).
- A rubric: rank by `(architectural_leverage × confidence) ÷ (blast × risk)`; return the top 2–4; quality over quantity; an empty list is a valid honest result.
- A structured deliverable per observation: `title, locus, current_shape, radiated_complexity, proposed_shape, beauty_gains[], steelman_for_status_quo, blast_radius, migration_cost, risk, architectural_leverage, confidence`.

Pass `--model=<value>` through to every agent if it was passed (see `build-cycle` flags); otherwise omit `model:` and inherit the session's model.

### Steelman the top candidates (the fence gate)

For the top ~5 ranked observations, spawn an **adversarial fence-defender** per candidate whose job is to *prove the complexity is load-bearing*: read the actual code, comments, docstrings, and any issue references at the locus to find why the current shape exists. Each returns a verdict — `load_bearing` (kill), `partially_removable` (narrow), or `genuinely_removable` (survives) — plus the strongest counter-argument, hidden migration costs, and a refined framing. Only candidates that **survive their own steelman** advance.

## Phase 1.5 — kaikaku quality gate

Apply the quality-gate principle (resist filing something every iteration — non-productive is a normal, healthy outcome; see `/loop`'s contract). For kaikaku, the surviving top candidate clears the gate only if **all** are true:

1. **Survives the fence.** The steelman verdict is `genuinely_removable` or `partially_removable` with a real simplification left after narrowing. `load_bearing` fails.
2. **High leverage.** The change dissolves a *class* of downstream complexity (or makes a load-bearing invariant structural) — not a one-off tidy. If the only payoff is a handful of fewer lines, it's a `/kaizen` candidate; fail the gate and route it there.
3. **Decision-ready, not contentious-without-data.** The proposal is concrete enough that a human can approve the *direction* from the issue alone. If the right answer genuinely depends on data you don't have (a benchmark, a usage count, a product decision), the deliverable is a *spike/investigation* framing, not a confident proposal — say so.
4. **Honest migration cost.** The migration plan and blast/risk are stated plainly. An `epic`-blast proposal is allowed (it's a design-issue, not a PR) but must say so.

If nothing clears the gate: report what was found and why each was rejected (especially fence kills — those are valuable architectural *confirmations* that the complexity is earned), then declare `LOOP-OUTCOME: gate_killed` (a non-productive iteration; `/loop` handles idle/cadence). Filing a weak architectural issue is worse than filing none — it erodes trust in the queue.

## Phase 4′ — Author the design-issue and hand off

This is kaikaku's whole build-equivalent — there is no branch/implement/ship/merge. The deliverable is a single groomed architectural design-issue.

### Write the proposal

Structure the issue body as a steelmanned RFC:

- **Current shape** — what the architecture does now, with the concrete loci (`path:concept`).
- **Radiated complexity** — the downstream cost this shape forces across the system (the *why-this-matters*; quantify with site counts / file spread where you can).
- **Proposed shape** — the simpler/more-beautiful architecture, concretely.
- **Beauty + simplicity gains** — which of conceptual-integrity / orthogonality / correct-by-construction / symmetry / minimal-concepts / legibility this buys.
- **Steelman (Chesterton's Fence)** — the strongest case for the status quo, and why the proposal survives it (or how it was narrowed). This section is mandatory — it's what distinguishes a kaikaku proposal from a naive rewrite.
- **Migration plan** — the staged path (expand/contract, stacked PRs, data backfill if any), and what stays back-compatible across the window.
- **Blast radius & risk** — files/modules touched, the reversibility, and the failure modes.
- **Open questions** — anything that needs a human/product call before implementation.

### Label by the steelman's outcome, not by default

The deliverable's label depends on **how the fence pass resolved**, not a fixed default. Don't reflexively stamp every proposal `needs-design`:

- **Steelman *converged*** — it landed on a concrete plan with no open architectural forks (a standard/obvious mechanism, a staged migration, and only *empirical assumptions* left to check). This is a groomed **spec**, not a design question. **Verify the steelman's key empirical assumptions yourself first** (e.g. "no non-pool connection reads this type" — go confirm it), then label it `shovel-ready` (or the repo's build-queue label). It can enter the build queue; a human glance is welcome but not a gate.
- **Steelman left *open forks*** — it narrowed the candidate but a real decision/trade-off remains (`partially_removable` with a genuine choice of shape, competing migrations, or a load-bearing concern you couldn't fully resolve). Label it `needs-design` and gate on human direction. Frame the open fork explicitly in the issue.

The failure mode to avoid: labeling a verified, converged, standard-pattern change `needs-design` and parking it behind a human review it doesn't need. (Precedent: the first kaikaku run filed the JSONB-codec proposal `needs-design`, but its steelman had fully converged — the only open item was verifying the pool-scope assumption; once verified it was reclassified `shovel-ready`.)

### The hand-off gate

- **Default (no `--auto`):** present the drafted proposal in-session for the user to approve, redirect, or reject *before* filing. Architecture decisions deserve human eyes; expect the user to reframe. On approval, file the issue (`gh issue create`) and report the URL.
- **With `--auto`:** file the issue directly (labeled for async review) and notify (`PushNotification`) — the human reviews on the issue tracker, like a PR review. The steelman gate is what makes unattended filing safe.

Either way, the loop's output is the filed issue. Record the proposed target in conversation/memory so the next recon doesn't re-surface it while it's pending review.

## Phase 7 — Declare the outcome

End the iteration with the `LOOP-OUTCOME` line; `/loop` owns everything after.

- `filed` — a design-issue was filed (Phase 4′). `/loop` treats it as productive and re-arms **long** (a filed proposal awaits human review; there's no fast-follow). This is the architectural cadence — nothing to do at 60s.
- `gate_killed` — recon found candidates but none survived the fence + gate (Phase 1.5). Non-productive; `/loop` idles. A fence-kill is a valuable *confirmation* that the complexity is earned — report it.
- `empty` — recon surfaced nothing at architectural altitude. Non-productive.
- `blocked` — you escalated to the user (e.g. two candidates within leverage noise, or the default hand-off gate is awaiting approval).

`--auto` is kaikaku's own flag — "file without in-session approval" (Phase 4′), *not* "merge" (there is no merge). The loop flags (`--once`, `--idle-count`, `--drain`) are `/loop`'s.

**The pending-review backlog is kaikaku's diminishing-returns signal.** If several filed proposals are still awaiting human direction, don't keep filing more — declare `gate_killed` (noting the backlog as the reason) or `blocked` to ask the user, so `/loop` stops fast-filing into an unreviewed pile.

## Boundaries

(Additive to build-cycle's shared boundaries.)

- **Don't build.** No implementation PRs from this loop. If a surviving candidate is small and contained enough to *want* to build, it was probably a `/kaizen` target — route it there.
- **Don't propose a rewrite as a simplification.** "Rewrite subsystem X" is not a kaikaku proposal unless the steelman shows the current shape is genuinely accidental *and* the migration is staged and bounded.
- **Don't re-surface a pending or rejected proposal.** Track filed/rejected targets so recon excludes them until they're resolved.
- **Don't speculate on future flexibility.** Kaikaku proposes present-tense structural integrity, not extensibility for hypothetical futures (that's the speculative-generality smell it *removes*).

## Escalation

`AskUserQuestion` when (in addition to build-cycle's shared triggers):

- Two surviving candidates are within leverage noise of each other (let the user pick the architectural direction).
- A candidate's steelman is genuinely split (`partially_removable` with a load-bearing core) — surface the trade-off rather than deciding it.
- The proposal touches a public API / wire format / data model that other systems depend on.
- The recon keeps surfacing the same fence-kill — that's a signal the *project's* documented rationale for that complexity might be worth re-confirming with the user, not re-litigating every iteration.

When in doubt, ask. A wrong architectural proposal costs more credibility than a wrong kaizen PR — it asks the user to seriously consider tearing out something that works.
