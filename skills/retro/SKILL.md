---
name: retro
description: This skill should be used when the user asks to "run a retro", "do a retrospective", "reflect on the session", "what did you learn", "improve skills based on this session", "codify learnings", or mentions wanting to capture insights from the current conversation for future sessions.
---

# Session Retrospective

Reflect on the current session to identify learnings that could improve future development sessions, then propose skill improvements for user approval.

## Process Overview

1. **Review existing skills** - Read `~/.claude/skills/*/SKILL.md` and `.claude/skills/*/SKILL.md` to understand what's already covered
2. **Analyze the session** - Identify what went well, user corrections, struggles, and discoverability issues
3. **Present findings** - Structured summary with proposed changes
4. **Get user approval** - Wait for explicit approval before making changes
5. **Implement** - Create/update skills using the `plugin-dev:skill-development` skill for new skills

## Key Principle

**The key question:** "Will I need to do this same task/workflow again in future sessions?"
- If yes → potentially a skill
- If no → document in code comments, CLAUDE.md, or just let it go

## Reference Files

- **`references/skill-criteria.md`** - What skills ARE and ARE NOT for, with examples
- **`references/analysis-framework.md`** - Detailed session analysis and proposal templates
