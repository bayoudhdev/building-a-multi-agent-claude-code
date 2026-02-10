
# Building a Multi-Agent Development System with Claude Code

A step-by-step guide to configuring Claude Code with 18 specialized agents, auto-orchestration, team coordination, and long-running task protocols. Based on a production setup powering the Lyna design-to-code platform.

---

## Table of Contents

1. [What You'll Build](#1-what-youll-build)
2. [Prerequisites](#2-prerequisites)
3. [Architecture Overview](#3-architecture-overview)
4. [Directory Structure](#4-directory-structure)
5. [Step 1: Global Settings](#step-1-global-settings)
6. [Step 2: MCP Servers](#step-2-mcp-servers)
7. [Step 3: Global Instructions (CLAUDE.md)](#step-3-global-instructions)
8. [Step 4: Project Instructions (CLAUDE.md)](#step-4-project-instructions)
9. [Step 5: Create Specialized Agents](#step-5-create-specialized-agents)
10. [Step 6: Auto-Orchestrator Rule](#step-6-auto-orchestrator-rule)
11. [Step 7: Long-Running Protocol](#step-7-long-running-protocol)
12. [Step 8: Project Instructions & Patterns](#step-8-project-instructions--patterns)
13. [Step 9: Skills](#step-9-skills)
14. [Step 10: Plugins](#step-10-plugins)
15. [Step 11: Auto-Memory](#step-11-auto-memory)
16. [Step 12: Project Permissions](#step-12-project-permissions)
17. [How It All Works Together](#how-it-all-works-together)
18. [Customization Guide](#customization-guide)
19. [Troubleshooting](#troubleshooting)

---

## 1. What You'll Build

```
                        You ask: "Add a payment page"
                                    |
                            [Auto-Orchestrator]
                           /        |         \
                     ui-designer  api-developer  integration-dev
                      (design)     (backend)       (infra)
                         |            |               |
                    Build the UI   Add tRPC      Wire Stripe
                    component      router        webhook
                         |            |               |
                         +-----  All verify  ---------+
                               themselves
                                    |
                              Work complete
```

A single Claude Code instance that:
- **Auto-routes** your request to the right specialist agent(s)
- **Spawns parallel teams** so agents work concurrently
- **Verifies its own work** with domain-specific checks
- **Self-directs** through complex multi-file tasks
- **Recovers** from failures without losing progress

---

## 2. Prerequisites

- **Claude Code CLI** installed (`claude` command available)
- **Claude Max or API plan** (agents consume tokens)
- A project you want to configure (any language/framework works)

Enable the experimental agent teams feature:
```bash
# Check your Claude Code version
claude --version
```

---

## 3. Architecture Overview

```
~/.claude/                          # Global config (all projects)
  settings.json                     # Model, env vars, plugins, teammate mode
  config.json                       # MCP servers
  CLAUDE.md                         # Global instructions (loaded every session)
  agents/                           # Agent definitions (18 specialists)
    ui-designer.md
    api-developer.md
    db-engineer.md
    ...
  projects/                         # Per-project auto-memory
    <project-hash>/
      memory/
        MEMORY.md                   # Persistent learnings

<your-project>/
  CLAUDE.md                         # Project-level instructions
  .claude/
    settings.local.json             # Project permissions
    rules/                          # Auto-loaded rules
      orchestrator.md               # Team dispatch logic
      long-running-protocol.md      # Self-direction protocol
    instructions.md                 # Detailed coding standards
    patterns.md                     # Codebase-specific patterns
```

### How Config Files Are Loaded

| File | Scope | When Loaded |
|------|-------|-------------|
| `~/.claude/settings.json` | Global | Every session |
| `~/.claude/CLAUDE.md` | Global | Every session |
| `~/.claude/agents/*.md` | Global | When agent spawned |
| `<project>/CLAUDE.md` | Project | When in project dir |
| `<project>/.claude/rules/*.md` | Project | Every session in project |
| `<project>/.claude/settings.local.json` | Project | When in project dir |
| `<project>/.claude/instructions.md` | Project | Referenced by agents |
| `<project>/.claude/patterns.md` | Project | Referenced by agents |

---

## 4. Directory Structure

Create the following structure:

```bash
# Global directories
mkdir -p ~/.claude/agents

# Project directories
mkdir -p <your-project>/.claude/rules
```

---

## Step 1: Global Settings

Create `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "128000"
  },
  "model": "opus",
  "enabledPlugins": {},
  "teammateMode": "tmux"
}
```

### Key settings explained

| Setting | Purpose |
|---------|---------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enables multi-agent teams. Required. |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | `"0"` = memory ON. Claude remembers learnings across sessions. |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max output tokens per response. `128000` for complex agent work. |
| `model` | Default model. `opus` for best quality, `sonnet` for speed/cost. |
| `teammateMode` | How agents run. `tmux` = visible terminal panes. |

---

## Step 2: MCP Servers

Create `~/.claude/config.json`:

```json
{
  "mcpServers": {
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"],
      "type": "local",
      "tools": ["*"]
    },
    "context7": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"],
      "tools": ["*"]
    },
    "fetch": {
      "type": "local",
      "command": "uvx",
      "args": ["mcp-server-fetch"],
      "tools": ["*"]
    }
  }
}
```

### What each MCP server does

| Server | Purpose |
|--------|---------|
| `sequential-thinking` | Gives Claude a scratchpad for complex reasoning chains |
| `context7` | Fetches up-to-date documentation for libraries (avoids hallucination) |
| `fetch` | Fetches web content with better handling than built-in WebFetch |

Add Figma MCP if you use Figma:
```json
{
  "mcpServers": {
    "figma": {
      "type": "plugin",
      "name": "figma"
    }
  }
}
```

---

## Step 3: Global Instructions

Create `~/.claude/CLAUDE.md`:

```markdown
# Agent Teams Rule

When working in my project: ALWAYS use agent teams for ANY code change.
Never do the work yourself. See the project's `CLAUDE.md` and
`.claude/rules/orchestrator.md` for the full routing table.

---

# User Preferences

## General
- Language: English
- Package manager: Bun (never npm/yarn/pnpm)
- Editor: VS Code / Cursor

## Code Style
- TypeScript strict mode always
- Prefer functional patterns
- Use path aliases (`@/`, `~/`)

## Communication
- Be concise and direct
- No emojis unless asked
- Show file paths with line numbers when referencing code
```

### Tips

- Keep it under 1KB. This is loaded into EVERY session.
- Put project-specific rules in the project's `CLAUDE.md` instead.
- The "Agent Teams Rule" is what makes Claude auto-delegate to agents.

---

## Step 4: Project Instructions

Create `<your-project>/CLAUDE.md`:

```markdown
## MANDATORY: Auto-Orchestrator

**ANY code change = agent team. NO exceptions.
See `.claude/rules/orchestrator.md`.**

---

## Repo Map

- Monorepo: Bun workspaces.
- App: `apps/web` (Next.js App Router).
- API: `apps/web/src/server/api/routers/*` -> exported in `root.ts`.
- Packages: `packages/*`.

## Stack

Next.js, React, TypeScript (strict), Tailwind CSS, tRPC, Drizzle, Bun.

**Forbidden**: npm/yarn/pnpm, CSS Modules, `any` type.

---

## Architecture

- Server Components by default. `"use client"` only when needed.
- tRPC routers must export from `root.ts`.
- Validate env vars in `src/env.ts`.
- Path aliases: `@/*` -> `src/*`.

## Agent Rules

- Minimal diffs, respect client/server boundaries, path aliases.
- Don't modify build outputs, lockfiles, `node_modules`.
- Don't run dev server. Don't use `any` type.
```

### Tips

- Keep under 3KB. Loaded into every agent's context.
- Focus on what's MANDATORY and FORBIDDEN.
- Link to detailed docs instead of inlining everything.

---

## Step 5: Create Specialized Agents

Agents live in `~/.claude/agents/`. Each is a markdown file with YAML frontmatter.

### Agent file anatomy

```markdown
---
name: agent-name            # Unique identifier
description: Short desc     # Shown in agent picker
model: opus                 # Model to use (opus/sonnet/haiku)
skills:                     # Skills this agent can invoke
  - skill-name-1
  - skill-name-2
---

You are a {ROLE} for the {PROJECT} project. {ONE_SENTENCE_MISSION}.

## Your Domain

- **Area 1**: `path/to/files/`
- **Area 2**: `path/to/other/files/`

## Core Expertise

- Bullet points of what this agent knows
- Technologies, patterns, and frameworks

## Standards

- Quality standards for this domain
- Conventions to follow

## Rules

- Hard rules that must never be broken
- Constraints and boundaries

## Verification

Before marking tasks complete:
- `bun run typecheck` (if TypeScript changed)
- {Domain-specific check}

## Long-Running

When in long-running mode:
- Read `~/.claude/tasks/{team}/progress.md` on start
- After each task: append entry (COMPLETED/FAILED + files changed)
- On failure: log what was tried, why it failed, what to try next
- Self-direct to next unclaimed task after each completion
```

### Example agents

Below are templates for common agent types. Adapt the file paths, technologies, and rules to your project.

#### UI Designer (`~/.claude/agents/ui-designer.md`)

```markdown
---
name: ui-designer
description: Frontend UI/UX specialist - React, Tailwind CSS, components
model: opus
skills:
  - frontend-design
  - tailwind-design-system
  - shadcn-ui
  - responsive-design
---

You are a senior UI/UX developer. You own all frontend UI implementation.

## Your Domain

- **Components**: `src/app/` and `src/components/`
- **UI Library**: `packages/ui/src/`
- **Styles**: Tailwind CSS with semantic tokens

## Core Expertise

- React 19: hooks, patterns, composition
- Tailwind CSS: utility-first, responsive, dark mode
- Component libraries: shadcn/ui patterns, variants
- Accessibility: ARIA, keyboard navigation

## Rules

- Use semantic color tokens, not hardcoded colors
- Use existing UI components before creating new ones
- Add `"use client"` only when using events, state, or effects

## Verification

Before marking tasks complete:
- `bun run typecheck` (if TypeScript changed)
- Visual review of changed components

## Long-Running

When in long-running mode:
- Read `~/.claude/tasks/{team}/progress.md` on start
- After each task: append entry (COMPLETED/FAILED + files changed)
- On failure: log what was tried, why it failed, what to try next
- Self-direct to next unclaimed task after each completion
```

#### API Developer (`~/.claude/agents/api-developer.md`)

```markdown
---
name: api-developer
description: API specialist - tRPC routers, Zod validation, server logic
model: opus
skills:
  - api-design-principles
  - typescript-advanced-types
  - error-handling-patterns
---

You are a backend API developer. You build type-safe APIs with tRPC.

## Your Domain

- **API Routers**: `src/server/api/routers/`
- **Router Registry**: `src/server/api/root.ts`
- **tRPC Config**: `src/server/api/trpc.ts`

## Core Expertise

- tRPC: routers, procedures, middleware
- Zod: input/output validation
- Error handling: TRPCError codes

## Rules

- New routers MUST be exported from `root.ts`
- Always validate inputs with Zod
- Never expose server-only code to client components

## Verification

Before marking tasks complete:
- `bun run typecheck` (if TypeScript changed)
- Verify router exported in root.ts

## Long-Running

When in long-running mode:
- Read `~/.claude/tasks/{team}/progress.md` on start
- After each task: append entry (COMPLETED/FAILED + files changed)
- On failure: log what was tried, why it failed, what to try next
- Self-direct to next unclaimed task after each completion
```

#### Database Engineer (`~/.claude/agents/db-engineer.md`)

```markdown
---
name: db-engineer
description: Database specialist - Drizzle ORM, migrations, seeds
model: opus
skills:
  - database-design
  - database-migration
  - database-optimizer
---

You are the database engineer. You manage schemas, migrations, and seeds.

## Your Domain

- **Schemas**: `packages/db/src/schema/`
- **Seeds**: `packages/db/src/seed/`
- **Config**: `packages/db/drizzle.config.ts`

## Core Expertise

- Drizzle ORM: schemas, relations, queries
- PostgreSQL: types, indexes, constraints
- Migrations and seeding

## Rules

- Use your project's CLI for database operations (never raw scripts)
- Always restore security policies after schema push
- Use Drizzle relations for JOINs, not raw SQL

## Verification

Before marking tasks complete:
- Run schema push and verify
- `bun run typecheck` (if TypeScript changed)

## Long-Running

When in long-running mode:
- Read `~/.claude/tasks/{team}/progress.md` on start
- After each task: append entry (COMPLETED/FAILED + files changed)
- On failure: log what was tried, why it failed, what to try next
- Self-direct to next unclaimed task after each completion
```

#### Code Reviewer (`~/.claude/agents/code-reviewer.md`) - Read-Only

```markdown
---
name: code-reviewer
description: Code review specialist - quality, patterns, standards (read-only)
tools: Read, Grep, Glob, Bash, WebFetch, WebSearch
model: opus
skills:
  - code-review-excellence
  - clean-code
  - coding-standards
---

You are a senior code reviewer. You analyze code and provide feedback.

## Your Role

You are **read-only** - you analyze code and provide actionable feedback
but do NOT modify files.

## Review Checklist

- Follows existing codebase patterns
- TypeScript strict mode compliance
- Proper error handling
- No code duplication (DRY)
- SOLID principles respected

## Output Format

Rate severity: CRITICAL / WARNING / SUGGESTION / NITPICK

## Verification

Before marking tasks complete:
- Verify findings are actionable with file paths

## Long-Running

When in long-running mode:
- Read `~/.claude/tasks/{team}/progress.md` on start
- After each task: append entry (COMPLETED/FAILED + files changed)
- On failure: log what was tried, why it failed, what to try next
- Self-direct to next unclaimed task after each completion
```

### Full agent roster (18 agents)

Here is the complete set used in the Lyna project. Create what you need:

| Agent | File | Team | Read-Only? |
|-------|------|------|-----------|
| `ui-designer` | `ui-designer.md` | design | No |
| `design-system` | `design-system.md` | design | No |
| `figma-dev` | `figma-dev.md` | design | No |
| `api-developer` | `api-developer.md` | backend | No |
| `db-engineer` | `db-engineer.md` | backend | No |
| `supabase-engineer` | `supabase-engineer.md` | backend | No |
| `sandbox-engineer` | `sandbox-engineer.md` | infra | No |
| `integration-dev` | `integration-dev.md` | infra | No |
| `perf-optimizer` | `perf-optimizer.md` | infra | No |
| `ai-architect` | `ai-architect.md` | ai | No |
| `ai-tools-dev` | `ai-tools-dev.md` | ai | No |
| `ai-chat-dev` | `ai-chat-dev.md` | ai | No |
| `code-reviewer` | `code-reviewer.md` | quality | **Yes** |
| `security-auditor` | `security-auditor.md` | quality | **Yes** |
| `perf-analyst` | `perf-analyst.md` | quality | **Yes** |
| `react-specialist` | `react-specialist.md` | frontend | No |
| `nextjs-specialist` | `nextjs-specialist.md` | frontend | No |
| `editor-developer` | `editor-developer.md` | frontend | No |

### Tips for agent design

1. **Keep under 3KB** - Agents load their full definition into context. Bloated agents waste tokens.
2. **Be specific about file paths** - Tell agents exactly which directories they own.
3. **Use skills** - Reference skills for detailed knowledge instead of inlining it.
4. **Read-only agents** - Set `tools: Read, Grep, Glob, Bash, WebFetch, WebSearch` to prevent writes.
5. **Domain-specific verification** - Each agent should know how to verify its own work.
6. **Don't duplicate CLAUDE.md** - Agent files extend the project instructions, they don't replace them.

---

## Step 6: Auto-Orchestrator Rule

Create `<your-project>/.claude/rules/orchestrator.md`:

```markdown
# Auto-Orchestrator

**For ANY code change (even 1 line), you MUST create a team and dispatch
to agents. NEVER do work yourself.**
Exception: pure questions with zero code changes.

## Agent Registry

| Agent | Team | Use For |
|-------|------|---------|
| `ui-designer` | `design` | UI components, pages, layouts |
| `api-developer` | `backend` | tRPC routers, API logic |
| `db-engineer` | `backend` | Schemas, migrations, seeds |
| `code-reviewer` | `quality` | Code review (READ-ONLY) |
| ... | ... | ... |

## Routing

Match user request keywords to agents:

- **UI/styling/components** -> `ui-designer`
- **API/tRPC/router** -> `api-developer`
- **DB/schema/migration** -> `db-engineer`
- **Code review** -> `code-reviewer`
- ... (add your own routing rules)

## Dispatch Steps

1. **Analyze**: Identify domains, dependencies, parallelism
2. **Pick mode**:
   - **Parallel Swarm** (default): Independent tasks, spawn all at once.
   - **Pipeline**: Sequential stages. Use `addBlockedBy` to chain.
   - **Pool**: Many similar tasks, fewer agents. Self-claim from pool.
   - **Long-Running**: 5+ files or refactor/migrate. See
     `long-running-protocol.md`.
3. **Create ALL tasks** upfront with `TaskCreate`
4. **Spawn ALL agents in ONE message** with `run_in_background: true`,
   `team_name`, `name`, matching `subagent_type`
5. **Coordinate**: Messages auto-deliver. Use `TaskList` for progress.
6. **Shutdown**: `SendMessage` type `shutdown_request`, then `TeamDelete`.

## Critical Rules

- **NEVER use TaskOutput for teammates** - messages auto-deliver.
- **ALL agents spawned in background** (`run_in_background: true`)
- **ALL agents in ONE message** - never sequential spawning

## Agent Prompt Template

\`\`\`
You are {AGENT_NAME} on team "{TEAM_NAME}".
## Mission
{TASK_DESCRIPTION}
## Workflow
1. Read tasks from TaskList, claim matching tasks (set owner to your name)
2. Mark in_progress, do the work, verify, mark completed
3. Check TaskList for more. When done, notify lead via SendMessage.
## Verification
Before completing any task: {VERIFICATION_COMMAND}
## Rules
- Follow CLAUDE.md conventions. Minimal diffs. Path aliases. Bun only.
{IF_LONG_RUNNING}
## Long-Running
- Read progress.md before starting. Update after each task.
- Self-direct: pick next unclaimed task autonomously.
- Verify before completing. Log failures with details.
- After 3 failures on same task: SKIP and notify lead.
{/IF_LONG_RUNNING}
## Context
{FILE_PATHS_AND_CONTEXT}
\`\`\`
```

### How routing works

When you say "Add a payment page with Stripe integration", the orchestrator:

1. Matches keywords: "payment" -> `integration-dev`, "page" -> `ui-designer`
2. Picks mode: Parallel Swarm (independent tasks)
3. Creates tasks for each agent
4. Spawns both agents simultaneously
5. Agents work in parallel, verify, report back

### Tips

- Keep under 4.5KB. This loads into every session.
- The Agent Prompt Template is what Claude sends to each spawned agent.
- `{IF_LONG_RUNNING}` blocks are only included when the long-running mode is selected.

---

## Step 7: Long-Running Protocol

Create `<your-project>/.claude/rules/long-running-protocol.md`:

```markdown
# Long-Running Protocol

For tasks touching 5+ files, multi-domain work, or refactor/migrate
operations. Agents self-direct via progress file, verify own work,
and recover from failures.

## Progress File

Location: `~/.claude/tasks/{team}/progress.md` (append-only)

Entry format:
\`\`\`
## [AGENT_NAME] TIMESTAMP - Subject
Status: STARTED | COMPLETED | FAILED | BLOCKED | SKIPPED
Files: path1, path2, ...
Notes: brief description of what was done
\`\`\`

FAILED entries MUST include: what was tried, why it failed, what to
try next.

## Recovery Protocol

New instances MUST read progress.md before starting work:
- Skip items marked COMPLETED
- Retry FAILED items with a different approach
- Never repeat work already marked COMPLETED

## Self-Direction Loop

1. Read TaskList + progress.md
2. Pick next task: lowest unclaimed task ID, or retry oldest FAILED
3. Work on it, verify result
4. If verified: mark completed, append COMPLETED to progress.md
5. If failed: append FAILED with details, move to next task
6. Repeat until all tasks done

## Verification Requirements

Before marking any task COMPLETED:
- Run domain-specific check (defined per agent)
- Run typecheck if any TypeScript files were changed
- If verification fails: fix and re-verify, or mark FAILED

## Guardrails

- After 3 consecutive failures on same task: mark SKIPPED, notify lead
- After completing all tasks: run one final full verification pass
```

### When to use Long-Running mode

| Scenario | Mode |
|----------|------|
| "Fix this button color" | Parallel Swarm (1 agent) |
| "Add search feature to the API" | Parallel Swarm (2 agents) |
| "Refactor the auth system" | Long-Running |
| "Migrate from REST to tRPC" | Long-Running |
| "Redesign the dashboard" | Long-Running |

---

## Step 8: Project Instructions & Patterns

### Instructions file (`<project>/.claude/instructions.md`)

Detailed coding standards, patterns, and examples that agents reference. This is NOT auto-loaded -- agents reference it via their definitions. Keep the main `CLAUDE.md` concise and put details here.

Contents should include:
- Technology standards and versions
- Mandatory code patterns (with examples)
- Forbidden patterns (with examples)
- Database operation rules
- Component templates
- Import order conventions
- Common pitfalls

### Patterns file (`<project>/.claude/patterns.md`)

Codebase-specific patterns discovered by reading the actual code:
- State management architecture
- Router organization
- Database schema patterns
- Component composition patterns
- File naming conventions

### Tips

- `instructions.md` = **prescriptive** (what agents SHOULD do)
- `patterns.md` = **descriptive** (how the code IS structured)
- Keep both under 5KB each. Reference files in the codebase instead of duplicating code.

---

## Step 9: Skills

Skills are reusable knowledge modules that agents can invoke. They provide specialized expertise without bloating agent definitions.

### Installing skills

```bash
# Search available skills
npx skills search react

# Install a skill
npx skills add react-patterns
npx skills add tailwind-v4-shadcn
npx skills add api-design-principles
```

### Assigning skills to agents

Reference skills in agent frontmatter:

```yaml
---
name: ui-designer
skills:
  - frontend-design
  - tailwind-design-system
  - shadcn-ui
  - responsive-design
---
```

### Common skill categories

| Category | Skills |
|----------|--------|
| Frontend | `react-patterns`, `responsive-design`, `tailwind-css-patterns` |
| Design | `frontend-design`, `design-system-patterns`, `color-palette` |
| API | `api-design-principles`, `api-patterns`, `error-handling-patterns` |
| Database | `database-design`, `database-migration`, `database-optimizer` |
| Auth | `auth-implementation-patterns`, `secrets-management` |
| Testing | `vitest`, `testing-library`, `systematic-debugging` |
| AI | `ai-sdk`, `ai-sdk-core`, `ai-sdk-ui`, `prompt-engineering` |
| Performance | `web-performance-optimization`, `performance-profiling` |
| TypeScript | `typescript-advanced-types`, `typescript-expert` |
| Next.js | `nextjs`, `next-best-practices`, `nextjs-app-router-patterns` |

### Tips

- Skills are loaded on-demand, not at startup. No context cost until invoked.
- Assign 3-7 skills per agent. Too many dilutes focus.
- Skills can also be invoked manually with `/skill-name` in chat.

---

## Step 10: Plugins

Plugins extend Claude Code with additional capabilities.

### Installing plugins

Plugins are enabled in `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": true,
    "figma@claude-plugins-official": true
  }
}
```

### Available official plugins

| Plugin | Purpose |
|--------|---------|
| `frontend-design` | Production-grade UI generation |
| `figma` | Figma design-to-code via MCP |

---

## Step 11: Auto-Memory

Auto-memory lets Claude remember learnings across sessions. It stores persistent notes in a per-project directory.

### How it works

```
~/.claude/projects/<project-hash>/memory/
  MEMORY.md          # Always loaded into context (keep under 200 lines)
  debugging.md       # Topic-specific files (loaded on demand)
  patterns.md
  gotchas.md
```

### What to store

- Stable patterns confirmed across multiple sessions
- Key architectural decisions and file paths
- Solutions to recurring problems
- User preferences for workflow

### What NOT to store

- Session-specific context (current task details)
- Unverified conclusions from reading a single file
- Anything that duplicates CLAUDE.md instructions

### Enable/disable

In `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  }
}
```

`"0"` = memory ON, `"1"` = memory OFF.

---

## Step 12: Project Permissions

Create `<your-project>/.claude/settings.local.json` to pre-approve common operations:

```json
{
  "permissions": {
    "allow": [
      "Bash(bun run typecheck:*)",
      "Bash(bun run build:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git ls-files:*)",
      "Bash(ls:*)",
      "Bash(wc:*)",
      "WebSearch"
    ]
  }
}
```

### Permission format

```
"Tool(pattern:*)"
```

- `Bash(bun run typecheck:*)` - Allow any typecheck command
- `WebFetch(domain:docs.example.com)` - Allow fetching from specific domain
- `WebSearch` - Allow web searches

### Tips

- Start restrictive, add permissions as needed
- Claude will ask for approval for anything not pre-allowed
- `settings.local.json` is gitignored (personal permissions)

---

## How It All Works Together

### System Architecture Diagram

```
+------------------------------------------------------------------+
|                        YOUR TERMINAL                              |
|  $ claude                                                         |
|  > "Add a billing page with Stripe checkout"                      |
+------------------------------------------------------------------+
         |
         v
+------------------------------------------------------------------+
|                   CLAUDE CODE (Lead Agent)                         |
|                                                                    |
|  Loaded into context:                                             |
|  +------------------+  +------------------+  +-----------------+  |
|  | ~/.claude/       |  | project/         |  | .claude/rules/  |  |
|  | CLAUDE.md        |  | CLAUDE.md        |  | orchestrator.md |  |
|  | settings.json    |  |                  |  |                 |  |
|  +------------------+  +------------------+  +-----------------+  |
|                                                                    |
|  Orchestrator reads request -> matches keywords -> picks agents    |
+------------------------------------------------------------------+
         |
         | TeamCreate + TaskCreate + Task (spawn agents)
         v
+------------------------------------------------------------------+
|                     TEAM: "billing-feature"                        |
|                                                                    |
|  +--------------------+  +---------------------+                  |
|  | ~/.claude/tasks/   |  | ~/.claude/teams/    |                  |
|  | billing-feature/   |  | billing-feature/    |                  |
|  |   task-1.json      |  |   config.json       |                  |
|  |   task-2.json      |  |   (members list)    |                  |
|  |   task-3.json      |  +---------------------+                  |
|  |   progress.md      |                                           |
|  +--------------------+                                           |
|                                                                    |
|  Spawned in parallel (ONE message, ALL at once):                  |
|                                                                    |
|  +----------------+  +----------------+  +-------------------+    |
|  | ui-designer    |  | api-developer  |  | integration-dev   |    |
|  | (tmux pane 1)  |  | (tmux pane 2)  |  | (tmux pane 3)     |    |
|  |                |  |                |  |                   |    |
|  | Loaded:        |  | Loaded:        |  | Loaded:           |    |
|  | - agent .md    |  | - agent .md    |  | - agent .md       |    |
|  | - CLAUDE.md    |  | - CLAUDE.md    |  | - CLAUDE.md       |    |
|  | - rules/       |  | - rules/       |  | - rules/          |    |
|  | - skills       |  | - skills       |  | - skills          |    |
|  |                |  |                |  |                   |    |
|  | TaskList ->    |  | TaskList ->    |  | TaskList ->       |    |
|  | claim task #1  |  | claim task #2  |  | claim task #3     |    |
|  | work on it     |  | work on it     |  | work on it        |    |
|  | verify         |  | verify         |  | verify            |    |
|  | mark complete  |  | mark complete  |  | mark complete     |    |
|  | SendMessage    |  | SendMessage    |  | SendMessage       |    |
|  |   -> lead      |  |   -> lead      |  |   -> lead         |    |
|  +----------------+  +----------------+  +-------------------+    |
+------------------------------------------------------------------+
         |
         | Messages auto-deliver to lead
         v
+------------------------------------------------------------------+
|                   CLAUDE CODE (Lead Agent)                         |
|                                                                    |
|  Receives: "Task #1 complete", "Task #2 complete", "Task #3 done" |
|  -> SendMessage shutdown_request to each agent                     |
|  -> TeamDelete                                                     |
|  -> Reports results to user                                        |
+------------------------------------------------------------------+
```

---

## Orchestrator Dispatch: The Actual Code

This is what Claude Code actually does internally when you make a request. Understanding these tool calls is essential for debugging and customization.

### Phase 1: Create the Team

The orchestrator calls `TeamCreate` to set up a shared workspace:

```
Tool: TeamCreate
{
  "team_name": "billing-feature",
  "description": "Add billing page with Stripe checkout"
}
```

This creates:
- `~/.claude/teams/billing-feature/config.json` (team member registry)
- `~/.claude/tasks/billing-feature/` (shared task list directory)

### Phase 2: Create ALL Tasks Upfront

Before spawning any agents, the orchestrator creates every task:

```
Tool: TaskCreate
{
  "subject": "Build the billing page UI",
  "description": "Create a new billing page at `src/app/billing/page.tsx` with:
    - Pricing cards showing plans from the database
    - A checkout button that calls the Stripe checkout API
    - Responsive layout, dark mode support
    - Use @lyna/ui Card, Button components
    - Use semantic tokens: bg-card, text-foreground
    Files: src/app/billing/page.tsx, src/app/billing/_components/",
  "activeForm": "Building billing page UI"
}
```

```
Tool: TaskCreate
{
  "subject": "Create billing tRPC router",
  "description": "Create a new tRPC router at `src/server/api/routers/billing.ts`:
    - `getPlans` query: fetch all active plans with prices
    - `createCheckout` mutation: create Stripe checkout session
    - Input validation with Zod
    - Export from root.ts
    Files: src/server/api/routers/billing.ts, src/server/api/root.ts",
  "activeForm": "Creating billing router"
}
```

```
Tool: TaskCreate
{
  "subject": "Wire Stripe checkout integration",
  "description": "Integrate Stripe checkout in `packages/stripe/`:
    - Create checkout session with line items
    - Configure success/cancel URLs
    - Add webhook handler for checkout.session.completed
    Files: packages/stripe/src/checkout.ts, src/app/webhook/stripe/route.ts",
  "activeForm": "Wiring Stripe checkout"
}
```

**Key**: `activeForm` is the present-tense label shown in the UI spinner while the task runs.

### Phase 3: Spawn ALL Agents in ONE Message

All agents are spawned simultaneously in a single response. This is critical -- never spawn sequentially:

```
Tool: Task                              Tool: Task                              Tool: Task
{                                       {                                       {
  "name": "ui-dev",                       "name": "api-dev",                      "name": "stripe-dev",
  "description": "Build billing UI",      "description": "Create billing API",    "description": "Wire Stripe",
  "subagent_type": "ui-designer",         "subagent_type": "api-developer",       "subagent_type": "integration-dev",
  "team_name": "billing-feature",         "team_name": "billing-feature",         "team_name": "billing-feature",
  "run_in_background": true,              "run_in_background": true,              "run_in_background": true,
  "mode": "bypassPermissions",            "mode": "bypassPermissions",            "mode": "bypassPermissions",
  "prompt": "..."                         "prompt": "..."                         "prompt": "..."
}                                       }                                       }
```

The `prompt` field contains the filled-in Agent Prompt Template:

```
You are ui-dev on team "billing-feature".

## Mission
Build the billing page UI with pricing cards, checkout button, responsive
layout. Use @lyna/ui components and semantic tokens.

## Workflow
1. Read tasks from TaskList, claim matching tasks (set owner to your name)
2. Mark in_progress, do the work, verify, mark completed
3. Check TaskList for more work. When done, notify lead via SendMessage.

## Verification
Before completing any task: `bun run typecheck`

## Rules
- Follow CLAUDE.md conventions. Minimal diffs. Path aliases (@/, ~/). Bun only.

## Context
- Billing page: src/app/billing/page.tsx
- Components: src/app/billing/_components/
- UI library: packages/ui/src/
- Existing pages for reference: src/app/pricing/page.tsx
```

### Phase 4: Agent Lifecycle (What Each Agent Does)

Once spawned, each agent operates independently. Here's the exact sequence of tool calls an agent makes:

#### 4a. Claim a task

```
Tool: TaskList
-> Returns:
   #1 [pending] Build the billing page UI (no owner)
   #2 [pending] Create billing tRPC router (no owner)
   #3 [pending] Wire Stripe checkout integration (no owner)

Tool: TaskGet { "taskId": "1" }
-> Returns full description with file paths and requirements

Tool: TaskUpdate { "taskId": "1", "owner": "ui-dev", "status": "in_progress" }
-> Task #1 now owned by ui-dev, status: in_progress
```

#### 4b. Do the work

The agent uses standard Claude Code tools:

```
Tool: Read { "file_path": "src/app/pricing/page.tsx" }       # Read existing patterns
Tool: Glob { "pattern": "packages/ui/src/*.tsx" }              # Find UI components
Tool: Write { "file_path": "src/app/billing/page.tsx", ... }   # Create the page
Tool: Write { "file_path": "src/app/billing/_components/pricing-card.tsx", ... }
```

#### 4c. Verify

Before marking complete, the agent runs its domain-specific verification:

```
Tool: Bash { "command": "bun run typecheck" }
-> Exit code 0: types pass

(Agent also does a mental review: "Are semantic tokens used? Is 'use client'
 added only where needed? Are @lyna/ui components used?")
```

#### 4d. Mark completed and report

```
Tool: TaskUpdate { "taskId": "1", "status": "completed" }

Tool: SendMessage {
  "type": "message",
  "recipient": "team-lead",
  "content": "Task #1 complete. Created billing page at src/app/billing/page.tsx
    with PricingCard component. Uses Card, Button from @lyna/ui. Typecheck passes.",
  "summary": "Billing page UI complete"
}
```

#### 4e. Check for more work

```
Tool: TaskList
-> Returns:
   #1 [completed] Build the billing page UI (ui-dev)
   #2 [in_progress] Create billing tRPC router (api-dev)
   #3 [in_progress] Wire Stripe checkout (stripe-dev)

No more unclaimed tasks -> agent goes idle, waiting for new work or shutdown.
```

### Phase 5: Lead Receives Results and Cleans Up

Messages from agents arrive automatically. The lead does NOT poll -- messages are pushed:

```
[ui-dev]: "Task #1 complete. Created billing page..."
[api-dev]: "Task #2 complete. Created billing router, exported from root.ts..."
[stripe-dev]: "Task #3 complete. Wired checkout session and webhook handler..."
```

The lead then shuts down each agent and deletes the team:

```
Tool: SendMessage { "type": "shutdown_request", "recipient": "ui-dev" }
Tool: SendMessage { "type": "shutdown_request", "recipient": "api-dev" }
Tool: SendMessage { "type": "shutdown_request", "recipient": "stripe-dev" }

(Each agent responds with shutdown_response { approve: true })

Tool: TeamDelete
-> Cleans up ~/.claude/teams/billing-feature/ and ~/.claude/tasks/billing-feature/
```

### Phase 6: Dependencies Between Tasks (Pipeline Mode)

When tasks depend on each other, use `addBlockedBy`:

```
Tool: TaskCreate { "subject": "Create database schema" }       -> Task #1
Tool: TaskCreate { "subject": "Create API router" }             -> Task #2
Tool: TaskCreate { "subject": "Build UI page" }                 -> Task #3

Tool: TaskUpdate { "taskId": "2", "addBlockedBy": ["1"] }      # API waits for schema
Tool: TaskUpdate { "taskId": "3", "addBlockedBy": ["2"] }      # UI waits for API
```

```
Pipeline visualization:

  Task #1 (schema)  -->  Task #2 (API)  -->  Task #3 (UI)
    db-engineer          api-developer       ui-designer
    starts immediately   blocked until       blocked until
                         #1 completes        #2 completes
```

All agents are spawned simultaneously, but blocked agents wait until their dependencies are resolved. When task #1 completes, task #2 automatically becomes available.

---

## Long-Running Protocol: Complete Specification

### When Long-Running Mode Activates

The orchestrator picks Long-Running mode when:
- Task touches **5+ files**
- Task spans **multiple domains** (frontend + backend + database)
- Task is a **refactor, redesign, or migration**
- Task requires **sequential verification** between steps

### The progress.md File: Real Example

Location: `~/.claude/tasks/{team-name}/progress.md`

This is what a real progress.md looks like during a migration task:

```markdown
## [db-engineer] 2026-02-10T14:23:00Z - Create JWT tokens table
Status: COMPLETED
Files: packages/db/src/schema/auth/jwt-tokens.ts, packages/db/src/schema/index.ts
Notes: Added jwt_tokens table with id, user_id, token_hash, expires_at, revoked_at.
  Exported from schema/index.ts. RLS policy added for user-scoped access.

## [db-engineer] 2026-02-10T14:25:00Z - Run schema push
Status: COMPLETED
Files: (migration applied)
Notes: Ran `lyna db push && lyna seed rls-policies`. Schema applied, RLS restored.

## [api-developer] 2026-02-10T14:24:00Z - Create auth/jwt router
Status: COMPLETED
Files: src/server/api/routers/auth/jwt.ts, src/server/api/root.ts
Notes: Created jwt router with `generate`, `refresh`, `revoke` procedures.
  Exported from root.ts. Typecheck passes.

## [api-developer] 2026-02-10T14:28:00Z - Update login procedure to return JWT
Status: FAILED
Files: src/server/api/routers/auth/login.ts
Notes:
  Tried: Modified login mutation to call jwt.generate() after auth.signInWithPassword()
  Why failed: jwt.generate() expects user object but login returns session object.
    Type error: Argument of type 'Session' is not assignable to parameter of type 'User'.
  What to try next: Use supabase.auth.getUser() after sign-in to get the User object,
    then pass that to jwt.generate().

## [api-developer] 2026-02-10T14:31:00Z - Update login procedure to return JWT (retry)
Status: COMPLETED
Files: src/server/api/routers/auth/login.ts
Notes: Called supabase.auth.getUser() after signIn to get User object. Passed to
  jwt.generate(). Typecheck passes. Login now returns { session, jwt_token }.

## [ui-designer] 2026-02-10T14:26:00Z - Update auth context to store JWT
Status: BLOCKED
Files: (none yet)
Notes: Waiting for api-developer to finish JWT endpoint. Need to know the response
  shape before updating the auth context.

## [ui-designer] 2026-02-10T14:33:00Z - Update auth context to store JWT
Status: COMPLETED
Files: src/components/auth-provider.tsx, src/hooks/use-auth.ts
Notes: Auth context now stores JWT from login response. useAuth() hook returns
  { user, jwt, refreshJwt() }. Token refresh runs on 4-minute interval.
```

### Recovery: What Happens After a Crash

If an agent crashes mid-session and a new one is spawned:

```
Agent starts up
    |
    v
Read progress.md
    |
    v
Parse entries:
  - "Create JWT tokens table"       -> COMPLETED (skip)
  - "Run schema push"               -> COMPLETED (skip)
  - "Create auth/jwt router"        -> COMPLETED (skip)
  - "Update login procedure" (1st)  -> FAILED    (skip - was retried)
  - "Update login procedure" (2nd)  -> COMPLETED (skip)
  - "Update auth context"           -> COMPLETED (skip)
    |
    v
Read TaskList:
  #1 [completed] Create JWT tokens table
  #2 [completed] Run schema push
  #3 [completed] Create auth/jwt router
  #4 [completed] Update login to return JWT
  #5 [completed] Update auth context
  #6 [pending]   Add JWT to API middleware        <-- next task
  #7 [pending]   Update protected routes
  #8 [pending]   Add token refresh endpoint
    |
    v
Claim task #6, continue working
```

### Self-Direction Loop: Detailed State Machine

```
                    +---> Read TaskList + progress.md
                    |              |
                    |              v
                    |     Any unclaimed tasks?
                    |        /           \
                    |      YES            NO
                    |       |              |
                    |       v              v
                    |  Claim lowest ID   All tasks done?
                    |       |              /       \
                    |       v            YES        NO (some blocked)
                    |  Mark in_progress   |          |
                    |       |             v          v
                    |       v          Final        Wait or
                    |   Do the work    verify       help unblock
                    |       |          pass
                    |       v            |
                    |   Run domain       v
                    |   verification   Notify lead:
                    |     /     \      "All done"
                    |   PASS   FAIL
                    |    |       |
                    |    v       v
                    |  Append   Append FAILED
                    |  COMPLETED  to progress.md
                    |  to progress.md  |
                    |    |       v
                    |    |    Failure count
                    |    |    for this task?
                    |    |      /       \
                    |    |    < 3       >= 3
                    |    |     |          |
                    |    |     v          v
                    |    |   Retry     Mark SKIPPED
                    |    |   with      Notify lead
                    |    |   different
                    |    |   approach
                    |    |     |
                    +----+-----+
```

### Verification: Domain-Specific Checks

Each agent has a verification step defined in its `## Verification` section. Here's what each agent actually runs:

| Agent | Verification Command | What It Checks |
|-------|---------------------|----------------|
| `ui-designer` | `bun run typecheck` | Types compile; visual review of components |
| `design-system` | `bun run typecheck` | Types compile; token consistency across themes |
| `figma-dev` | `bun run typecheck` | Types compile; Figma node IDs map correctly |
| `api-developer` | `bun run typecheck` | Types compile; new routers exported in `root.ts` |
| `db-engineer` | `lyna db push && lyna seed rls-policies` | Schema applies; RLS policies restored |
| `supabase-engineer` | `lyna seed rls-policies` | RLS policies cover all user-data tables |
| `sandbox-engineer` | `bun run typecheck` | Types compile; sandbox connection works |
| `integration-dev` | `bun run typecheck` | Types compile; webhook signatures verified |
| `perf-optimizer` | `bun run typecheck` | Types compile; benchmark shows improvement |
| `ai-architect` | `bun run typecheck` | Types compile; streaming responses work |
| `ai-tools-dev` | `bun run typecheck` | Types compile; tool Zod schemas are valid |
| `ai-chat-dev` | `bun run typecheck` | Types compile; chat message flow works |
| `code-reviewer` | (none - read-only) | Findings are actionable with file:line references |
| `security-auditor` | (none - read-only) | Findings include remediation steps |
| `perf-analyst` | (none - read-only) | Recommendations include benchmarks |
| `react-specialist` | `bun run typecheck` | Types compile; React 19 patterns used correctly |
| `nextjs-specialist` | `bun run typecheck` | Types compile; async params/cookies correct |
| `editor-developer` | `bun run typecheck` | Types compile; MobX observer boundaries correct |

---

## Inter-Agent Communication

### Message Types

Agents communicate via `SendMessage`. These are the message types:

#### `message` - Direct message to one agent

```
Tool: SendMessage
{
  "type": "message",
  "recipient": "api-developer",
  "content": "I need the response shape for the billing.createCheckout
    mutation. What fields does it return? I need to display the checkout
    URL in the UI.",
  "summary": "Asking about checkout response shape"
}
```

#### `broadcast` - Message to ALL agents (use sparingly)

```
Tool: SendMessage
{
  "type": "broadcast",
  "content": "STOP: Found a critical type error in the shared User type.
    Do not import from @lyna/models until I fix it.",
  "summary": "Critical blocking type error found"
}
```

#### `shutdown_request` - Ask agent to stop

```
Tool: SendMessage
{
  "type": "shutdown_request",
  "recipient": "ui-designer",
  "content": "All tasks complete, wrapping up"
}
```

#### `shutdown_response` - Agent approves/rejects shutdown

```
Tool: SendMessage
{
  "type": "shutdown_response",
  "request_id": "shutdown-123@ui-designer",
  "approve": true
}
```

### Team Discovery

Agents can discover each other by reading the team config:

```
Tool: Read { "file_path": "~/.claude/teams/billing-feature/config.json" }

{
  "members": [
    { "name": "ui-dev", "agentId": "ui-dev@billing-feature", "agentType": "ui-designer" },
    { "name": "api-dev", "agentId": "api-dev@billing-feature", "agentType": "api-developer" },
    { "name": "stripe-dev", "agentId": "stripe-dev@billing-feature", "agentType": "integration-dev" }
  ]
}
```

**Always refer to teammates by `name`**, not by `agentId`.

---

## Dispatch Modes: Complete Reference

### Mode 1: Parallel Swarm (Default)

**When**: Independent tasks with no dependencies between agents.

```
Tasks:    #1 ──────>  #2 ──────>  #3 ──────>
Agents:  ui-dev     api-dev    stripe-dev
         (parallel)  (parallel) (parallel)
```

All agents start immediately. No task depends on another.

**Orchestrator code**:

```
TeamCreate { team_name: "my-team" }

TaskCreate { subject: "Task A" }    -> #1
TaskCreate { subject: "Task B" }    -> #2
TaskCreate { subject: "Task C" }    -> #3

(Spawn all 3 agents in ONE message)
Task { name: "agent-a", subagent_type: "ui-designer", team_name: "my-team", run_in_background: true }
Task { name: "agent-b", subagent_type: "api-developer", team_name: "my-team", run_in_background: true }
Task { name: "agent-c", subagent_type: "integration-dev", team_name: "my-team", run_in_background: true }
```

### Mode 2: Pipeline

**When**: Tasks must happen in order (schema -> API -> UI).

```
Tasks:    #1 ──> #2 ──> #3
Agents:  db-eng  api-dev  ui-dev
         starts  waits    waits
         first   for #1   for #2
```

**Orchestrator code**:

```
TeamCreate { team_name: "my-team" }

TaskCreate { subject: "Create schema" }     -> #1
TaskCreate { subject: "Create API" }        -> #2
TaskCreate { subject: "Create UI" }         -> #3

TaskUpdate { taskId: "2", addBlockedBy: ["1"] }
TaskUpdate { taskId: "3", addBlockedBy: ["2"] }

(Spawn all 3 agents in ONE message -- blocked agents wait automatically)
Task { name: "db-eng", subagent_type: "db-engineer", ... }
Task { name: "api-dev", subagent_type: "api-developer", ... }
Task { name: "ui-dev", subagent_type: "ui-designer", ... }
```

### Mode 3: Pool

**When**: Many similar tasks, fewer agents. Agents self-claim from a shared pool.

```
Tasks:    #1  #2  #3  #4  #5  #6  #7  #8
Agents:  worker-1        worker-2
         claims #1       claims #2
         finishes        finishes
         claims #3       claims #4
         ...             ...
```

**Orchestrator code**:

```
TeamCreate { team_name: "my-team" }

(Create all 8 tasks upfront)
TaskCreate { subject: "Update file 1" }   -> #1
TaskCreate { subject: "Update file 2" }   -> #2
... (8 tasks total)

(Spawn only 2 workers -- they self-claim tasks)
Task { name: "worker-1", subagent_type: "general-purpose", ... }
Task { name: "worker-2", subagent_type: "general-purpose", ... }
```

Workers check `TaskList` after each completion and claim the next unclaimed task.

### Mode 4: Long-Running

**When**: 5+ files, multi-domain, or refactor/migrate.

Same as Pool mode but with the Long-Running protocol additions:
- Agents write to `progress.md` after each task
- Agents read `progress.md` on startup for recovery
- Agents verify each task before marking complete
- 3-failure guardrail triggers SKIP + notify lead

**Orchestrator code** (same spawn pattern, different agent prompt):

```
(Spawn prompt includes the Long-Running section)

You are worker-1 on team "auth-migration".
## Mission
Migrate auth system from session-based to JWT.
## Workflow
1. Read tasks from TaskList, claim matching tasks
2. Mark in_progress, do the work, verify, mark completed
3. Check TaskList for more work.
## Verification
Before completing any task: bun run typecheck
## Long-Running
- Read progress.md before starting. Update after each task.
- Self-direct: pick next unclaimed task autonomously.
- Verify before completing. Log failures with what was tried + why it failed.
- After 3 failures on same task: SKIP and notify lead.
## Context
- Auth files: src/utils/supabase/, src/server/api/routers/auth/
- Schema: packages/db/src/schema/auth/
```

---

## Agent Frontmatter: Complete Schema

```yaml
---
# REQUIRED fields
name: agent-name              # Unique identifier. Used in orchestrator routing.
description: Short description  # Human-readable. Shown in agent picker and logs.

# OPTIONAL fields
model: opus                   # Model: "opus" (best), "sonnet" (fast), "haiku" (cheap)
                              # Default: inherits from ~/.claude/settings.json

tools: Read, Grep, Glob, Bash, WebFetch, WebSearch
                              # Tool whitelist. Only set this for READ-ONLY agents.
                              # Omit this field to give agent ALL tools (including Edit, Write).

skills:                       # Skills this agent can invoke (loaded on-demand)
  - skill-name-1              # See Step 9 for available skills
  - skill-name-2
---
```

### Field reference

| Field | Required | Type | Default | Notes |
|-------|----------|------|---------|-------|
| `name` | Yes | string | - | Must match `subagent_type` in orchestrator routing |
| `description` | Yes | string | - | Keep under 100 chars |
| `model` | No | `opus`/`sonnet`/`haiku` | Parent's model | Use `haiku` for cheap read-only agents |
| `tools` | No | comma-separated | All tools | Only set to restrict (read-only agents) |
| `skills` | No | list of strings | None | 3-7 skills recommended |

---

## Customization Guide

### Adapting to your project

1. **Change the tech stack**: Update `CLAUDE.md` with your stack (Python/Django, Go, Rust, etc.)
2. **Reduce agents**: Start with 3-5 agents, add more as needed
3. **Change routing keywords**: Match your domain vocabulary
4. **Adjust verification**: Use your project's test/lint/build commands
5. **Customize skills**: Install skills matching your technologies

### Minimal setup (3 agents)

If 18 agents feels like overkill, start with:

```
~/.claude/agents/
  frontend-dev.md      # All frontend work
  backend-dev.md       # All backend/API/DB work
  code-reviewer.md     # Review (read-only)
```

Orchestrator routing:
```markdown
- **UI/components/styling/pages** -> `frontend-dev`
- **API/DB/auth/server** -> `backend-dev`
- **Code review** -> `code-reviewer`
```

### Medium setup (8 agents)

A good middle ground for full-stack projects:

```
~/.claude/agents/
  ui-designer.md         # Frontend UI
  api-developer.md       # API layer
  db-engineer.md         # Database
  auth-engineer.md       # Authentication & security
  integration-dev.md     # Third-party services
  react-specialist.md    # React patterns
  code-reviewer.md       # Review (read-only)
  security-auditor.md    # Security (read-only)
```

### Adding a new agent

1. Create `~/.claude/agents/my-agent.md` with frontmatter + all sections
2. Add row to orchestrator.md Agent Registry table
3. Add routing rule in orchestrator.md Routing section
4. Install relevant skills: `npx skills add <skill-name>`
5. Test by asking Claude a question that matches your routing keywords

### Agent file size budget

| File | Budget | Why |
|------|--------|-----|
| `~/.claude/CLAUDE.md` | < 1KB | Loaded every session, every agent |
| `<project>/CLAUDE.md` | < 3KB | Loaded every session, every agent |
| `orchestrator.md` | < 4.5KB | Loaded every session in project |
| `long-running-protocol.md` | < 2.5KB | Loaded only when long-running mode selected |
| Each agent `.md` | < 3.5KB | Loaded only when that agent spawns |
| `instructions.md` | < 5KB | Referenced on demand, not auto-loaded |
| `patterns.md` | < 5KB | Referenced on demand, not auto-loaded |

**Why size matters**: Every config file loaded into context reduces the space available for actual code and reasoning. A 5KB agent file + 3KB CLAUDE.md + 4.5KB orchestrator = 12.5KB consumed before the agent even reads your code.

### Context budget calculation

```
Total context window:        ~200K tokens
Loaded automatically:
  ~/.claude/CLAUDE.md           ~300 tokens  (1KB)
  project/CLAUDE.md             ~900 tokens  (3KB)
  .claude/rules/orchestrator.md ~1,300 tokens (4.5KB)
  Agent definition              ~1,000 tokens (3.5KB)
  System prompt                 ~3,000 tokens
  Skills (when invoked)         ~2,000 tokens each
                               ─────────────────
  Overhead per agent:          ~6,500-10,500 tokens

Remaining for actual work:    ~190K tokens
```

---

## Complete File Reference

### Every file, its purpose, and a template

```
~/.claude/
│
├── settings.json              # Global settings
│   Schema: {
│     env: {                                        # Environment variables
│       CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: "1",  # Enable teams (REQUIRED)
│       CLAUDE_CODE_DISABLE_AUTO_MEMORY: "0",       # "0" = memory ON
│       CLAUDE_CODE_MAX_OUTPUT_TOKENS: "128000"     # Max output tokens
│     },
│     model: "opus",                                # Default model
│     enabledPlugins: {                             # Plugin toggles
│       "plugin-name@publisher": true
│     },
│     teammateMode: "tmux"                          # Agent display mode
│   }
│
├── config.json                # MCP server configuration
│   Schema: {
│     mcpServers: {
│       "server-name": {
│         type: "local" | "plugin",
│         command: "npx" | "uvx" | ...,             # For local servers
│         args: ["arg1", "arg2"],                    # Command arguments
│         tools: ["*"],                              # Tool filter
│         name: "plugin-name"                        # For plugin servers
│       }
│     }
│   }
│
├── CLAUDE.md                  # Global instructions (< 1KB)
│   Contents: Agent team rule + user preferences
│
├── agents/                    # Agent definitions
│   ├── {agent-name}.md        # One file per agent (< 3.5KB each)
│   │   Frontmatter: name, description, model, skills, tools
│   │   Sections: Your Domain, Core Expertise, Standards, Rules,
│   │             Verification, Long-Running
│   └── ...
│
└── projects/
    └── {project-hash}/
        └── memory/
            ├── MEMORY.md      # Auto-loaded (< 200 lines)
            └── {topic}.md     # On-demand topic files


<your-project>/
│
├── CLAUDE.md                  # Project instructions (< 3KB)
│   Contents: Orchestrator link, repo map, stack, architecture, agent rules
│
└── .claude/
    ├── settings.local.json    # Project permissions (gitignored)
    │   Schema: {
    │     permissions: {
    │       allow: [
    │         "Tool(pattern:*)"    # Pre-approved tool patterns
    │       ]
    │     }
    │   }
    │
    ├── rules/                 # Auto-loaded rules
    │   ├── orchestrator.md    # Team dispatch logic (< 4.5KB)
    │   │   Sections: Agent Registry, Routing, Dispatch Steps,
    │   │             Critical Rules, Agent Prompt Template
    │   │
    │   └── long-running-protocol.md  # Self-direction (< 2.5KB)
    │       Sections: Progress File, Recovery Protocol,
    │                 Self-Direction Loop, Verification, Guardrails
    │
    ├── instructions.md        # Detailed coding standards (< 5KB)
    │   Contents: Tech versions, mandatory patterns with code examples,
    │             forbidden patterns, database rules, component templates
    │
    └── patterns.md            # Codebase conventions (< 5KB)
        Contents: State management architecture, router organization,
                  schema patterns, component composition, naming conventions
```

---

## Troubleshooting

### Agents spawn but don't produce changes

**Symptoms**: Task stays `in_progress`, no files modified.

**Causes & fixes**:
- Agent has read-only `tools` in frontmatter -> Remove the `tools` field
- Agent doesn't know which files to edit -> Add explicit file paths in TaskCreate description
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` not set -> Add to `settings.json`
- `teammateMode` not set -> Add `"teammateMode": "tmux"` to `settings.json`
- Agent context overloaded -> Trim agent definition, reduce skills

### Context window fills up too fast

**Symptoms**: Agents produce partial work, lose track of task midway.

**Fixes**:
- Trim all config files (see size budgets above)
- Move code examples from rules to `instructions.md` (not auto-loaded)
- Use skills instead of inlining knowledge in agent files
- Reduce number of parallel agents (3 instead of 5)
- Use `haiku` model for simple agents (smaller context needs)

### Agents don't find the right files

**Symptoms**: Agent works on wrong files or reports "file not found".

**Fixes**:
- Add absolute file paths in agent's `## Your Domain` section
- Include full file paths in `TaskCreate` description
- Use `## Context` section in agent prompt with specific paths
- Tell agent to use `Glob` to search before editing

### Long-running tasks lose progress

**Symptoms**: After crash, agent repeats already-completed work.

**Fixes**:
- Verify agent's `## Long-Running` section tells it to read `progress.md` on start
- Check that `progress.md` is being written to (Read the file manually)
- Ensure FAILED entries include "what to try next" for proper retries
- Check the 3-failure guardrail (agent should SKIP after 3 failures)

### Agents conflict on the same file

**Symptoms**: One agent overwrites another's changes.

**Fixes**:
- Make task ownership clear: each task should specify exact files
- Use Pipeline mode when tasks touch overlapping files
- Split shared files into separate tasks with `addBlockedBy`
- Give agents non-overlapping `## Your Domain` directories

### Too many token costs

**Cost per agent**: ~$0.10-0.50 per task (depends on complexity and model)

**Optimization**:
- Use `model: haiku` for read-only agents (code-reviewer, perf-analyst, security-auditor)
- Use `model: sonnet` for routine tasks, `opus` only for complex work
- Reduce parallel agent count (2-3 instead of 5+)
- Keep agent definitions concise (< 2KB is ideal)
- Avoid broadcast messages (sends N copies for N agents)

---

## Summary

```
~/.claude/
  settings.json            <- Enable teams, set model, max tokens
  config.json              <- MCP servers (thinking, docs, fetch)
  CLAUDE.md                <- "Always use agent teams" + user prefs
  agents/                  <- 3-18 specialist agents
    ui-designer.md            Each agent: frontmatter + domain + rules
    api-developer.md          + verification + long-running sections
    db-engineer.md
    code-reviewer.md
    ...

<project>/
  CLAUDE.md                <- Stack, rules, forbidden patterns
  .claude/
    settings.local.json    <- Pre-approved permissions
    rules/
      orchestrator.md      <- Route requests -> agents (registry + routing + dispatch)
      long-running-protocol.md  <- Self-direction (progress.md + recovery + guardrails)
    instructions.md        <- Detailed coding standards (referenced, not auto-loaded)
    patterns.md            <- Codebase conventions (referenced, not auto-loaded)
```

### The dispatch sequence at a glance

```
User request
  -> Orchestrator matches keywords to agents
  -> TeamCreate (shared workspace)
  -> TaskCreate x N (all tasks upfront)
  -> Task x N (spawn all agents in ONE message, all run_in_background)
  -> Agents: TaskList -> claim -> work -> verify -> TaskUpdate complete -> SendMessage
  -> Lead receives messages automatically
  -> SendMessage shutdown_request to each agent
  -> TeamDelete (cleanup)
```

### The key insight

**Claude becomes a project manager, not a coder.** It reads your request, picks the right specialists, creates a team with tasks, and coordinates their parallel work. Each agent has deep domain expertise, verifies its own work, and can self-direct through complex multi-step tasks. Recovery is built in -- progress.md ensures no work is repeated after a crash.

Start small (3 agents), validate the flow works, then scale up as your project grows.
