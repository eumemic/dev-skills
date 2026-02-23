# Session Analysis Framework

## Phase 1: Session Analysis

Review the conversation history to identify:

### What Went Well
- Successful workflows that skills helped with
- Patterns that proved effective

### User Corrections (HIGH VALUE)
Look for places where the user had to instruct or correct the agent:
- "You need to not block yourself like that" → workflow improvement
- "Keep monitoring after you make changes" → missing proactive behavior
- "Use a shorter timeout" → operational best practice

These corrections often translate directly into skill content.

### Struggles & Workarounds
- Recurring tasks that took multiple attempts
- Workflows that had to be figured out from scratch
- Skills that should have helped but didn't load

### Discoverability Issues
- Skills that existed but weren't triggered when they should have been
- Trigger phrases that didn't match how the user actually asked

## Phase 2: Present Findings

```
## Session Retrospective

### Existing Skills Reviewed
- [skill-name]: [what it covers]

### What Went Well
- [List successful workflows]

### User Corrections
- [Quote what the user said] → [What this teaches]

### Skill Observations

#### Potential New Skills
- [Only recurring workflows/tasks - NOT code documentation]

#### Skills Needing Updates
- [Existing skills with incorrect or incomplete information]

#### Discoverability Issues
- [Skills with weak trigger phrases]

#### Not Skill Material
- [Learnings that should go in code comments or CLAUDE.md instead]
```

## Phase 3: Propose Changes

### For New Skills
```
### Proposed New Skill: [skill-name]

**Recurring task this addresses:** [What operational workflow]
**Why this is a skill (not code docs):** [Explain why this will be done repeatedly]
**Trigger phrases:** [When this should load]
**Key content:** [Workflow steps]
**Location:** [user-level or project-level, with reasoning]
```

### For Skill Updates
```
### Proposed Update: [existing-skill-name]

**Current issue:** [What's wrong or missing]
**Proposed change:** [Specific modification]
```

### For Discoverability Fixes
```
### Proposed Discoverability Fix: [skill-name]

**Current triggers:** [Existing description]
**Missing triggers:** [Phrases that should also trigger this skill]
```

## Quality Checks

Before finalizing any skill:
- [ ] Addresses a recurring operational workflow (not code documentation)
- [ ] Description uses third person ("This skill should be used when...")
- [ ] Description includes specific trigger phrases
- [ ] Location matches scope (project-specific vs general-purpose)
- [ ] Doesn't duplicate existing skills
