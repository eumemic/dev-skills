# dev-skills

General-purpose development skills for Claude Code: an autonomous loop driver + iteration skills, ship pipeline, retrospectives, git workflow, investigation methodology, and testing.

## Skills

### The autonomous loop

| Skill | Trigger | Description |
|-------|---------|-------------|
| **loop** | "/loop &lt;skill&gt; [flags]" | The loop driver. Runs one iteration of an iteration skill, reads its declared `LOOP-OUTCOME`, then paces / escalates / re-arms. Owns cadence, idle escalation, and flag passthrough so the iteration skills stay pure. E.g. `/loop kaizen --auto`, `/loop kaikaku`, `/loop shovel-ready --auto --drain`. |

### Iteration skills (one unit of work each; loop them with `/loop`)

| Skill | Trigger | Description |
|-------|---------|-------------|
| **shovel-ready** | "/shovel-ready", "drain the shovel-ready queue" | One GitHub issue-label queue iteration: take the highest-ROI labeled issue (TDD, /ship, /retro, merge), or audit-and-refill when empty. |
| **kaizen** | "/kaizen", "accidental-complexity reduction" | One audit-and-simplify iteration: parallel-agent codebase audit → highest-ROI simplification → /ship → /retro → merge. |
| **bughunt** | "/bughunt", "speculative bug-hunting" | One bug-discovery iteration: risk-pattern audit → hypothesis → red test → root-cause fix → /ship → /retro → merge. |
| **kaikaku** | "/kaikaku", "architectural simplification", "correctness by construction" | One architectural iteration: whole-system recon → Chesterton's-Fence steelman → autonomously file groomed design-issue(s) for every candidate that clears the gate (multi-output, no approval step), labeled by validation. Spans both architectural cells — simplify-and-beautify *and* correctness-by-construction (foreclose a defect-class by making an illegal state unrepresentable). Grooms, doesn't build (the architectural-altitude sibling of kaizen). |
| **build-cycle** | (not user-invokable) | Reference: the single-iteration build skeleton (branch → /ship → /retro → merge → declare outcome) shared by shovel-ready / kaizen / bughunt. |

### Pipeline + utilities

| Skill | Trigger | Description |
|-------|---------|-------------|
| **ship** | "/ship", invoked by build-cycle | Drive a working tree to a green PR: project checks → /simplify → code-review subagent → commit → push → CI monitor. Stops at green; caller handles merge. |
| **retro** | "/retro", "run a retrospective", invoked by build-cycle with `--scope=iteration` | Quality-gated retrospective: identify durable codifiable learnings; produce nothing on most invocations. |
| **git-workflow** | "commit", "push", "create a PR", "merge" | Full git workflow orchestration with safety checks. |
| **investigate** | "diagnose", "debug", "find the root cause" | Systematic investigation methodology for bugs and mysteries. |
| **test** | "test this feature", "verify this works" | Thorough testing with cleanup and go/no-go reporting. |

## How the loop composes

The loop is decomposed into two layers, so "do one unit of work" and "decide whether to do another" stay orthogonal:

- **Iteration skills** (`shovel-ready`, `kaizen`, `bughunt`, `kaikaku`) each do **one** unit of work and end by declaring `LOOP-OUTCOME: <status>`. They know nothing about looping.
- **`/loop`** runs an iteration, reads that outcome, and owns all the looping — cadence, idle escalation, re-arming, flag passthrough. It never touches repo content.

The build iteration skills share the `build-cycle` skeleton (branch → implement → /ship → /retro → merge → declare outcome); each owns only its target-acquisition logic and implementation discipline. `kaikaku` grooms a proposal instead of building, so it owns its whole iteration and skips `build-cycle`.

- **shovel-ready**: targets are issues bearing the `shovel-ready` GitHub label (extrinsic; humans populate the queue).
- **kaizen**: targets are produced intrinsically by parallel-agent codebase audits over craftsmanship dimensions.
- **bughunt**: targets are produced intrinsically by parallel-agent audits over project invariants and risk patterns; gated by red-test provability.
- **kaikaku**: targets are architectural-altitude opportunities surfaced by whole-system recon and steelmanned against Chesterton's Fence; the deliverable is a groomed design-issue, not a PR.

Each iteration applies a **quality gate** (Phase 1.5): "found something" never implies "worth shipping." A non-productive iteration (`empty` / `gate_killed`) is a healthy, expected outcome — `/loop` paces it with long-cadence idle, not a forced ship. The same principle applies to `/retro` — most retros produce no actionable findings, and that's the healthy default.

## Installation

Install via the eumemic marketplace in Claude Code:

```
/plugins → eumemic → dev-skills
```
