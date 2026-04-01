# Instinct Store

Instincts are learned patterns, preferences, and corrections that agents accumulate
from human feedback and self-assessment. They make agents smarter across runs.

## How It Works

1. Human reviews agent output at a checkpoint
2. Feedback is captured via `@checkpoint-feedback`
3. Proposed instincts are extracted from feedback
4. After human approval, instincts are added to the relevant agent's instinct file
5. Next time the agent runs, it reads its instincts FIRST — applying learned patterns

## File Structure

- `shared.instincts.md` — Cross-agent instincts (naming conventions, org preferences, etc.)
- `{agent-name}.instincts.md` — Agent-specific instincts

## Instinct Format

Each instinct is a clear, actionable statement:

- ✅ "Always use snake_case for PostgreSQL column names"
- ✅ "Prefer JdbcTemplate over JPA native queries for full-text search"
- ❌ "Be better at architecture" (too vague)

## Scope

- **Per-project**: Instincts in this repo apply to this project
- **Shared**: Copy instinct files to new projects to carry learnings forward
