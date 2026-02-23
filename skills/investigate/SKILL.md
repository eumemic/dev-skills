---
name: investigate
description: This skill should be used when the user asks to "diagnose", "debug", "investigate", "why is this broken", "find the root cause", "figure out why", "look at the logs", "what's going on", or when the user corrects with "you failed to get to the bottom of it" or questions apparent success. Also use when multiple attempts at fixing the same issue haven't worked, or when a problem needs systematic investigation.
---

# Investigation Methodology

Systematic approach to investigating bugs, unexpected behavior, or mysteries in a codebase.

## The Investigation Loop

Investigation follows a continuous loop until all mysteries are resolved:

### 1. Identify Highest-ROI Action

Evaluate current mysteries and questions. Determine the single action that will shed the most light on the unknowns. Prioritize actions that:
- Definitively confirm or deny a hypothesis
- Reveal actual data rather than metadata
- Have low cost relative to information gained

Avoid:
- Writing tests before confirming root cause
- Making fixes based on theories
- Broad exploration when a targeted query would suffice

### 2. Carry Out the Action

Execute the investigation work:
- Read relevant code paths
- Examine actual data (not just counts or metadata)
- Run targeted queries
- Check logs for specific timestamps
- Reproduce the issue in a controlled way

When examining data, look at content, not just structure. For example, if nodes have unexpected timestamps, look at what text those nodes contain.

### 3. Watch for New Mysteries

As work progresses, note inconsistencies or things that don't make sense. These are new mysteries. Create tasks to track them:

```
Mystery: Why do these nodes share the same timestamp when they represent different turns?
Mystery: Why is this config value 200 when it should be null?
Mystery: Why does the query return span_start > span_end?
```

Capturing mysteries as tasks prevents losing track of open questions when context shifts.

### 4. Return to Step 1

Continue the loop until:
- Root cause is confirmed with evidence
- All mysteries are resolved or explained
- A clear fix can be proposed

## Key Principles

### Confirm, Don't Theorize

Before claiming understanding, verify with evidence:

**Bad:** "The bug is probably caused by X because that would explain the symptoms."

**Good:** "Let me check if X is actually happening by examining the data at that point."

When asked "have we confirmed this?", the answer should be based on observed evidence, not logical deduction.

### Keep Asking Why (The Feynman Principle)

When you find what looks like an answer, it's usually another question. Think like a 5-year-old: keep asking "why?" until there's nothing left to understand.

**Example investigation:**
- "Summarization is stuck" → Why?
- "There's a 31-character gap blocking it" → Why is there a gap?
- "The chunking didn't cover that span" → Why didn't it cover that span?
- "The turn grouping skipped those characters" → Why did it skip them?
- "The JSONL record had an unexpected format" → Why? What record?
- ... keep going until you hit bedrock

Stop asking "why" only when you reach:
1. A fundamental design decision that can't be questioned further
2. A clear bug with an obvious fix
3. External constraints (API behavior, hardware limits)

**If you're tempted to fix something, you probably stopped asking "why" too soon.**

A constraint blocking summarization might be *correctly* detecting a malformed document. The question isn't "how do I disable the constraint" but "why is the document malformed?"

### Preserve Evidence

Avoid mutating state until the bug is understood:
- Don't restart services that might clear relevant state
- Don't delete logs that might contain clues
- Export or screenshot data before making changes
- If reproduction requires state changes, document the original state first

### Examine Actual Data

Surface-level inspection often misses the real issue:

**Surface level:** "There are 198 leaves in the document"

**Actual data:** "Let me look at what text those leaves contain and what their timestamps are"

The bug often hides in the content, not the counts.

### Delegate Deep Dives

For context-heavy investigation work, use subagents:
- Subagents get fresh context focused on the specific mystery
- Main context stays available for coordination
- Results come back summarized

## Anti-Patterns

### Theorizing Without Evidence

Going in circles making confident assertions without verifying:
- "It must be X" → "Actually maybe Y" → "Could be Z"
- Each theory sounds plausible but none are confirmed

**Fix:** Stop theorizing. Pick the highest-ROI action to confirm one hypothesis.

### Premature Test Writing

Writing a regression test before confirming the root cause:
- Test may not actually reproduce the bug
- Time wasted if theory is wrong
- May encode incorrect assumptions

**Fix:** Confirm root cause first, then write a test that definitively fails without the fix.

### Losing Track of Mysteries

Starting to investigate one question, discovering another, and forgetting the first:
- Context window fills with tangents
- Original question never answered

**Fix:** Create tasks for each mystery immediately. Return to the task list when finishing an action.

### Surface-Level Examination

Checking metadata instead of actual content:
- "The document has 198 leaves" (but what's in them?)
- "The timestamps span 8 hours" (but are they monotonic?)
- "The config exists" (but what values does it contain?)

**Fix:** Always drill down to the actual data when something seems off.
