# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code skill definition (`SKILL.md`) that bootstraps specialized AI subagents (orchestrator, architect, coder, tester, reviewer, design-qa) tailored to any project. Invoked via `/setup-agents` or when a user says "setup agents."

## Architecture

The skill is a single `SKILL.md` file using YAML frontmatter for configuration and markdown body as the skill prompt. It follows a 4-phase flow:

1. **Detect** — Scans the target project (languages, frameworks, test runners, linters, project type)
2. **Interview** — Asks the user preference questions via `AskUserQuestion` (4 batches)
3. **Generate** — Creates tailored subagent `.md` files, updates `CLAUDE.md` with Agent Workflow section, and installs hooks
4. **Verify** — Offers to test the orchestrator with a small task

Key design decisions:
- **Orchestrator pattern** — An `orchestrator` agent manages the full pipeline (architect → coder → tester ↔ coder → reviewer → design-qa) internally and returns a concise structured summary. All verbose agent output stays in the orchestrator's context, keeping the user's main session clean.
- **Response format constraints** — Every agent has a response format section capping output length and requiring structured responses. This prevents context explosion even when agents are invoked directly.
- Project type classification (frontend/backend/fullstack/native mobile) drives which agents and validation strategies are generated
- CodeGraph integration (`codegraph affected --stdin` + MCP exploration tools) is used for dependency-aware scoping and code understanding when `.codegraph/` exists
- Playwright screenshots use inline `node -e` scripts with `domcontentloaded` (never `npx playwright screenshot` which uses `networkidle` and hangs)
- Only the tester agent runs tests; the coder runs the linter only; design-qa takes screenshots only — no Playwright overlap
- Each generated subagent gets restricted tool access appropriate to its role (e.g., reviewer is read-only)
- **CLAUDE.md update is critical** — after generating agents, the skill prepends an "Agent Workflow" section pointing to `@orchestrator`. Without this, fresh sessions ignore the agents.
- **Agent-reminder hook** (always installed) — brief nudge on every prompt pointing to `@orchestrator`. Git-context hook is optional.
- **verify.sh is NOT a Stop hook** — it's a utility script the tester agent calls. Running tests on every response is wasteful.

## Editing Guidelines

- The `allowed-tools` frontmatter field controls which tools the skill itself can use at runtime
- Interview questions use `AskUserQuestion` with structured options — never plain text
- Generated agent files use their own frontmatter schema: `name`, `description`, `tools`, `model`, `memory`, `maxTurns`
- The orchestrator spawns agents via `claude --agent <name> -p` (Bash), NOT the Agent tool — agents can't spawn other agents via the Agent tool (Claude Code limitation)
- Every agent must have a `## Response Format` section in its system prompt
- The "Update Project CLAUDE.md" step in Phase 3 must always run — it's what makes agents actually get used in practice
