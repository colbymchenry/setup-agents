<div align="center">

<br>

# /setup-agents

**One command. Five specialized agents. Tailored to your stack.**

A [Claude Code](https://claude.ai/code) skill that builds your AI development team.

<br>

</div>

---

<br>

## The Problem

Claude Code is powerful, but it's a generalist. When you ask it to plan, code, review, test, and QA in one session — context bleeds, focus drifts, and quality suffers.

**Setup Agents** fixes this by generating **purpose-built subagents** that each do one job exceptionally well, scoped to your actual project.

<br>

## What You Get

```
.claude/agents/
├── architect.md    ← Plans before code gets written
├── coder.md        ← Implements following your conventions
├── reviewer.md     ← Reviews with your priorities in mind
├── tester.md       ← Writes & runs tests that matter
└── design-qa.md    ← Visual QA via screenshots (web + mobile)
```

Each agent is a markdown file with its own **system prompt**, **tool restrictions**, **model choice**, and **persistent memory**. They're fully editable — tweak them anytime.

<br>

## How It Works

The skill runs a 4-phase process that takes about 2 minutes:

```
 DETECT          INTERVIEW          GENERATE          VERIFY
┌──────┐        ┌──────────┐       ┌──────────┐      ┌────────┐
│Scan  │───────▶│Ask about │──────▶│Create    │─────▶│Test an │
│stack │        │preferences│      │agents    │      │agent   │
└──────┘        └──────────┘       └──────────┘      └────────┘
 Languages       Review focus       Tailored to       Try it with
 Frameworks      Architecture       YOUR stack        a real task
 Test runners    Conventions        YOUR tools
 Linters         Validation         YOUR patterns
```

<br>

### Phase 1 — Detect

Scans your project and classifies it:

| Type | Validation Strategy |
|:-----|:---|
| **Web** — React, Next.js, Vue, Svelte, Shopify, etc. | Playwright screenshots + tests |
| **Backend** — APIs, CLI tools, libraries | curl + unit/integration tests |
| **Fullstack** — Both | Playwright for UI, tests for API |
| **Mobile** — iOS, Android, React Native, Flutter | Maestro + simulator screenshots |

<br>

### Phase 2 — Interview

Four focused question batches via interactive prompts — not a wall of text:

> **Review** — What matters most? Security, performance, readability, consistency.
>
> **Architecture** — Feature-based, component-driven, MVC, or follow existing?
>
> **Testing** — Pragmatic, thorough, full coverage, or unit only?
>
> **Conventions** — Conventional Commits, descriptive, or match existing?

<br>

### Phase 3 — Generate

Creates agents that reference **your actual stack** — not generic instructions.

A coder agent for a Next.js + TypeScript project looks completely different from one for a Django + Python project. The generated prompts include your linter config, your test runner, your file patterns, your commit style.

<br>

### Phase 4 — Verify

Pick an agent and test it with a real task before you leave the session.

<br>

## Visual Validation

Agents can't see what they're building — unless you give them eyes.

**Web projects** get Playwright screenshots. After every UI change, the coder agent screenshots the page, reads it, and fixes issues before reporting done.

**Mobile projects** get Maestro + simulator/emulator screenshots. Same loop — build, screenshot, inspect, fix — on iOS Simulator or Android Emulator.

```
 ┌─────────┐     ┌────────────┐     ┌──────────┐     ┌───────┐
 │ Edit    │────▶│ Screenshot │────▶│ Inspect  │────▶│ Fix   │──┐
 │ code    │     │ the UI     │     │ visually │     │ issues│  │
 └─────────┘     └────────────┘     └──────────┘     └───────┘  │
      ▲                                                          │
      └──────────────────────────────────────────────────────────┘
                        repeat until correct
```

<br>

## CodeGraph Integration

If [CodeGraph](https://github.com/nicepkg/codegraph) is installed, agents trace the full dependency graph to scope their work:

```bash
# Instead of guessing which files are affected...
git diff --name-only HEAD | codegraph affected --stdin --quiet
```

Changed a utility? CodeGraph finds every test, component, and module that transitively depends on it — even files not in the diff. Without it, agents fall back to `git diff` pattern matching.

<br>

## Agent Details

<details>
<summary><b>Architect</b> — Plans before code gets written</summary>

<br>

- Read-only tools (no Edit or Write)
- Analyzes requests and breaks them into components
- Identifies affected files via CodeGraph
- Flags breaking changes, performance concerns, security implications
- Outputs structured plans: summary, affected files, approach, risks
- Asks clarifying questions before finalizing

</details>

<details>
<summary><b>Coder</b> — Implements following your conventions</summary>

<br>

- Full tool access (Read, Edit, Write, Bash)
- Follows your detected code style, linter rules, naming conventions
- Reads neighboring files to match existing patterns
- Runs your formatter after writing code
- Visual verification loop for UI changes (Playwright or Maestro)
- Follows your commit conventions

</details>

<details>
<summary><b>Reviewer</b> — Reviews with your priorities in mind</summary>

<br>

- Read-only tools
- Checks against your stated priorities (security, performance, readability, consistency)
- Looks for stack-specific anti-patterns
- Verifies affected tests were updated via CodeGraph
- Categorizes findings: blockers, suggestions, nits
- Actionable feedback — "change X to Y because Z"

</details>

<details>
<summary><b>Tester</b> — Writes & runs only the tests that matter</summary>

<br>

- Full tool access
- Scopes to affected test files via CodeGraph (never runs the full suite)
- Reads existing tests to match your project's test style
- Writes meaningful assertions, not coverage padding
- Fixes failures and re-runs until green
- Visual regression testing for UI projects

</details>

<details>
<summary><b>Design QA</b> — Visual quality gate for UI projects</summary>

<br>

- Read-only tools
- Auto-generated for any project with visual validation enabled
- Screenshots at multiple viewports/devices
- Web audit: layout, responsive, typography, CSS quality, accessibility
- Mobile audit: safe areas, scroll behavior, dynamic type, platform conventions, dark mode
- Reports findings with specific fixes

</details>

<br>

## Usage

```bash
# In any project, just run:
/setup-agents

# After setup, agents work automatically:
# Claude delegates planning to architect, reviews to reviewer, etc.

# Or invoke a specific agent:
@architect plan the authentication flow
@coder implement the login page
@reviewer check my latest changes
@tester write tests for the new API endpoints
```

<br>

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
# Global (available in all projects)
mkdir -p ~/.claude/skills/setup-agents
cp SKILL.md ~/.claude/skills/setup-agents/SKILL.md
```

Or clone this repo directly:

```bash
git clone https://github.com/colbymchenry/setup-agents.git ~/.claude/skills/setup-agents
```

<br>

---

<div align="center">

<br>

Built for [Claude Code](https://claude.ai/code)

<br>

</div>
