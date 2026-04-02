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
.claude/
├── agents/
│   ├── orchestrator.md ← Runs the full pipeline, returns concise summary
│   ├── architect.md    ← Plans before code gets written
│   ├── coder.md        ← Implements following your conventions
│   ├── tester.md       ← Writes & runs tests (the ONLY agent that runs tests)
│   ├── reviewer.md     ← Reviews with your priorities in mind
│   └── design-qa.md    ← Visual QA via screenshots (web + mobile)
├── hooks/
│   ├── git-context.sh    ← Branch-aware git history on session start (optional)
│   └── agent-reminder.sh ← Points Claude to @orchestrator (always installed)
└── scripts/
    └── verify.sh         ← Finds & runs affected tests (called by tester agent)
```

The **orchestrator** is the key. Instead of each agent dumping verbose output into your main chat, the orchestrator runs the full pipeline internally and returns a structured summary. Your main context stays clean.

Each agent is a markdown file with its own **system prompt**, **tool restrictions**, **model choice**, and **persistent memory**. Everything is fully editable.

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

Also updates your `CLAUDE.md` with an Agent Workflow section and optionally installs **session hooks** that keep Claude on track (see below).

<br>

### Phase 4 — Verify

Pick an agent and test it with a real task before you leave the session.

<br>

## Visual Validation

Agents can't see what they're building — unless you give them eyes.

All agent output stays inside the orchestrator — your main chat gets back a structured summary with changes, test results, review findings, and any issues.

**Web projects** use inline Playwright scripts with `domcontentloaded` (never `networkidle`). **Mobile projects** use Maestro + simulator/emulator screenshots (iOS Simulator or Android Emulator).

```
 You ──▶ @orchestrator ──▶ Concise summary back to you
              │
              │  (all verbose output stays inside)
              ▼
         ┌──────────┐
         │Architect │── plans
         └────┬─────┘
              ▼
         ┌──────────┐     ┌────────┐
         │  Coder   │◀───▶│ Tester │ ← loop until green
         └────┬─────┘     └────────┘
              ▼
         ┌──────────┐
         │ Reviewer │── reviews tested code
         └────┬─────┘
              ▼
         ┌──────────┐
         │Design QA │── screenshots only (UI changes)
         └──────────┘
```

<br>

## CodeGraph Integration

If [CodeGraph](https://github.com/colbymchenry/codegraph) is installed, agents trace the full dependency graph to scope their work:

```bash
# Instead of guessing which files are affected...
git diff --name-only HEAD | codegraph affected --stdin --quiet
```

Changed a utility? CodeGraph finds every test, component, and module that transitively depends on it — even files not in the diff. Without it, agents fall back to `git diff` pattern matching.

<br>

## Session Hooks

Agents only work if Claude actually uses them. Setup Agents solves this with three layers:

| Layer | When | What it does |
|:------|:-----|:-------------|
| **CLAUDE.md** | Loaded every session | Points Claude to `@orchestrator` for all significant work |
| **`agent-reminder.sh`** | Every prompt | Brief nudge: "delegate to @orchestrator" |
| **`git-context.sh`** | Session start (optional) | Injects branch-aware git history, diff stats, and working directory state |

The `agent-reminder.sh` hook is always installed. It's kept deliberately short — just a one-liner pointing to `@orchestrator`. The orchestrator's own system prompt has the full workflow details.

The `git-context.sh` hook is optional. If installed, it gives Claude immediate project awareness — which branch you're on, recent commits, and what's been changed. It's smart about branches:

- **On main/master/develop** — shows the last 10 commits
- **On a feature branch** — shows all branch commits since diverging, plus `git diff --stat` for scope
- **Just branched** — shows the base branch's last 10 commits
- **Working directory** — shows `git status --short` with staged/unstaged markers and file count

The `verify.sh` script finds changed files, traces affected tests via CodeGraph (or falls back to file pattern matching), and runs only those tests. It's called by the **tester agent** as part of the workflow — not as a Stop hook, which would wastefully run on every response.

All hooks and scripts are fully editable.

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
- Uses CodeGraph exploration tools to understand code structure before editing
- Follows your detected code style, linter rules, naming conventions
- Reads neighboring files to match existing patterns
- Runs your linter/formatter after writing code — does NOT run tests
- Follows your commit conventions

</details>

<details>
<summary><b>Tester</b> — The only agent that runs tests</summary>

<br>

- Full tool access
- The ONLY agent that runs tests — coder and design-qa do not
- Scopes to affected test files via CodeGraph (never runs the full suite)
- Curls pages first to see actual rendered HTML before writing tests
- Max 2 debug cycles per test — prevents debug spirals
- If failures are implementation bugs, reports back to coder for a fix loop
- Uses `domcontentloaded` (never `networkidle`) for web projects

</details>

<details>
<summary><b>Reviewer</b> — Reviews with your priorities in mind</summary>

<br>

- Read-only tools
- Reviews after tests pass (coder → tester loop completes first)
- Uses CodeGraph impact analysis to assess blast radius of changes
- Checks against your stated priorities (security, performance, readability, consistency)
- Looks for stack-specific anti-patterns
- Categorizes findings: blockers, suggestions, nits
- Actionable feedback — "change X to Y because Z"

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

# After setup, use the orchestrator for any significant work:
@orchestrator add user authentication to the dashboard

# It runs the full pipeline and returns a concise summary.
# You can also invoke agents directly:
@architect plan the authentication flow
@coder implement the login page
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
