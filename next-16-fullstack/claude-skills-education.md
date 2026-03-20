# Claude Code Skills — Capabilities, Architecture, Patterns & Engineering Guide

**Date:** 2026-03-20
**Sources:** Anthropic official docs, Context7 indexed repos, community guides (March 2026)

---

## Table of Contents

- [1. What Are Skills](#1-what-are-skills)
- [2. Anatomy of a Skill](#2-anatomy-of-a-skill)
- [3. SKILL.md Frontmatter Reference](#3-skillmd-frontmatter-reference)
- [4. Progressive Disclosure Architecture](#4-progressive-disclosure-architecture)
- [5. Skill Discovery & Activation](#5-skill-discovery--activation)
- [6. Activation Reliability](#6-activation-reliability)
- [7. Invocation Control](#7-invocation-control)
- [8. Supporting Files & Bundled Resources](#8-supporting-files--bundled-resources)
- [9. Scripts — Deterministic Logic](#9-scripts--deterministic-logic)
- [10. Hooks — Event-Driven Automation](#10-hooks--event-driven-automation)
- [11. Sub-Agents & Parallel Execution](#11-sub-agents--parallel-execution)
- [12. Agent Teams — True Multi-Agent](#12-agent-teams--true-multi-agent)
- [13. MCP Server Integration](#13-mcp-server-integration)
- [14. State & Memory Patterns](#14-state--memory-patterns)
- [15. Plugin Architecture](#15-plugin-architecture)
- [16. Skill Composition Patterns](#16-skill-composition-patterns)
- [17. Engineering Complex Robust Skills](#17-engineering-complex-robust-skills)
- [18. Testing & Validation](#18-testing--validation)
- [19. Distribution & Marketplace](#19-distribution--marketplace)
- [20. Limitations & Failure Modes](#20-limitations--failure-modes)

---

## 1. What Are Skills

A skill is a folder containing instructions that teach Claude how to handle specific tasks or workflows. Instead of re-explaining your preferences and domain expertise in every conversation, skills let you teach Claude once and benefit every time.

**Key characteristics:**
- **Auto-triggering**: Claude reads skill descriptions and activates them when relevant
- **Progressive loading**: Only the description loads at startup (~100 tokens); full instructions load on activation
- **Modular**: Each skill is a self-contained directory
- **Composable**: Skills can reference other skills, spawn sub-agents, execute scripts
- **Standard-based**: Follow the [AgentSkills.io](https://agentskills.io) open standard (cross-tool compatible)

**Mental model:** If Claude is a new hire, skills are the SOPs you hand to them. They don't memorize every SOP — they pull out the relevant one when needed and follow it.

---

## 2. Anatomy of a Skill

```
skill-name/
├── SKILL.md              # Required — frontmatter + instructions
├── scripts/              # Optional — executable code (Python, Bash, Node)
│   ├── validate.sh
│   └── transform.py
├── references/           # Optional — heavy docs, loaded on demand
│   └── implementation.md
├── templates/            # Optional — boilerplate for generation
│   └── report-template.md
├── assets/               # Optional — static resources (configs, examples)
│   └── config.json
└── evals/                # Optional — evaluation scenarios
    └── evals.json
```

Only `SKILL.md` matters at startup. Everything else loads when explicitly needed during execution.

### Where Skills Live

| Location | Path | Applies To |
|----------|------|------------|
| Enterprise | Managed settings (read-only) | All users in org |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where plugin is enabled |

Priority: Enterprise > Personal > Project. Plugin skills use `plugin-name:skill-name` namespace (no conflicts).

---

## 3. SKILL.md Frontmatter Reference

Every SKILL.md starts with YAML frontmatter between `---` delimiters:

```yaml
---
# Required (AgentSkills.io minimum)
name: processing-invoices
description: |
  Parse invoice PDFs, extract line items, and generate reconciliation reports.
  Use when processing vendor invoices or during monthly closes.
  Trigger with 'process invoices', 'reconciliation', 'invoice PDF'.

# Tools (recommended — always scope Bash)
allowed-tools: "Read, Write, Glob, Grep, Bash(python:*)"

# Identity (marketplace-scored)
version: 1.0.0
author: Your Name <you@example.com>
license: MIT

# Claude Code extensions
model: inherit                    # sonnet, opus, haiku, or inherit
context: fork                     # Run in forked subagent
agent: general-purpose            # Subagent type (requires context: fork)
argument-hint: "<file-path>"      # Autocomplete hint after /name
user-invocable: false             # Hide from / menu (background knowledge only)
disable-model-invocation: true    # Only activate via explicit /name

# Discovery
compatible-with: claude-code, codex, openclaw
tags: [devops, automation]
---
```

### Required Fields

| Field | Type | Constraints |
|-------|------|-------------|
| `name` | string | 1-64 chars, kebab-case, must match directory name |
| `description` | string | 1-1024 chars, third person, what + when + trigger phrases |

### Description — The Most Important Field

The description determines whether and when the skill activates. Rules:

- **Third person always**: "Generates reports..." not "I generate..."
- **Must include**: what it does, when to use it (`Use when...`), and trigger phrases (`Trigger with '...'`)
- **Action verbs**: analyze, detect, generate, configure, deploy
- **Aggressive claiming**: "Make sure to use this skill whenever..." (skills frequently undertrigger)
- Every description loads into the system prompt at startup — keep it efficient (<200 chars optimal)

### Claude Code Extension Fields

| Field | What It Does | Values |
|-------|-------------|--------|
| `model` | Model override | `inherit`, `sonnet`, `opus`, `haiku` (short names only) |
| `context: fork` | Run in forked subagent (isolated context) | `fork` or `inline` (default) |
| `agent` | Subagent type (requires `context: fork`) | `Explore`, `Plan`, `general-purpose` |
| `argument-hint` | Autocomplete hint | `"<file-path>"`, `"[pr-number]"` |
| `user-invocable` | Show in `/` menu | `true` (default) or `false` |
| `disable-model-invocation` | Prevent auto-activation | `true` or `false` (default) |
| `hooks` | Skill-scoped lifecycle hooks | `{ PreToolUse: ... }` |
| `compatibility` | Environment requirements | `"Node.js >= 18"` |

### Gotchas

- `disable-model-invocation: true` + `user-invocable: false` = **unreachable skill**
- Unscoped `Bash` is a security risk — always scope: `Bash(git:*)`, `Bash(python:*)`
- `allowed-tools` must be a **CSV string**, not a YAML array (parse errors)
- `version`, `author`, `license`, `tags` must be **top-level**, not nested under `metadata:`
- Hardcoded model IDs break on deprecation — use short names: `sonnet`, `opus`, `haiku`

---

## 4. Progressive Disclosure Architecture

The core architectural idea. Skills load in three levels:

```
Level 1 — Metadata (~100 tokens)
  Just name + description. Loads for EVERY installed skill at startup.
  Total budget: ~15,000 chars via SLASH_COMMAND_TOOL_CHAR_BUDGET.
  This is your discovery surface.

Level 2 — SKILL.md body (<500 lines / <5,000 tokens)
  Loads when the skill activates. Instructions, examples, edge cases.
  Keep it concise — Claude is already capable.

Level 3 — Bundled resources (unlimited)
  references/, scripts/, templates/, assets/.
  Loaded only when explicitly needed during execution.
  Put heavy content here.
```

### Line Budgets

| Component | Guideline | Hard Max |
|-----------|-----------|----------|
| SKILL.md body | <=200 lines (optimal) | 500 lines |
| Word count | <=3,500 words | 5,000 words |
| Description | <=200 chars (efficient) | 1,024 chars |

If your SKILL.md exceeds 200 lines, extract to `references/`:

```markdown
See [implementation details](${CLAUDE_SKILL_DIR}/references/implementation.md) for advanced patterns.
```

### Why This Matters

- 10 skills with 200-char descriptions = ~2,000 tokens always in context (acceptable)
- 10 skills with 500-line bodies always loaded = context window bloat (unusable)
- Progressive disclosure keeps startup cheap and execution deep

---

## 5. Skill Discovery & Activation

Claude decides which skill to load by matching your request against all skill descriptions:

1. Claude reads `description` field of every available SKILL.md
2. Matches against your request's keywords and intent
3. If match found, loads full SKILL.md body (Level 2)
4. During execution, may load Level 3 resources as needed

### Discovery Rules

- **Vague descriptions = missed triggers**: "helps with documents" won't match "write a quarterly report"
- **Specific descriptions = reliable triggers**: "Generates quarterly performance reports from meeting notes. Use when asked for 'weekly report', 'performance summary', or 'meeting recap'."
- Multiple skills can activate for one request (multi-skill matching)
- Claude picks the best match, not all matches

### Monorepo Discovery

When editing files in subdirectories, Claude auto-discovers skills from nested `.claude/skills/` directories:

```
packages/
├── frontend/.claude/skills/    # Discovered when editing frontend files
├── backend/.claude/skills/     # Discovered when editing backend files
└── shared/.claude/skills/      # Discovered from either
```

---

## 6. Activation Reliability

Based on Scott Spence's 200+ test study (the most rigorous public benchmark):

| Approach | Success Rate | Notes |
|----------|-------------|-------|
| No hook (auto-invoke) | ~30-40% | Claude often ignores skills |
| Simple instruction hook | 20% | Passive suggestion — Claude ignores it |
| **Forced eval hook** | **84%** | Forces Claude to evaluate each skill YES/NO before proceeding |
| LLM pre-eval hook | 80% | Smarter but can fail spectacularly on edge cases |
| **Manual invocation** (`/skill-name`) | **100%** | User explicitly triggers — guaranteed |

### The Forced Eval Pattern

The difference is a **commitment mechanism**:

**Simple instruction (20%):**
```
INSTRUCTION: If the prompt matches any available skill keywords, use Skill(skill-name)
```
Claude sees this, acknowledges it, then completely ignores it.

**Forced eval (84%):**
```
Step 1 - EVALUATE: For each skill, state YES/NO with reason
Step 2 - ACTIVATE: Use Skill() tool NOW
Step 3 - IMPLEMENT: Only after activation
CRITICAL: The evaluation is WORTHLESS unless you ACTIVATE the skills.
```

The aggressive language ("MANDATORY", "WORTHLESS", "CRITICAL") makes it harder for Claude to skip. Once Claude writes "YES — need reactive state", it's committed.

### Recommendation for Complex Skills

For production workflows, **always use `user-invocable` with explicit `/skill-name` invocation**. Auto-triggering is good for reference/knowledge skills, not for complex multi-step orchestrations.

---

## 7. Invocation Control

### Two Modes

**Auto-invocation (Claude decides):**
```yaml
---
name: code-standards
description: Coding standards for this project. Use when writing or reviewing code.
---
```
Claude loads this whenever it thinks it's relevant. Good for conventions/knowledge.

**Manual invocation (user decides):**
```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---
```
Only activates when user types `/deploy`. Good for destructive/expensive operations.

### Hybrid Pattern

```yaml
---
name: research-deep
description: |
  Conducts deep web research on a topic with structured output.
  Use when asked to "research", "investigate", or "find latest info on".
  Make sure to use this skill whenever deep analysis is requested.
---
```
Auto-triggers on keyword match, but user can also invoke `/research-deep topic`.

---

## 8. Supporting Files & Bundled Resources

### File References

Reference files from SKILL.md using `${CLAUDE_SKILL_DIR}`:

```markdown
## Templates
Use the report template: @${CLAUDE_SKILL_DIR}/templates/report.md

## Reference Documentation
For advanced patterns, see: @${CLAUDE_SKILL_DIR}/references/implementation.md
```

The `@` prefix tells Claude to read the file. `${CLAUDE_SKILL_DIR}` resolves to the skill's directory.

### String Substitutions

| Syntax | Resolves To |
|--------|-------------|
| `$ARGUMENTS` | Everything after `/skill-name` |
| `$ARGUMENTS[0]` or `$1` | First positional argument |
| `$2`, `$3`, etc. | Second, third argument |
| `${CLAUDE_SKILL_DIR}` | Skill's directory path |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin root directory |
| `${CLAUDE_SESSION_ID}` | Current session identifier |
| `!`command`` | Runs command at activation, injects output |

### Dynamic Context Injection

Execute a command at skill activation and inject its output:

```markdown
---
name: git-summary
description: Summarize recent git activity
---

Current branch status:
!`git status --short`

Recent commits:
!`git log --oneline -10`

Analyze the above and provide a summary.
```

The `!`command`` syntax runs at activation time, not when Claude decides to — useful for injecting fresh context.

---

## 9. Scripts — Deterministic Logic

Scripts are the key to making skills reliable for complex workflows. They provide **deterministic computation** that doesn't depend on Claude's interpretation.

### Pattern: Skill + Python Script

```
data-analysis/
├── SKILL.md
└── scripts/
    ├── score.py         # Deterministic scoring logic
    ├── transform.py     # Data transformation
    └── validate.sh      # Input validation
```

**SKILL.md:**
```markdown
---
name: lead-scoring
description: Score leads based on engagement data. Use when processing lead lists.
allowed-tools: "Read, Write, Bash(python:*)"
---

## Instructions
1. Read the input CSV from $1
2. Run scoring: !`python ${CLAUDE_SKILL_DIR}/scripts/score.py $1`
3. Analyze the scored output
4. Generate recommendations based on scores
```

### When to Use Scripts vs Prompts

| Use Scripts For | Use Prompts For |
|----------------|-----------------|
| Mathematical calculations | Creative analysis |
| Data filtering/transformation | Summarization |
| Schema validation | Natural language generation |
| API calls with exact payloads | Interpretation of results |
| Deterministic business rules | Strategic recommendations |

### Script Best Practices

- Scripts read from stdin or file arguments, write to stdout
- Use JSON for structured data exchange between Claude and scripts
- Exit codes: 0 = success, non-zero = failure (Claude sees the error)
- Scripts can call external APIs (curl, requests, httpx)
- Always scope Bash in `allowed-tools`: `Bash(python:*)`, `Bash(node:*)`

---

## 10. Hooks — Event-Driven Automation

Hooks are shell scripts or commands that fire in response to Claude Code lifecycle events. They enable validation, enforcement, logging, and workflow automation.

### Hook Events

| Event | When It Fires | Use Case |
|-------|---------------|----------|
| `SessionStart` | Session begins or resumes | Load context, check prerequisites |
| `UserPromptSubmit` | User submits prompt, before Claude processes | Pre-validate, inject context |
| `PreToolUse` | Before a tool call executes | Validate, block dangerous operations |
| `PostToolUse` | After a tool call succeeds | Log, transform output, trigger follow-up |
| `PostToolUseFailure` | After a tool call fails | Error handling, retry logic |
| `SubagentStart` | Subagent spawned | Track parallel work |
| `SubagentStop` | Subagent finishes | Aggregate results |
| `Stop` | Claude finishes responding | Save state, cleanup |
| `PreCompact` | Before context compaction | Save important state |
| `PostCompact` | After compaction completes | Restore critical context |
| `SessionEnd` | Session terminates | Persist learnings, cleanup |
| `Notification` | Claude sends a notification | Route to Slack, email, etc. |
| `TaskCompleted` | Task marked complete | Trigger downstream work |

### Hook Types

| Type | What It Does |
|------|-------------|
| `command` | Execute shell commands or scripts |
| `http` | POST event JSON to a URL |
| `prompt` | Evaluate a prompt with an LLM |
| `agent` | Run an agentic verifier with tools |

### Hook Configuration (hooks.json)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/log-action.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/save-progress.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook Decision Flow (PreToolUse)

Hooks can block tool calls via exit codes:

| Exit Code | Meaning |
|-----------|---------|
| 0 | Allow — proceed normally |
| 2 | Deny — block the tool call, return reason to Claude |
| Other | Error — logged but doesn't block |

```bash
#!/bin/bash
# PreToolUse hook: block writes to protected files
file_path=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // empty')

if [[ "$file_path" == *"production.config"* ]]; then
  echo '{"decision": "deny", "reason": "Cannot modify production config without approval"}' >&2
  exit 2
fi
exit 0
```

### Memory Persistence via Hooks

Advanced pattern for cross-session memory:

```json
{
  "hooks": {
    "PreCompact": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "node ${CLAUDE_PLUGIN_ROOT}/scripts/save-state.js"
      }]
    }],
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "node ${CLAUDE_PLUGIN_ROOT}/scripts/persist-learnings.js"
      }]
    }],
    "SessionStart": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "node ${CLAUDE_PLUGIN_ROOT}/scripts/load-context.js"
      }]
    }]
  }
}
```

---

## 11. Sub-Agents & Parallel Execution

Sub-agents are parallel workers spawned within a session. They handle independent pieces of work simultaneously.

### How to Trigger

**Implicit** — describe parallel work naturally:
```
Analyze all CSV files in the /data folder. For each one, compute summary statistics,
detect anomalies, and write a structured markdown report. Process them in parallel.
```
Claude sees "for each" + "in parallel" and spawns sub-agents automatically.

**Explicit** — from SKILL.md:
```markdown
---
name: batch-analysis
description: Analyze multiple data sources in parallel
context: fork
---

Launch 3 parallel analysis agents:

Agent 1: Financial data (revenue.csv, costs.csv, margins.csv)
Agent 2: Customer data (segments.csv, churn.csv, nps.csv)
Agent 3: Operations data (inventory.csv, logistics.csv, quality.csv)

Each agent should:
1. Read the analysis template from @${CLAUDE_SKILL_DIR}/references/
2. Run the analysis script for each file
3. Write output to /outputs/{filename}-report.md
4. Update /progress-tracker.md when done
```

### Subagent Execution via Frontmatter

```yaml
---
name: research-category
context: fork          # Isolated subagent
agent: general-purpose # Agent type
model: opus            # Model override
allowed-tools: "Read, Write, Bash(python:*), mcp__ddg-search__search"
---
```

The `context: fork` flag runs the skill in a **forked subagent** with its own context window. This prevents one category's research from consuming the main session's context.

### Sub-Agent Limitations

- Shared filesystem (sub-agents can read/write same files — coordinate via progress tracker)
- Each sub-agent consumes context independently
- No direct inter-agent communication (use files as message passing)
- Claude sometimes does sequential instead of parallel (~90% reliability for parallel spawn)

---

## 12. Agent Teams — True Multi-Agent

Shipped February 2026. Multiple Claude Code instances coordinated with shared tasks.

```
Orchestrator (main session)
  ├── Agent: api-migration (worktree isolation)
  ├── Agent: test-writer (worktree isolation)
  ├── Agent: docs-updater (worktree isolation)
  └── Shared: task list, progress files
```

### Agent Team Coordination

```markdown
# Task Assignment: API Migration
---
agent_name: api-migration
task_number: 2
coordinator_session: main-orchestrator
dependencies: ["Task 1: Schema Design"]
---

Migrate REST endpoints from v1 to v2 format.

## Requirements
- Update all route handlers in /src/api/v1/ to v2 patterns
- Maintain backward compatibility with deprecation headers
- Write output to /outputs/migration-report.md

## Success Criteria
- All v2 endpoints pass integration tests
- No breaking changes to existing v1 consumers
- Report status to coordinator when complete
```

### Agent Teams vs Sub-Agents

| Feature | Sub-Agents | Agent Teams |
|---------|-----------|-------------|
| Context isolation | Forked (shared parent) | Full isolation (separate instances) |
| Parallelism | 3-5 concurrent | 5-30+ concurrent |
| Communication | Shared filesystem | Shared tasks + filesystem |
| Setup | Automatic from skill | Requires `/batch` or explicit config |
| Git isolation | Shared branch | Worktree per agent |
| Best for | Medium parallelism | Large-scale parallel work |

---

## 13. MCP Server Integration

Skills can use any configured MCP server. MCP tools are referenced by their full name: `mcp__<server>__<tool>`.

### In SKILL.md

```yaml
---
name: web-researcher
allowed-tools: "Read, Write, mcp__ddg-search__search, mcp__ddg-search__fetch_content"
---

## Research Process
1. Search for information: use mcp__ddg-search__search with targeted queries
2. Fetch full page content for key sources: use mcp__ddg-search__fetch_content
3. Synthesize findings into structured output
```

### Plugin-Bundled MCP Servers

Plugins can bundle their own MCP servers that start automatically:

```json
{
  "mcpServers": {
    "plugin-database": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {
        "DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data"
      }
    }
  }
}
```

### Available MCP Integrations (Common)

- **DuckDuckGo**: `mcp__ddg-search__search`, `mcp__ddg-search__fetch_content`
- **Playwright**: Browser automation (`browser_navigate`, `browser_snapshot`, `browser_click`, etc.)
- **Context7**: Documentation lookup (`resolve-library-id`, `query-docs`)
- **Custom**: Any MCP server you configure in `.mcp.json`

---

## 14. State & Memory Patterns

### Within-Session State

```markdown
## Progress Tracking
After completing each step, update the progress file:
Write to ${CLAUDE_SKILL_DIR}/../../outputs/progress.md:
- Step name
- Status (complete/failed/skipped)
- Key findings summary
```

### Cross-Session State via Hooks

**SessionStart hook** loads previous state:
```bash
#!/bin/bash
# Load previous progress if exists
if [ -f "./outputs/progress.md" ]; then
  echo "Previous session progress found. Categories completed:"
  grep "complete" ./outputs/progress.md
fi
```

**Stop hook** saves current state:
```bash
#!/bin/bash
# Save session summary
echo "Session ended at $(date)" >> ./outputs/session-log.md
echo "Last action: $CLAUDE_LAST_ACTION" >> ./outputs/session-log.md
```

### Memory File Pattern

For cross-session learning (not task state):

```
~/.claude/memory/
├── MEMORY.md           # Auto-loaded every session (keep under 200 lines)
├── skill-configs.md    # Skill-specific preferences
└── domain-knowledge.md # Accumulated domain knowledge
```

Skills can write to memory:
```markdown
After completing the task, save key learnings:
1. Which approaches worked best for this type of problem
2. Any edge cases or gotchas discovered
3. Configuration preferences the user expressed

Write to ~/.claude/memory/skill-learnings.md
```

---

## 15. Plugin Architecture

A plugin bundles multiple components into a distributable package:

```
my-plugin/
├── plugin.json          # Manifest — name, version, components
├── skills/              # Skills (SKILL.md directories)
├── agents/              # Subagent definitions
├── commands/            # Slash commands (legacy, skills preferred)
├── hooks/
│   └── hooks.json       # Event-driven automation
├── .mcp.json            # Bundled MCP servers
├── .lsp.json            # Language server configuration
└── scripts/             # Shared scripts
```

### plugin.json Manifest

```json
{
  "name": "code-quality-toolkit",
  "version": "1.0.0",
  "description": "Automated code review, linting, and security analysis",
  "author": "Your Name",
  "skills": ["skills/code-reviewer", "skills/security-scanner"],
  "agents": ["agents/test-writer"],
  "hooks": "hooks/hooks.json"
}
```

### Installation Scopes

| Scope | Settings File | Use Case |
|-------|--------------|----------|
| `user` | `~/.claude/settings.json` | Personal (default) |
| `project` | `.claude/settings.json` | Team (version controlled) |
| `local` | `.claude/settings.local.json` | Project-specific (gitignored) |
| `managed` | Enterprise managed settings | Org-wide (read-only) |

### Plugin CLI

```bash
# Install from marketplace
/plugin marketplace add author/plugin-name
/plugin install plugin-name

# Install from local directory
/plugin install ./path/to/plugin

# Manage
/plugin list
/plugin disable plugin-name
/plugin enable plugin-name
/plugin uninstall plugin-name
```

---

## 16. Skill Composition Patterns

### Pattern 1: Orchestrator + Worker Skills

Main skill delegates to specialized sub-skills:

```markdown
---
name: full-migration
description: Run complete API migration pipeline across all modules
disable-model-invocation: true
---

## Orchestration Steps

1. Validate inputs using /validate-migration-config
2. For each module in the migration plan:
   a. Run /migrate-module with module name
   b. Save output to /outputs/{module}-migration.md
   c. Update progress tracker
3. Run /migration-verification on all outputs
4. Generate summary report
```

### Pattern 2: Skill + Agent Handoff

Skill detects the need, agent does the deep work:

```markdown
---
name: quality-checker
description: Checks code output quality automatically
---

## Quality Checks
1. Verify test coverage meets minimum threshold
2. Check for security vulnerabilities (OWASP top 10)
3. Validate API contract compliance

If issues found, recommend: @security-auditor agent for targeted follow-up
```

### Pattern 3: Multi-Phase Workflow

```markdown
---
name: comprehensive-review
description: Comprehensive review workflow
allowed-tools: "Bash(node:*), Read"
---

Target: @$1

Phase 1 - Static Analysis:
!`node ${CLAUDE_SKILL_DIR}/scripts/lint.js $1`

Phase 2 - Deep Review:
Launch code-reviewer agent for detailed analysis.

Phase 3 - Standards Check:
Use coding-standards skill for validation.

Phase 4 - Report:
Template: @${CLAUDE_SKILL_DIR}/templates/review.md
Compile findings into report following template.
```

### Pattern 4: Conditional Skill Loading

```markdown
---
name: smart-research
description: Research with adaptive depth based on topic complexity
---

## Decision Tree

1. Assess topic complexity:
   - Simple fact-finding -> Quick search, 2-3 sources
   - Technical deep-dive -> Deep search, 5-10 sources, page fetches
   - Comprehensive audit -> Exhaustive search, verify against official docs

2. Based on complexity:
   - Quick: Use mcp__ddg-search__search directly
   - Deep: Run @${CLAUDE_SKILL_DIR}/scripts/deep-search.py
   - Exhaustive: Spawn 3 sub-agents with different search strategies
```

---

## 17. Engineering Complex Robust Skills

### Principle 1: Deterministic Core, Creative Shell

Put deterministic logic in scripts, creative work in prompts:

```
┌─────────────────────────────────────┐
│  SKILL.md (Creative Shell)          │
│  - Orchestration decisions          │
│  - Natural language generation      │
│  - Research synthesis               │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  scripts/ (Deterministic)   │    │
│  │  - Data filtering           │    │
│  │  - Schema validation        │    │
│  │  - API calls with payloads  │    │
│  │  - Business rule evaluation │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

### Principle 2: Fail Gracefully at Every Step

```markdown
## Error Recovery

After each task step completes:
1. Verify output file exists and meets minimum criteria
2. If verification fails:
   a. Log failure to progress-tracker.md
   b. Continue with next step (don't block)
   c. At end, report which steps need retry
3. Never silently skip — always record what happened
```

### Principle 3: Checkpoint Everything

```markdown
## Checkpointing

After each major step, write to progress-tracker.md:
- timestamp
- step completed
- output file path
- word count
- status (success/partial/failed)

On session resume, read progress-tracker.md first.
Only run steps not marked as "success".
```

### Principle 4: Scope Blast Radius

Use `context: fork` for expensive sub-operations so they don't consume the main context:

```yaml
---
name: research-category
context: fork
model: opus
allowed-tools: "Read, Write, mcp__ddg-search__search, mcp__ddg-search__fetch_content"
---
```

Each forked sub-agent gets its own 200K context window.

### Principle 5: Template Prompts, Not Freeform

Instead of relying on Claude to generate consistent prompts, bundle exact templates:

```markdown
## Report Generation

For each section:
1. Load prompt template: @${CLAUDE_SKILL_DIR}/references/section-{id}.md
2. Run data filter: !`python ${CLAUDE_SKILL_DIR}/scripts/filter-data.py {id}`
3. Run formatter: !`python ${CLAUDE_SKILL_DIR}/scripts/format-context.py {mode}`
4. Inject filtered data and formatted context into the loaded template
5. Execute generation using the assembled prompt
```

### Principle 6: Validate Outputs

```bash
#!/bin/bash
# scripts/validate-output.sh
file="$1"
min_words=500

if [ ! -f "$file" ]; then
  echo "FAIL: Output file $file does not exist"
  exit 1
fi

word_count=$(wc -w < "$file")
if [ "$word_count" -lt "$min_words" ]; then
  echo "WARN: $file has only $word_count words (minimum: $min_words)"
  exit 1
fi

# Check for required sections
sections=$(grep -c "^## " "$file")
if [ "$sections" -lt 3 ]; then
  echo "WARN: Only $sections sections found (minimum: 3)"
  exit 1
fi

echo "PASS: $file — $word_count words, $sections sections"
exit 0
```

### Principle 7: Keep SKILL.md Under 200 Lines

```
SKILL.md (orchestration)  ←  Under 200 lines
  ├── references/impl.md  ←  Heavy documentation
  ├── references/prompts/  ←  Category prompt templates
  ├── scripts/             ←  Deterministic logic
  └── templates/           ←  Output templates
```

---

## 18. Testing & Validation

### Manual Testing

```bash
# Test skill activation
/skill-name test-argument

# Test scripts independently
cd ~/.claude/skills/my-skill/
python scripts/transform-data.py --input test-data/sample.json
bash scripts/validate-output.sh test-data/sample-output.md
```

### Hook Testing Framework

```bash
# Validate hooks.json structure
./validate-hook-schema.sh hooks/hooks.json

# Test individual hook scripts
./test-hook.sh scripts/my-hook.sh test-input.json

# Test with verbose output
./test-hook.sh -v -t 30 scripts/my-hook.sh test-input.json

# Generate sample test input
./test-hook.sh --create-sample PreToolUse > test-input.json
```

### Eval Files

```json
// evals/evals.json
[
  {
    "name": "basic_report",
    "input": "/generate-report quarterly-sales",
    "expected_outputs": ["quarterly-sales-report.md"],
    "min_word_count": 1000,
    "required_sections": ["Executive Summary", "Key Metrics", "Recommendations"]
  }
]
```

---

## 19. Distribution & Marketplace

### Publishing to Marketplace

1. **Package:** Ensure `plugin.json` manifest is complete
2. **Validate:** Run marketplace validator (100-point scoring)
3. **Submit:** Via `/plugin marketplace submit` or [submission form](https://clau.de/plugin-directory-submission)
4. **Install:** Users run `/plugin marketplace add author/plugin-name`

### Marketplace Scoring (100 Points)

| Category | Points | What's Checked |
|----------|--------|---------------|
| Frontmatter | 15 | Required fields, description quality |
| Body sections | 16 | Overview, Prerequisites, Instructions, Output, Error Handling, Examples |
| Code quality | 15 | Code blocks, practical examples |
| Progressive disclosure | 7 | SKILL.md size, references/, scripts/ |
| Identity | 10 | Version, author, license, tags |
| Compatibility | 5 | Platform declarations |

Score 80+: Featured. Score 60-79: Listed. Score <60: Needs improvement.

---

## 20. Limitations & Failure Modes

### Hard Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| **No cross-session state** | Skills start fresh each session | Use hooks (SessionStart/Stop) + local files |
| **Context window** | 200K tokens shared across all activity | Use `context: fork` for heavy sub-tasks |
| **No output length control** | Can't cap Claude's response to exact token count | Validate output length in post-processing script |
| **Tool call reliability** | ~90% per call, compounds across chains | Checkpoint after each step, validate, retry failed steps |
| **Search depth** | DDG MCP returns snippets, not full pages | Use `fetch_content` for key sources, limit to 3-5 per task |
| **Parallel reliability** | Sub-agent spawning ~90% parallel (sometimes sequential) | Explicit instructions: "Process these in parallel — do NOT do them one at a time" |

### Common Failure Modes

| Failure | Cause | Prevention |
|---------|-------|------------|
| Skill not triggering | Vague description | Specific trigger phrases, forced eval hook |
| Wrong skill triggers | Overlapping descriptions | Distinct keywords, non-overlapping descriptions |
| Skill ignores instructions | Body too long (>500 lines) | Extract to references/, keep body under 200 lines |
| Script not found | Wrong path | Use `${CLAUDE_SKILL_DIR}` or `${CLAUDE_PLUGIN_ROOT}` |
| Output inconsistency | Prompt variation between runs | Template-based prompts in references/, not freeform |
| Context overflow | Too much data loaded | Progressive disclosure, `context: fork` for sub-tasks |
| Partial completion | Session timeout or error | Checkpoint every step, Stop hook saves progress |

### Compounding Reliability Math

For a skill with N sequential tool calls at R reliability each:

```
P(all succeed) = R^N

Example: 45 tool calls at 90% each
P = 0.90^45 = 0.87% (perfect execution unlikely)

Mitigation: checkpoint after each step, validate, retry failures
Effective reliability with retry: ~95% per step -> 0.95^45 = 10%
Effective reliability with checkpoint + resume: ~99% (most steps succeed, failures retried)
```

**Bottom line:** Complex skills need checkpoint-and-resume architecture, not pray-for-perfect-execution.
