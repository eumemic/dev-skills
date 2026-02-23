# Skill Criteria

## What Skills ARE For

Recurring operational workflows and tasks:
- Deploying to production environments
- Debugging specific types of issues (production logs, CI failures)
- Development workflows (committing, PR creation, testing)
- Setting up or configuring tools and services
- Multi-step procedures that get repeated across sessions

## What Skills Are NOT For

- Documenting code patterns or architecture (use code comments or CLAUDE.md)
- Recording one-time learnings about a specific bug fix
- API documentation or SDK usage notes (use code comments or references)
- Implementation details specific to a single feature

## Examples

| Learning | Skill? | Why |
|----------|--------|-----|
| "Railway deploys need --service flag" | Yes | Recurring deployment task |
| "Our PreToolUse hooks block paths using _is_blocked_path()" | No | Code pattern - add comment in code |
| "Debug production issues by checking Railway logs first" | Yes | Recurring debugging workflow |
| "The can_use_tool SDK callback doesn't work with our client" | No | One-time discovery - add code comment |
| "Always run tests before pushing to main" | Yes | Recurring workflow |

## Location Decision

**User-level (`~/.claude/skills/`)** - General-purpose workflows:
- Applies across multiple projects
- Generic development workflows (git, testing, debugging patterns)
- Tool-agnostic procedures

**Project-level (`.claude/skills/`)** - Project-specific workflows:
- Uses project-specific tools or services (Railway, specific APIs)
- Follows project-specific conventions
- References project-specific paths or configurations
