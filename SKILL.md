---
name: setup-agents
description: Bootstrap specialized AI subagents (architect, coder, reviewer, tester) tailored to any project. Use when the user wants to set up agent workflows, create specialized coding agents, or says "setup agents". Analyzes the codebase, interviews the user about preferences, and generates project-specific subagent definitions.
user_invocable: true
allowed-tools: Read, Glob, Grep, Bash, Write, AskUserQuestion
---

# Setup Agents

You are bootstrapping specialized subagents for a project. Each subagent runs in its own isolated context window with a custom system prompt, tool restrictions, model choice, and optional persistent memory.

Subagents are markdown files in `.claude/agents/` (project-level) or `~/.claude/agents/` (global). They use YAML frontmatter for configuration and the markdown body as the system prompt.

## Overview

1. **Detect** — Scan the project, understand the stack, classify as frontend/backend/fullstack
2. **CodeGraph** — Ensure codegraph is installed and project is indexed
3. **Interview** — Ask the user targeted questions using AskUserQuestion
4. **Generate** — Create tailored subagent `.md` files based on findings + answers
5. **Verify** — Confirm everything is wired up

---

## Phase 1: Detect

Scan the project to build a profile. Look for:

| Signal | Where to look |
|--------|--------------|
| Language(s) | File extensions, config files |
| Framework | package.json, Cargo.toml, requirements.txt, go.mod, composer.json, Gemfile, etc. |
| Project type | **Frontend** (React, Vue, Svelte, Next.js, Shopify theme, HTML/CSS, etc.), **Backend** (Express, Django, FastAPI, Go services, etc.), or **Fullstack** (both) |
| Test framework | Jest, Vitest, Pytest, Playwright, Cypress, Go test, RSpec, etc. |
| Linting/formatting | ESLint, Prettier, Biome, Black, Ruff, RuboCop, etc. |
| CI/CD | .github/workflows/, .gitlab-ci.yml, Jenkinsfile, etc. |
| Build tools | Vite, Webpack, Turbopack, esbuild, etc. |
| Project structure | Monorepo vs single, feature-based vs layered, src/ layout |
| Existing conventions | CLAUDE.md, .editorconfig, contribution guides |
| Package manager | npm, pnpm, yarn, bun, pip, cargo, etc. |
| Type system | TypeScript, Flow, Python type hints, etc. |
| CodeGraph | Does `.codegraph/` exist? Agents should use codegraph-scoped workflows |
| Dev server | What command starts the local server? What port? (for Playwright screenshots) |
| Has UI | Does this project render visual output a user sees in a browser? |

**Classify the project type.** This determines the agent set and validation strategy:

| Type | Indicators | Agent strategy |
|------|-----------|----------------|
| **Frontend / Web app** | HTML/CSS, templates, components, browser rendering, Shopify themes, SPAs, SSR sites, Storybook | Playwright screenshots + visual QA + e2e tests (strongly recommended) |
| **Backend / API** | REST/GraphQL APIs, CLI tools, libraries, no browser UI | curl/httpie + unit/integration tests |
| **Fullstack** | Both frontend rendering and backend API | Both strategies — Playwright for UI, curl/tests for API |
| **Native mobile** | iOS (Swift/SwiftUI), Android (Kotlin), React Native, Flutter | Maestro screenshots via simulator/emulator + unit/integration tests |

**Important:** If the project renders anything in a browser, it qualifies for the Playwright screenshot validation loop. This includes Shopify themes, Next.js, React SPAs, Vue apps, Svelte, static sites, component libraries, etc. Native mobile apps use Maestro + simulator/emulator screenshots instead.

**Mobile sub-classification** (when project type is native mobile):

| Platform | Indicators | Simulator/Emulator | Screenshot command |
|----------|-----------|-------------------|-------------------|
| **iOS** | .xcodeproj, .xcworkspace, Swift/SwiftUI files, Podfile | iOS Simulator (`xcrun simctl`) | `xcrun simctl io booted screenshot /tmp/screen.png` |
| **Android** | build.gradle, AndroidManifest.xml, Kotlin/Java files | Android Emulator (`adb`) | `adb exec-out screencap -p > /tmp/screen.png` |
| **React Native** | react-native in package.json, Metro bundler | Either — detect from project config | Platform-specific command above |
| **Flutter** | pubspec.yaml, dart files, flutter in config | Either — detect from project config | Platform-specific command above |

Present a concise summary table of what you found, including the project type classification.

---

## Phase 1.5: CodeGraph Setup

CodeGraph builds a semantic knowledge graph of the codebase, enabling agents to trace dependencies, find affected test files, and scope their work to only what changed. Without it, agents have to guess which files are affected by a change. With it, they know precisely — including transitive dependencies the diff doesn't show.

**This is strongly recommended for all projects.** Tell the user why: "CodeGraph lets your agents trace the full dependency graph so they only test, review, and QA files actually affected by your changes — including downstream files not in the diff."

### Step 1: Check if codegraph is installed

```bash
command -v codegraph && codegraph --version
```

**If NOT installed:** Use AskUserQuestion:

- header: "CodeGraph"
- question: "I strongly recommend CodeGraph — it lets agents trace dependencies so they only test/review affected files, not the whole project. Install it now?"
- multiSelect: false
- options:
  - "Yes, install it (Recommended)" — Installs CodeGraph via npx
  - "Skip for now" — Agents will fall back to git diff file matching (less precise)

If the user chooses to install, run:

```bash
npx @colbymchenry/codegraph
```

This runs the interactive installer which configures the MCP server, permissions, and global instructions.

### Step 2: Check if the project is indexed

If codegraph IS installed, check for an existing index:

```bash
ls .codegraph/ 2>/dev/null
```

**If `.codegraph/` does NOT exist:** Use AskUserQuestion:

- header: "Index"
- question: "CodeGraph is installed but this project hasn't been indexed yet. Index it now so agents can trace dependencies?"
- multiSelect: false
- options:
  - "Yes, index it (Recommended)" — Builds the knowledge graph for this project
  - "Skip for now" — Agents will fall back to git diff file matching

If the user chooses to index, run:

```bash
codegraph init -i
```

This initializes the project and runs a full index.

**If `.codegraph/` already exists:** Good — note this in the detection summary. Run `codegraph status` to confirm the index is current and show the user.

### Impact on agent generation

- **CodeGraph available + indexed:** All agents get the full codegraph-scoped workflow (`git diff | codegraph affected --stdin`). This is the optimal setup.
- **CodeGraph not available or not indexed:** All agents get fallback scoping instructions using `git diff --name-only` + manual file pattern matching. Still works, but can miss transitive dependencies.

---

## Phase 2: Interview

**IMPORTANT**: Use the `AskUserQuestion` tool for ALL questions. Do NOT ask questions as plain text. Each AskUserQuestion call supports 1-4 questions with 2-4 options each.

Wait for the user's answers before proceeding to the next batch.

### Batch 1 — Confirm detection (1 AskUserQuestion call)

**Q1:**
- header: "Stack"
- question: "I detected [summarize stack + project type]. Does this look right?"
- multiSelect: false
- options:
  - "Yes, looks good" — Detected stack is correct
  - "Needs corrections" — I'll provide corrections after this

**Q2** (only if no test framework detected):
- header: "Testing"
- question: "What testing framework do you want to use?"
- multiSelect: false
- options: Pick 3-4 relevant options based on the detected language/framework. For frontend projects, always include Playwright. Examples:
  - "Playwright" — E2E browser testing with visual regression (Recommended for frontend)
  - "Jest" — Popular for JavaScript/TypeScript unit tests
  - "Vitest" — Fast, Vite-native test runner
  - "Pytest" — Python testing framework

**Q3** (only if no linter detected):
- header: "Linting"
- question: "Do you use a linter or formatter?"
- multiSelect: false
- options: Pick 2-3 relevant options based on the detected language. Examples:
  - "ESLint + Prettier" — Standard JS/TS linting and formatting
  - "Biome" — Fast all-in-one linter and formatter
  - "None" — No linter or formatter

If the user selects "Needs corrections", ask a follow-up AskUserQuestion or read their text input before continuing.

### Batch 2 — Preferences (1-2 AskUserQuestion calls, up to 5 questions)

**Q1:**
- header: "Review"
- question: "What matters most in code review?"
- multiSelect: true
- options:
  - "Security" — Catch vulnerabilities, injection, auth issues
  - "Performance" — Identify bottlenecks, unnecessary re-renders, N+1 queries
  - "Readability" — Clear naming, simple logic, easy to follow
  - "Consistency" — Follow existing patterns, naming conventions, project style

**Q2:**
- header: "Architecture"
- question: "Any architecture patterns to enforce?"
- multiSelect: false
- options: Pick 3-4 relevant options based on the detected framework. Examples for a web app:
  - "Feature-based" — Group by feature (auth/, products/, orders/)
  - "Component-driven" — Composable, reusable UI components
  - "MVC/Layered" — Separate models, views, controllers/routes
  - "No preference" — Follow whatever the codebase already does

**Q3:**
- header: "Testing"
- question: "What level of test coverage do you aim for?"
- multiSelect: false
- options:
  - "Pragmatic" — Test what matters, skip the obvious (Recommended)
  - "Unit + Integration" — Thorough unit tests plus integration tests
  - "Full coverage" — Unit, integration, and e2e tests
  - "Unit only" — Just unit tests

**Q4:**
- header: "Conventions"
- question: "Any commit/PR conventions?"
- multiSelect: false
- options:
  - "Conventional Commits" — feat:, fix:, chore:, etc.
  - "Descriptive" — Plain English, no prefix format
  - "Project has existing" — Follow what's already in git log
  - "No preference" — Whatever works

**Q5:**
- header: "Attribution"
- question: "Should commits and PRs hide AI involvement?"
- multiSelect: false
- options:
  - "Undercover mode" — No "Co-Authored-By: Claude", no AI mentions in commit messages or PR descriptions
  - "Show attribution (Default)" — Include Co-Authored-By trailer and standard AI attribution

### Batch 3 — Visual validation strategy (1 AskUserQuestion call)

**Skip this batch entirely for backend-only projects.**

---

#### Batch 3A — Web projects (frontend or fullstack)

**For any project that renders in a browser** (web apps, Shopify themes, SPAs, SSR sites, component libraries with Storybook, etc.), Playwright screenshots are the recommended validation approach. This is critical — AI agents can't see what they're building without screenshots. Even if the project already uses Jest, Vitest, or another test runner for unit tests, Playwright screenshots should be layered on top as the visual verification loop.

Tell the user: "For any frontend that renders in a browser, I strongly recommend Playwright screenshots as your visual validation loop. This gives agents actual eyes on what they're building — they screenshot after every UI change, inspect it, and fix issues before reporting done. This works alongside whatever test framework you already use."

**Q1:**
- header: "Validation"
- question: "For browser-rendered UIs, Playwright screenshots let agents visually verify their work. Enable this?"
- multiSelect: false
- options:
  - "Yes, Playwright screenshots (Recommended)" — Agents screenshot after UI changes, visually inspect, fix issues before reporting done
  - "Tests only" — Just run the test suite, no visual verification
  - "Manual" — I'll verify visually myself

**Q2:**
- header: "Dev server"
- question: "What's your local dev server URL?"
- multiSelect: false
- options: Auto-detect from package.json scripts, framework defaults, or config. Examples:
  - "http://localhost:3000" — Detected from [framework/config]
  - "http://localhost:5173" — Vite default
  - "http://127.0.0.1:9292" — Shopify CLI dev server

**Q3:**
- header: "Viewports"
- question: "Which viewports should agents check?"
- multiSelect: false
- options:
  - "Mobile + Desktop (Recommended)" — 375x812 and 1440x900
  - "Mobile + Tablet + Desktop" — 375x812, 768x1024, and 1440x900
  - "Desktop only" — 1440x900

---

#### Batch 3B — Native mobile projects

**For native mobile apps** (iOS, Android, React Native, Flutter), Maestro + simulator/emulator screenshots give agents visual verification of their work. Without this, agents are coding blind — they can't see the UI they're building.

Tell the user: "For native mobile apps, I recommend Maestro as your visual validation loop. Maestro can launch your app in a simulator/emulator, navigate to screens, and take screenshots — giving agents actual eyes on what they're building. It's lightweight, uses simple YAML flows, and works with iOS Simulator and Android Emulator."

**Q1:**
- header: "Mobile Validation"
- question: "Maestro screenshots let agents visually verify your mobile app in a simulator/emulator. Enable this?"
- multiSelect: false
- options:
  - "Yes, Maestro screenshots (Recommended)" — Agents screenshot after UI changes via simulator/emulator, visually inspect, fix issues before reporting done
  - "Tests only" — Just run the test suite, no visual verification
  - "Manual" — I'll verify visually myself

**Q2:**
- header: "Platform"
- question: "Which platform(s) should agents validate?"
- multiSelect: false
- options: Auto-detect from project config. Examples:
  - "iOS only" — iOS Simulator (requires Xcode)
  - "Android only" — Android Emulator (requires Android Studio)
  - "Both iOS + Android" — Validate on both platforms

**Q3:**
- header: "Screens"
- question: "How should agents navigate to affected screens for screenshots?"
- multiSelect: false
- options:
  - "Maestro flows (Recommended)" — Agents write short YAML flows to navigate and screenshot. I'll provide existing flows if I have them.
  - "Deep links" — App supports deep links to specific screens (e.g., myapp://settings)
  - "Manual launch" — Just screenshot whatever is on screen after app launch

### Batch 4 — Agent configuration (1 AskUserQuestion call, up to 4 questions)

**Q1:**
- header: "Location"
- question: "Where should I install these agents?"
- multiSelect: false
- options:
  - "Project (Recommended)" — `.claude/agents/` — travels with the repo, shareable with team
  - "Global" — `~/.claude/agents/` — available in all your projects

**Q2:**
- header: "Models"
- question: "Which model strategy for agents?"
- multiSelect: false
- options:
  - "Optimized (Recommended)" — Opus for architect, Sonnet for reviewer/tester, inherit for coder
  - "All inherit" — Every agent uses whatever model your session is running
  - "All Sonnet" — Fast and cost-effective for all agents
  - "All Opus" — Maximum capability for all agents

**Q3:**
- header: "Memory"
- question: "Should agents have persistent memory across sessions?"
- multiSelect: false
- options:
  - "Project (Recommended)" — Agents learn project patterns, shareable via git
  - "Local" — Project-specific but gitignored
  - "Global" — Agents remember across all projects
  - "None" — No persistent memory

**Q4:**
- header: "Extras"
- question: "Any additional specialized agents?"
- multiSelect: true
- options: Pick 3-4 relevant suggestions based on the detected stack. Examples:
  - "DevOps" — Infrastructure, deployment, CI/CD specialist
  - "Security" — Security-focused analysis and hardening
  - "Documenter" — Documentation and API reference writer
  - "Migrator" — Upgrade and migration specialist

**Note:** For any project where visual validation was enabled in Batch 3 (Playwright for web OR Maestro for mobile), the `design-qa` agent is automatically generated — it does NOT need to be selected as an extra. It is part of the core agent set for any project with visual output.

---

## Phase 3: Generate

Create each subagent as a `.md` file in the chosen location.

Use the detected stack info AND user answers to tailor every agent. Don't write generic instructions — reference the actual framework, test runner, linter, and conventions of THIS project.

### Subagent file format

Each generated file follows this structure:

```markdown
---
name: {agent-name}
description: {what it does, when Claude should delegate to it — be specific}
tools: {comma-separated tool list — restrict appropriately}
model: {opus, sonnet, haiku, or inherit}
memory: {user, project, local, or omit}
---

{System prompt — this is the ONLY prompt the subagent sees.
It replaces the default Claude Code system prompt entirely.
Include everything the agent needs: role, process, conventions, commands.}
```

---

### Codegraph-scoped workflow (applies to ALL agents that touch tests or changed files)

If `.codegraph/` exists in the project, every agent that needs to identify affected files MUST use this pattern in their system prompt:

```
## Scoping — Only act on what changed

Always start by determining which files are affected. Do NOT scan the full project.

1. Get changed files:
   git diff --name-only HEAD

2. Find ALL affected files (including downstream dependencies) via codegraph:
   git diff --name-only HEAD | codegraph affected --stdin --quiet

3. Filter to specific file types if needed:
   git diff --name-only HEAD | codegraph affected --stdin --filter "e2e/*" --quiet

This traces the full import/dependency graph — if you changed a utility, it finds every
test and component that transitively depends on it, even if they're not in the diff.
```

If `.codegraph/` does NOT exist, agents should fall back to manual file mapping (git diff + known test file patterns).

---

### Agent: `architect`

**Purpose:** Plans implementations before code gets written. Thinks about trade-offs, identifies affected areas, considers edge cases, and produces a clear plan.

**Frontmatter:**
- `tools`: Read, Glob, Grep, Bash (read-only — no Edit or Write)
- `model`: from user's choice in Batch 4 Q2
- `memory`: from user's choice in Batch 4 Q3
- `description`: Include "Use proactively before implementing features" so Claude delegates planning automatically

**System prompt should instruct the agent to:**
- Analyze the request and break it into components
- Identify which files/modules will be affected (use codegraph tools if `.codegraph/` exists)
- Consider architectural trade-offs specific to the project's patterns (from Batch 2 Q2)
- Flag potential issues: breaking changes, performance concerns, security implications
- Output a structured plan with: summary, affected files, approach, risks, and open questions
- Include which tests need creating or updating (use codegraph affected to find them)
- Ask clarifying questions before finalizing the plan
- Reference the project's actual architecture patterns (detected + user-specified)
- Update its memory with architectural decisions and patterns discovered

---

### Agent: `coder`

**Purpose:** Writes code following the project's conventions. Focused on implementation quality.

**Frontmatter:**
- `tools`: Read, Edit, Write, Glob, Grep, Bash (full access — needs to write code and run commands)
- `model`: from user's choice in Batch 4 Q2
- `memory`: from user's choice in Batch 4 Q3
- `description`: Describe it as the implementation specialist for this specific stack

**System prompt should instruct the agent to:**
- Follow the project's detected code style (linter rules, formatting, naming conventions)
- Use the project's actual framework patterns and idioms
- Reference specific conventions from CLAUDE.md if they exist
- Write code that matches the existing codebase style (read neighboring files for context)
- Include the project's type system conventions
- Handle errors following the project's existing patterns
- NOT add unnecessary abstractions, comments, or over-engineering
- Run the project's linter/formatter after writing code if available
- Follow the commit conventions chosen in Batch 2 Q4
- If user chose "Undercover mode" in Batch 2 Q5: never include "Co-Authored-By" trailers mentioning Claude/AI, never mention AI assistance in commit messages or PR descriptions, write commits and PRs as if a human authored them
- Update its memory with codebase patterns and conventions discovered

**Frontend-specific (if project type is frontend or fullstack AND user chose Playwright screenshots):**

Add a visual verification step to the coder's process:

```
## Visual Verification (MANDATORY after any UI change)

After editing any file that affects visual output, take Playwright screenshots to verify:

npx playwright screenshot --viewport-size={viewport_sizes} {dev_server_url}/{affected_page} /tmp/screenshot.png

Read the screenshot with the Read tool to visually inspect it. If something looks wrong,
fix it and re-screenshot. Do NOT report done without visual confirmation.
```

Use the viewport sizes and dev server URL from Batch 3A.

**Mobile-specific (if project type is native mobile AND user chose Maestro screenshots):**

Add a mobile visual verification step to the coder's process:

```
## Visual Verification — Mobile (MANDATORY after any UI change)

After editing any file that affects visual output, use Maestro to screenshot in the simulator/emulator:

### iOS Simulator
xcrun simctl io booted screenshot /tmp/ios_screen.png

### Android Emulator
adb exec-out screencap -p > /tmp/android_screen.png

### To navigate to a specific screen first, use a Maestro flow:
maestro test -e SCREENSHOT=true {flow_file}.yaml

Read the screenshot with the Read tool to visually inspect it. If something looks wrong,
fix it and re-screenshot. Do NOT report done without visual confirmation.

If the app isn't running, build and launch it first:
{build_and_run_command}
```

Replace `{build_and_run_command}` with the project's actual build/run command (e.g., `npx react-native run-ios`, `flutter run`, `xcodebuild ... | xcrun simctl install booted`). Replace `{flow_file}` with the appropriate Maestro flow path if the user has existing flows, otherwise instruct the agent to write a short YAML flow to navigate to the affected screen.

---

### Agent: `reviewer`

**Purpose:** Reviews code changes for quality, focusing on what the user said matters most.

**Frontmatter:**
- `tools`: Read, Glob, Grep, Bash (read-only — reviewers don't modify code)
- `model`: from user's choice in Batch 4 Q2
- `memory`: from user's choice in Batch 4 Q3
- `description`: Include "Use proactively after code changes" so Claude auto-delegates reviews

**System prompt should instruct the agent to:**
- Use the codegraph-scoped workflow to identify all affected files from the diff
- Run `git diff` to see recent changes, or review specified files
- Check against the user's stated priorities from Batch 2 Q1 (security, performance, readability, consistency)
- Look for anti-patterns specific to the detected stack
- Verify consistency with the project's conventions
- Check that affected tests were updated (use codegraph affected to verify coverage)
- Provide actionable feedback — not just "this could be better" but "change X to Y because Z"
- Categorize findings: blockers, suggestions, nits
- Be opinionated but not pedantic — focus on things that actually matter
- Consult its memory for patterns and recurring issues it's seen before
- Update its memory with new patterns and anti-patterns discovered

---

### Agent: `tester`

**Purpose:** Writes and runs tests using the project's actual test framework.

**Frontmatter:**
- `tools`: Read, Edit, Write, Glob, Grep, Bash (needs to write test files and run them)
- `model`: from user's choice in Batch 4 Q2
- `memory`: from user's choice in Batch 4 Q3
- `description`: Describe it as the testing specialist for the specific test framework

**System prompt MUST include the codegraph-scoped workflow:**

```
## Step 1: Scope — Only test what changed

Always start by determining which tests are affected. Do NOT run the full suite.

# Get changed files
git diff --name-only HEAD

# Find affected test files via codegraph (if .codegraph/ exists)
git diff --name-only HEAD | codegraph affected --stdin --filter "{test_glob}" --quiet

# If codegraph is unavailable, fall back to the file mapping:
# {source_pattern} → {test_pattern}

This gives you the exact list of spec files to write/update/run.
```

Replace `{test_glob}` with the project's test file pattern (e.g., `e2e/*`, `__tests__/*`, `*_test.go`).

**System prompt should also instruct the agent to:**
- Read 2-3 existing test files to match the project's test style
- Read the changed source files to understand expected behavior
- Write/update ONLY the affected test files
- Run ONLY the affected tests, not the full suite
- Fix failures and re-run until green
- Focus on meaningful assertions, not just coverage padding
- Use the project's existing test utilities, fixtures, and helpers
- Update its memory with test patterns and common failure modes

**Frontend-specific (if project type is frontend or fullstack):**

Add visual regression testing to the tester's process:

```
## Visual Regression

For sections/components with visual output, include screenshot assertions:

npx playwright screenshot --viewport-size={viewport_sizes} {dev_server_url}/{page} /tmp/screenshot.png

Read screenshots with the Read tool to visually inspect. Add toHaveScreenshot()
assertions for layout-critical sections.
```

**Mobile-specific (if project type is native mobile AND user chose Maestro screenshots):**

Add mobile visual regression testing to the tester's process:

```
## Visual Regression — Mobile

For screens/components with visual output, screenshot via simulator/emulator after running tests:

### iOS Simulator
xcrun simctl io booted screenshot /tmp/ios_screen.png

### Android Emulator
adb exec-out screencap -p > /tmp/android_screen.png

### To navigate to a specific screen and screenshot in one step:
maestro test {flow_file}.yaml

Read screenshots with the Read tool to visually inspect. Write Maestro flows for
layout-critical screens to use as repeatable visual regression checks.
```

---

### Agent: `design-qa` (auto-generated for visual projects)

**Generate this agent automatically if:** The user enabled visual validation (Playwright screenshots in Batch 3A OR Maestro screenshots in Batch 3B). This is a core agent for any visual project, not an optional extra.

**Purpose:** Visual QA specialist. Screenshots affected screens at multiple viewports/devices, audits layout, checks responsive behavior, catches visual issues.

**Frontmatter:**
- `tools`: Read, Glob, Grep, Bash (read-only — QA doesn't modify code)
- `model`: from user's choice in Batch 4 Q2
- `memory`: from user's choice in Batch 4 Q3
- `description`: "Visual QA for layout, responsive design, and UI quality. Use after UI changes to verify visual quality."

**System prompt should instruct the agent to:**

1. **Scope using codegraph** — Use the codegraph-scoped workflow to find all files affected by the diff, filter to visual files (templates, CSS, components, views, screens)

2. **Screenshot affected screens** — Use the appropriate tool for the project type:

   **Web projects (Playwright):**
   ```
   npx playwright screenshot --viewport-size={width},{height} {dev_server_url}/{page} /tmp/{viewport}.png
   ```

   **Mobile projects (Maestro + simulator/emulator):**
   ```
   # Navigate to the affected screen via Maestro flow, then screenshot:
   maestro test {flow_file}.yaml

   # Or screenshot directly if already on the right screen:
   # iOS
   xcrun simctl io booted screenshot /tmp/ios_{screen_name}.png
   # Android
   adb exec-out screencap -p > /tmp/android_{screen_name}.png
   ```

   For mobile, screenshot on each platform the user chose in Batch 3B Q2.

3. **Read and inspect each screenshot** with the Read tool

4. **Audit against a checklist** tailored to the project type:

   **Web projects:**
   - Layout: grid alignment, overflow, spacing consistency
   - Responsive: each viewport renders correctly, no awkward breakpoints
   - Typography: font loading, heading hierarchy, line lengths
   - Images: responsive srcset/sizes, lazy loading, aspect ratios
   - CSS quality: no !important abuse, uses project's CSS variables, no fixed pixel widths
   - Accessibility: color contrast, focus indicators, reduced motion

   **Mobile projects:**
   - Layout: proper use of safe areas, no content under notch/status bar/nav bar
   - Scrolling: content not clipped, scroll views work correctly, keyboard avoidance
   - Typography: dynamic type / font scaling respected, text not truncated
   - Navigation: back buttons, tab bars, gestures working correctly
   - Platform conventions: follows iOS HIG or Material Design guidelines as appropriate
   - Dark mode: if supported, verify both light and dark appearances
   - Device sizes: check on smallest supported device (SE/compact) and largest

5. **Report findings** categorized by screen/viewport with specific fixes
6. **Update memory** with recurring UI issues, platform quirks, and layout patterns

---

### Additional agents

If the user selected extras in Batch 4 Q4, generate those too following the same pattern: purpose-built, project-specific, with appropriate tool restrictions, model, and codegraph-scoped workflows where relevant.

---

## Phase 4: Verify

After generating all agents, use AskUserQuestion:

- header: "Done"
- question: "Agents created! Want to test one with a small task?"
- multiSelect: false
- options:
  - "Test architect" — Give it a feature to plan
  - "Test reviewer" — Have it review recent changes
  - "Test coder" — Give it something small to implement
  - "Skip" — I'll try them out later

Also tell the user:
- Agents are markdown files they can edit to refine
- Run `/agents` to see all available agents
- Use `@agent-name` to guarantee a specific agent runs
- Agents with memory will get smarter over time
- If codegraph is initialized, agents will automatically scope their work to only affected files
