# dev-skills

General-purpose development skills for Claude Code: autonomous-loop drivers, ship pipeline, retrospectives, git workflow, investigation methodology, and testing.

## Skills

### Autonomous loop drivers

| Skill | Trigger | Description |
|-------|---------|-------------|
| **shovel-ready** | "/shovel-ready", "drain the shovel-ready queue" | GitHub issue-label queue drainer: pick highest-ROI labeled issue, TDD, /ship, /retro, merge; audit and refill when empty. |
| **kaizen** | "/kaizen", "continuous accidental-complexity reduction" | Parallel-agent codebase audit → highest-ROI simplification → /ship → /retro → merge → loop. |
| **bughunt** | "/bughunt", "continuous speculative bug-hunting" | Parallel-agent risk-pattern audit → hypothesis → red test → root-cause fix → /ship → /retro → merge → loop. |
| **loop-driver** | (not user-invokable) | Reference document shared by the three loop drivers above for branching, /ship handoff, /retro pass, merge handling, idle escalation. Only loaded when one of the three specializations directs you here. |

### Pipeline + utilities

| Skill | Trigger | Description |
|-------|---------|-------------|
| **ship** | "/ship", invoked by loop drivers | Drive a working tree to a green PR: project checks → /simplify → code-review subagent → commit → push → CI monitor. Stops at green; caller handles merge. |
| **retro** | "/retro", "run a retrospective", invoked by loop drivers with `--scope=iteration` | Quality-gated retrospective: identify durable codifiable learnings; produce nothing on most invocations. |
| **git-workflow** | "commit", "push", "create a PR", "merge" | Full git workflow orchestration with safety checks. |
| **investigate** | "diagnose", "debug", "find the root cause" | Systematic investigation methodology for bugs and mysteries. |
| **test** | "test this feature", "verify this works" | Thorough testing with cleanup and go/no-go reporting. |

## How the loop drivers compose

The three loop drivers share a common skeleton (target acquisition → branch → implement → /ship → /retro → merge → idle escalation). The shared phases live in `loop-driver/SKILL.md`; each specialization owns its target-acquisition logic and implementation discipline:

- **shovel-ready**: targets are issues bearing the `shovel-ready` GitHub label (extrinsic; humans populate the queue).
- **kaizen**: targets are produced intrinsically by parallel-agent codebase audits over craftsmanship dimensions.
- **bughunt**: targets are produced intrinsically by parallel-agent audits over project invariants and risk patterns; gated by red-test provability.

All three apply a per-iteration **quality gate** (Phase 1.5): "found something" never implies "worth shipping." Most iterations should ship a candidate; some iterations should ship nothing and proceed to idle escalation. The same principle applies to `/retro` — most retros produce no actionable findings, and that's the healthy default.

## Installation

Install via the eumemic marketplace in Claude Code:

```
/plugins → eumemic → dev-skills
```
