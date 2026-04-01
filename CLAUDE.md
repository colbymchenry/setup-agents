# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code skill definition (`SKILL.md`) that bootstraps specialized AI subagents (architect, coder, reviewer, tester, design-qa) tailored to any project. Invoked via `/setup-agents` or when a user says "setup agents."

## Architecture

The skill is a single `SKILL.md` file using YAML frontmatter for configuration and markdown body as the skill prompt. It follows a 4-phase flow:

1. **Detect** — Scans the target project (languages, frameworks, test runners, linters, project type)
2. **Interview** — Asks the user preference questions via `AskUserQuestion` (4 batches)
3. **Generate** — Creates tailored subagent `.md` files in `.claude/agents/` or `~/.claude/agents/`, then updates the project's `CLAUDE.md` with an Agent Workflow section
4. **Verify** — Offers to test an agent with a small task

Key design decisions:
- Project type classification (frontend/backend/fullstack/native mobile) drives which agents and validation strategies are generated
- CodeGraph integration (`codegraph affected --stdin`) is used for dependency-aware scoping when `.codegraph/` exists, with git diff fallback otherwise
- Playwright screenshot loops are added to coder/tester/design-qa agents for any browser-rendered project
- Each generated subagent gets restricted tool access appropriate to its role (e.g., reviewer is read-only)
- **CLAUDE.md update is critical** — after generating agents, the skill prepends an "Agent Workflow" section to the project's CLAUDE.md. Without this, fresh sessions ignore the agents and act as a generalist. The `description` field in agent frontmatter alone is not reliable enough to trigger delegation.

## Editing Guidelines

- The `allowed-tools` frontmatter field controls which tools the skill itself can use at runtime
- Interview questions use `AskUserQuestion` with structured options — never plain text
- Generated agent files use their own frontmatter schema: `name`, `description`, `tools`, `model`, `memory`
- The `description` field in generated agents influences when Claude auto-delegates to them (e.g., "Use proactively before implementing features"), but the CLAUDE.md workflow section is the primary mechanism for ensuring delegation
- The "Update Project CLAUDE.md" step in Phase 3 must always run — it's what makes agents actually get used in practice
