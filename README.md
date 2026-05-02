# Claude Code — Complete Feature Guide

> Reference for how every Claude Code feature works, why it exists, and when/how to use it.
> Examples use this project (6Valley Laravel eCommerce) for context.

---

## Table of Contents

1. [CLI Usage](#1-cli-usage)
2. [CLAUDE.md — Project Instructions](#2-claudemd)
3. [Rules System](#3-rules-system)
4. [Memory System](#4-memory-system)
5. [Slash Commands](#5-slash-commands)
6. [Skills — Custom Commands](#6-skills)
7. [Sub-agents](#7-sub-agents)
8. [MCP Servers](#8-mcp-servers)
9. [Hooks](#9-hooks)
10. [Settings & Permissions](#10-settings--permissions)
11. [Keyboard Shortcuts](#11-keyboard-shortcuts)
12. [Token & Cost Management](#12-token--cost-management)
13. [IDE Integration](#13-ide-integration)

---

## 1. CLI Usage

### What & Why
The `claude` CLI is how you invoke Claude Code outside an IDE. Supports interactive sessions, one-shot print mode, session continuity, and programmatic scripting.

### Invocation Modes

| Mode | Command | Use Case |
|------|---------|----------|
| Interactive | `claude` | Normal development work |
| With prompt | `claude "fix the auction controller"` | Start with a task |
| Print (non-interactive) | `claude -p "summarize changes"` | CI scripts, piping output |
| Continue last | `claude -c` | Resume where you left off |
| Resume specific | `claude -r "session-name"` | Go back to named session |

**Example — Print mode in CI:**
```bash
git diff HEAD~1 | claude -p "review this diff for security issues" --output-format json
```

**Example — Continue after closing terminal:**
```bash
claude -c   # Picks up the last conversation
```

### Key Flags

```bash
--model sonnet              # Use a specific model (cheaper)
--model opus                # Use most capable model
--add-dir ../Modules/AI     # Include another directory in context
--permission-mode acceptEdits   # Auto-accept file edits, no prompts
--permission-mode plan      # Read-only planning mode
--permission-mode bypassPermissions  # Skip all prompts (use carefully)
--max-turns 10              # Limit how many steps Claude can take
--max-budget-usd 0.50       # Stop before spending $0.50
--verbose                   # Show tool calls and reasoning
--system-prompt "You are a Laravel expert" # Override system prompt
```

**Example — Automated task with budget cap:**
```bash
claude -p "refactor the AuctionController to use the repository pattern" \
  --max-budget-usd 0.30 \
  --permission-mode acceptEdits
```

**Example — Add Auction module context to a non-module session:**
```bash
claude --add-dir Modules/Auction "why is the bid placement failing?"
```

### Worktrees (Parallel Work)
```bash
claude -w feature-payment    # Start in isolated git worktree
claude -w bugfix-auction     # Parallel session on same repo
```
Each worktree is independent. No conflicts between parallel Claude sessions.

---

## 2. CLAUDE.md

### What & Why
CLAUDE.md is the instruction file Claude reads at the start of every session. It defines project context, coding standards, commands, and rules. Replaces writing context in every prompt.

### Loading Hierarchy

```
/Library/Application Support/ClaudeCode/CLAUDE.md   → Managed (org-wide, IT-deployed)
~/.claude/CLAUDE.md                                  → User (all your projects)
./CLAUDE.md  or  ./.claude/CLAUDE.md                → Project (committed, shared)
./CLAUDE.local.md                                    → Local (gitignored, personal)
Modules/Auction/CLAUDE.md                            → Module-level (loads when working there)
```

**Loading rule:** Walks up from working directory. More specific = higher priority. Nested module CLAUDE.md only loads when Claude reads files in that directory.

### When to Use What Level

| Level | Use for |
|-------|---------|
| Project `CLAUDE.md` | Team-wide conventions, architecture, commands |
| `CLAUDE.local.md` | Your personal preferences, local paths, secrets |
| `~/.claude/CLAUDE.md` | Global habits across all projects |
| `Modules/X/CLAUDE.md` | Module-specific rules (only loads in-module) |

### `@` Directive — Import Files

```markdown
@.claude/rules/translate.md       # Import a rule file
@.claude/rules/response-style.md  # Import response style
@README.md                        # Import README into context
@package.json                     # Import package info
```

**Problem it solves:** Keeps CLAUDE.md short while pulling in focused rule files only when needed.

### Size Guideline
Keep under 200 lines. Beyond that, Claude's adherence drops and token cost rises.

**Bad (too much):**
```markdown
## Architecture
[500 lines of architecture docs...]
```

**Good:**
```markdown
## Architecture
Flow: Controller → Service → Repository → Model
Routes: non-standard, see app/Providers/RouteServiceProvider.php
Full architecture: AI_MODEL_PROJECT_CONTEXT.md (read on demand)
```

### Exclude Files You Don't Want
`.claude/settings.local.json`:
```json
{
  "claudeMdExcludes": ["**/SomeModule/CLAUDE.md"]
}
```

---

## 3. Rules System

### What & Why
Rule files in `.claude/rules/` are focused instruction files that can be scoped to specific file types. Solves the problem of global rules that don't apply to everything.

### Structure
```
.claude/rules/
├── translate.md          # Blade translation (always loaded via @)
├── response-style.md     # Response compression (always loaded via @)
├── api-conventions.md    # REST API patterns
└── frontend/
    └── vue-patterns.md   # Vue 2 component rules
```

### Scoped Rules (Only Load for Matching Files)

```markdown
---
paths:
  - "resources/views/**/*.blade.php"
  - "resources/js/**/*.vue"
---

# Frontend-only rules here
Only apply these rules when editing Blade or Vue files.
```

**Why this matters:** A rule about Blade translations doesn't need to load when you're editing a migration file. Path-scoped rules save tokens and reduce noise.

### Always-loaded vs. On-demand

| Type | How | When Loads |
|------|-----|-----------|
| `@.claude/rules/foo.md` in CLAUDE.md | Direct import | Every request |
| Path-scoped `---paths:---` | Automatic | When Claude reads matching file |
| No paths + no import | Never (orphaned) | Never — must be imported |

### Example — Project-specific API rule

`.claude/rules/auction-api.md`:
```markdown
---
paths:
  - "Modules/Auction/**/*"
---

# Auction API Conventions
- All auction API responses must include `auction_status` field
- Use AuctionRepository, never query AuctionProduct directly
- Bid validation lives in AuctionBidService, not controllers
```

This only loads when editing files inside `Modules/Auction/`.

---

## 4. Memory System

### What & Why
Auto memory is Claude's persistent notepad across sessions. It stores things Claude learns about you and your project so you don't repeat yourself.

### File Structure
```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md             # Index — first 200 lines load at startup
├── feedback_no_coauthor.md
├── feedback_response_style.md  ← moved to .claude/rules/ (visible in IDE)
└── user_preferences.md
```

### Memory Types

| Type | Purpose | Example |
|------|---------|---------|
| `user` | Who you are, your skill level | "Senior Laravel developer, new to Vue" |
| `feedback` | How Claude should behave | "Never add Co-Authored-By in commits" |
| `project` | Ongoing decisions & deadlines | "Merge freeze starts 2026-05-10 for v16.4" |
| `reference` | Where to find things | "Bug tracker is Linear project AUCTION" |

### Memory File Format
```markdown
---
name: No Co-Authored-By in commits
description: Never append attribution trailers when running /push
type: feedback
---

Never include "Co-Authored-By" in commits via /push.

**Why:** User doesn't want automated tags cluttering commit history.
**How to apply:** Commit message = one short subject line. No body, no trailers.
```

### Rules vs. Memory — When to Use Which

| | Rules (`.claude/rules/`) | Memory (`~/.claude/.../memory/`) |
|---|---|---|
| Visible in IDE | Yes | No |
| Version controlled | Yes | No |
| Shared with team | Yes | No |
| Loaded | Every request (if @-imported) | By relevance |
| Use for | Project standards | Personal behavioral feedback |

**Guideline:** If you want a rule to apply to the whole team → `.claude/rules/`. If it's personal feedback ("Claude keeps doing X wrong for me") → memory.

### `/memory` Command
Opens memory editor in Claude Code. Toggle auto memory on/off, view/edit all topic files.

---

## 5. Slash Commands

### What & Why
Built-in `/commands` are shortcuts for common Claude Code operations. Type `/` to see all available.

### Built-in Commands — Most Useful

| Command | What it does |
|---------|-------------|
| `/clear` | Start fresh conversation (old one saved for `/resume`) |
| `/compact [instructions]` | Summarize context, free up tokens |
| `/cost` or `/usage` | Show token usage and estimated cost |
| `/model sonnet` | Switch to Sonnet (cheaper) |
| `/model opus` | Switch to Opus (most capable) |
| `/memory` | View/edit memory files |
| `/init` | Auto-generate CLAUDE.md from codebase scan |
| `/doctor` | Diagnose configuration issues |
| `/status` | Show active settings sources |
| `/thinking` | Toggle extended thinking (deeper reasoning) |
| `/vim` | Toggle vim keybindings |
| `/mcp` | Manage MCP servers + OAuth |
| `/todo` | Show/manage task list |
| `/rewind` | Undo Claude's last action |
| `/review` | Review current PR (skill) |

### Scenario Examples

**Problem: Context getting stale mid-session:**
```
/compact Focus on the auction bid placement flow
```
Summarizes everything, re-injects CLAUDE.md, continues with fresh context.

**Problem: Don't know what model is active:**
```
/status
```
Shows model, permission mode, settings sources.

**Problem: Made a mistake, want to undo:**
```
/rewind
```
Restores files to pre-edit state. Double-press `Esc` also triggers.

**Problem: Session got expensive, want to check:**
```
/cost
```
Shows per-session token count and dollar estimate.

---

## 6. Skills

### What & Why
Skills are custom `/commands` you define in markdown files. Package complex, repeatable workflows into a single `/command`. Live in `.claude/commands/` or `.claude/skills/`.

### File Locations

| Scope | Path | Shared |
|-------|------|--------|
| Project | `.claude/commands/<name>.md` | Yes (git) |
| Project (new format) | `.claude/skills/<name>/SKILL.md` | Yes (git) |
| User (all projects) | `~/.claude/skills/<name>/SKILL.md` | No |

### Example — Create a `/deploy-check` skill

`.claude/commands/deploy-check.md`:
```markdown
Run the following pre-deployment checklist for this Laravel project:

1. Run `php artisan config:cache` and verify no errors
2. Run `php artisan route:cache` and verify no errors
3. Check `git status` — warn if uncommitted changes
4. Run `php artisan migrate --pretend` and list pending migrations
5. Check `.env` for `APP_DEBUG=false` and `APP_ENV=production`

Report: pass/fail for each item. List any warnings.
```

Usage: `/deploy-check` — runs the full checklist instantly.

### Skill Frontmatter Options

```yaml
---
description: When to suggest this skill to Claude automatically
argument-hint: [branch] [environment]   # Autocomplete hint
allowed-tools: Bash(php artisan *) Read  # Pre-approve these tools
model: sonnet                           # Use cheaper model for this skill
---
```

### Skill with Arguments

`.claude/commands/seed-module.md`:
```markdown
Seed the $ARGUMENTS module with test data by:
1. Running `php artisan db:seed --class=$ARGUMENTS`Seeder
2. Verifying 5+ records exist in the table
3. Reporting row counts
```

Usage: `/seed-module Auction` → seeds the Auction module.

### When to Create a Skill

- Task you repeat more than 3× per week
- Multi-step workflow that's hard to remember
- Process that requires specific tool permissions
- Team convention you want enforced consistently

---

## 7. Sub-agents

### What & Why
Sub-agents are specialized Claude instances spawned for specific tasks. Protect the main context window from large, verbose outputs. Enable parallel work.

### Built-in Agent Types

| Type | Specialization | Use when |
|------|---------------|----------|
| `Explore` | Read-only search | "Find all places X is called" |
| `Plan` | Architecture design | "Design the bid refund flow" |
| `general-purpose` | Default, all tools | Complex multi-step tasks |
| `claude-code-guide` | Claude Code questions | "How does MCP work?" |

### When to Use Sub-agents

**1. Protecting context from verbose output:**
```
Spawn an Explore agent to find all Auction-related routes.
```
The route list stays in the sub-agent's context, not yours.

**2. Parallel independent work:**
```
Agent 1: Audit Auction routes for missing auth middleware
Agent 2: Check all Auction migrations for missing indexes
```
Both run simultaneously.

**3. Worktree isolation:**
Agent with `isolation: "worktree"` gets its own git branch. Experiments don't affect your working tree. Auto-cleaned if no changes made.

### Foreground vs. Background

| | Foreground | Background |
|---|---|---|
| Blocks your session | Yes | No |
| Result shown | Full response | Summary only |
| Use when | You need results before continuing | Independent parallel task |

### Sub-agent Practical Example

**Problem:** You want to audit all 244 files in `Modules/Auction/` for a specific pattern without filling your context.

**Without agent:** Reading 244 files fills context, slows your session.

**With agent:**
```
Spawn an Explore agent: search Modules/Auction/ for any direct DB::table() calls 
that bypass the repository pattern. Report file:line for each one found.
```
Agent searches, returns a compact report. Your context stays clean.

---

## 8. MCP Servers

### What & Why
Model Context Protocol (MCP) connects Claude to external tools and services — databases, APIs, browsers, file systems. Extends Claude beyond the default tools.

### How It Works
MCP server = an external process that exposes tools/resources to Claude. Claude calls these tools like any built-in tool (Bash, Read, Edit).

### Add an MCP Server

```bash
# Local stdio server (script/binary)
claude mcp add my-db -- node /path/to/db-server.js

# Remote HTTP server
claude mcp add github --transport http https://mcp.github.com

# With environment variables
claude mcp add my-server --env DB_URL=mysql://... -- ./server.sh
```

### Scopes

```bash
claude mcp add --scope project my-server ...  # .mcp.json (committed)
claude mcp add --scope user my-server ...     # ~/.claude.json (personal)
claude mcp add --scope local my-server ...    # Not shared (default)
```

### Useful MCP Servers for This Project

| Server | What it gives Claude | Install |
|--------|---------------------|---------|
| `github` | Create PRs, review issues, manage releases | `claude mcp add github --transport http https://mcp.github.com` |
| `filesystem` | Expanded file operations | `claude mcp add filesystem -- npx @modelcontextprotocol/server-filesystem .` |
| `mysql` | Query the database directly | Custom server |

### MCP in Practice — Example

**Problem:** You want Claude to check the database for auction records instead of asking you.

**Solution:** Install a MySQL MCP server:
```bash
claude mcp add mysql -- node mysql-mcp-server.js --connection "mysql://root:pw@localhost/6valley"
```

Now Claude can run:
```
Check the auctions table for any bids placed in the last 24 hours 
where payment_status is null.
```
Claude queries the DB directly. No copy-pasting SQL results.

### Manage MCP Servers
```bash
claude mcp list        # List all configured servers
claude mcp get github  # Show server details
claude mcp remove name # Remove a server
```
Or use `/mcp` in Claude Code for GUI + OAuth management.

### MCP Resources (`@` mentions)
```
@github:issue://123     # Reference a GitHub issue
@filesystem:./README.md # Reference a file via MCP
```
Type `@` in the prompt to see all available resources.

---

## 9. Hooks

### What & Why
Hooks run shell commands (or HTTP/MCP calls) automatically at specific Claude lifecycle events. Automate validation, notifications, logging, or enforcement without manual intervention.

### Lifecycle Events

| Event | When it fires |
|-------|--------------|
| `PreToolUse` | Before Claude calls any tool |
| `PostToolUse` | After tool completes |
| `UserPromptSubmit` | When you send a message |
| `Stop` | When Claude finishes responding |
| `SessionStart` | When session begins |
| `SessionEnd` | When session ends |
| `FileChanged` | When a file is changed |

### Config Location

`.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/validate.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook Exit Codes

| Exit code | Behavior |
|-----------|---------|
| `0` | Success, parse JSON output if present |
| `2` | **Block the action** — stderr sent to Claude as explanation |
| Other | Continue, show stderr to user |

### Practical Hook Examples

**1. Block dangerous Artisan commands:**

`.claude/hooks/block-migrate-fresh.sh`:
```bash
#!/bin/bash
# $CLAUDE_TOOL_INPUT contains the command
if echo "$CLAUDE_TOOL_INPUT" | grep -q "migrate:fresh\|migrate:reset"; then
  echo "Blocked: migrate:fresh/reset destroys all data. Use migrate instead." >&2
  exit 2
fi
exit 0
```

`.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/block-migrate-fresh.sh"
        }]
      }
    ]
  }
}
```

**2. Auto-run lint after file edits:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "php artisan lint 2>&1 || true"
        }]
      }
    ]
  }
}
```

**3. Notify on session end (Slack/desktop):**
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "osascript -e 'display notification \"Claude finished\" with title \"Claude Code\"'"
        }]
      }
    ]
  }
}
```

**4. Log all Bash commands Claude runs:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "echo \"$(date): $CLAUDE_TOOL_INPUT\" >> ~/.claude/bash-audit.log"
        }]
      }
    ]
  }
}
```

### Matcher Patterns

```
"Bash"              # Exact match
"Edit|Write"        # Multiple tools
"^mcp__"            # Any MCP tool (regex)
"Bash(git push *)"  # Specific bash command pattern
""  or omit         # Match everything
```

---

## 10. Settings & Permissions

### What & Why
Settings control model, permissions, hooks, env vars, and behavior. Permissions control which tools Claude can use without asking.

### Settings File Hierarchy

```
Managed (IT)         → always wins, can't override
  ↓
Command-line flags
  ↓
.claude/settings.local.json   (local, not committed)
  ↓
.claude/settings.json         (project, committed)
  ↓
~/.claude/settings.json       (user, all projects)   ← lowest priority
```

### Core Settings Structure

`.claude/settings.json`:
```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Bash(php artisan *)",
      "Bash(composer *)",
      "Bash(npm run *)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(git status)",
      "Read(**)"
    ],
    "deny": [
      "Bash(php artisan migrate:fresh)",
      "Bash(php artisan migrate:reset)",
      "Bash(rm -rf *)"
    ]
  }
}
```

### Permission Modes

| Mode | What gets auto-approved | Prompt for |
|------|------------------------|-----------|
| `default` | Nothing | Everything |
| `acceptEdits` | All file edits | Bash, web |
| `plan` | Nothing (read-only) | Everything (view only) |
| `bypassPermissions` | Everything | Nothing |

**Switch during session:** `Shift+Tab` cycles through modes. Or `/model` then select.

### Permission Rule Syntax

```json
"allow": [
  "Bash(npm *)",            // All npm commands
  "Bash(git log *)",        // git log only
  "Read(./src/**)",         // Read anything in src/
  "Edit(./Modules/**)",     // Edit anything in Modules/
  "WebFetch(domain:packagist.org)"  // Specific domain
]
```

**Evaluation order: Deny → Ask → Allow** (first match wins)

### What to Put in Project vs. Local Settings

| Setting | `.claude/settings.json` (commit) | `.claude/settings.local.json` (local) |
|---------|----------------------------------|--------------------------------------|
| Allowed artisan commands | Yes | |
| Denied dangerous commands | Yes | |
| Personal API keys | | Yes |
| Your preferred model | | Yes |
| Team-wide hooks | Yes | |
| Personal shortcuts | | Yes |

### `/status` — Check Active Config
```
/status
```
Shows which settings files are loaded, active model, permission mode, and origins of each setting.

---

## 11. Keyboard Shortcuts

### Core Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel/interrupt current response |
| `Ctrl+D` | Exit Claude Code |
| `Shift+Tab` | Cycle permission modes |
| `Esc Esc` | Rewind last action |
| `Alt+T` / `Option+T` | Toggle extended thinking |
| `Alt+O` / `Option+O` | Toggle fast mode |
| `Alt+P` / `Option+P` | Switch model |
| `Ctrl+O` | Toggle transcript viewer |
| `Ctrl+T` | Toggle task list |
| `Ctrl+R` | Search command history |
| `Ctrl+L` | Redraw/clear screen |

### Text Editing

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Jump to start of line |
| `Ctrl+E` | Jump to end of line |
| `Ctrl+K` | Delete to end of line |
| `Ctrl+W` | Delete previous word |
| `Ctrl+U` | Delete to start of line |
| `Alt+B` | Move back one word |
| `Alt+F` | Move forward one word |

### Multiline Input

| Method | How |
|--------|-----|
| Quick | `\ + Enter` |
| macOS | `Option+Enter` |
| Universal | `Shift+Enter` |

### @ and ! Shortcuts

- `@filename` — Attach a file to your message (fuzzy search)
- `!command` — Run a shell command directly (e.g., `!git status`)
- `/command` — Run a skill or slash command

---

## 12. Token & Cost Management

### Why It Matters
Each request loads: system prompt (~4k), CLAUDE.md + rules (~500), memory (~200), conversation history, and your message. Optimizing each layer compounds into large savings.

### What Consumes Tokens

| Source | Est. Tokens | Controllable? |
|--------|------------|--------------|
| System prompt | ~3,000-4,500 | No |
| CLAUDE.md + @rules | ~200-500 | Yes — keep lean |
| Memory (MEMORY.md) | ~200-300 | Yes — keep index short |
| MCP tool schemas | ~800-1,200 | Yes — disable unused servers |
| Conversation history | Grows | Yes — `/compact` or `/clear` |
| `@Module/` directives | ~5,000-30,000 | **Yes — never use `@Directory`** |

### Reduce Per-Request Cost

1. **Never `@Directory` in CLAUDE.md** — 39k lines × every request = catastrophic
2. **Use nested CLAUDE.md** — `Modules/Auction/CLAUDE.md` only loads in that module
3. **Path-scope rules** — `.claude/rules/frontend.md` with `paths: [*.blade.php]` skips load on backend tasks
4. **Disable unused MCP servers** — Each adds schema tokens
5. **Keep CLAUDE.md under 200 lines** — Adherence drops + cost rises beyond that
6. **Use `sonnet` for most tasks** — `opus` only for complex reasoning tasks

### Mid-Session Cost Control

```
/cost          # Check current session cost
/compact       # Summarize + free context (preserves CLAUDE.md)
/clear         # Full reset (start fresh)
/model sonnet  # Switch to cheaper model mid-session
```

### Model Cost Comparison

| Model | Relative Cost | Best for |
|-------|--------------|----------|
| Haiku 4.5 | Cheapest | Sub-agents, search, simple tasks |
| Sonnet 4.6 | Mid | Most development tasks |
| Opus 4.7 | Most expensive | Complex reasoning, architecture |

**Practical pattern:** Use Sonnet for development. Spawn Haiku sub-agents for large searches/audits. Use Opus only for architecture decisions.

### Budget Cap (Non-interactive)
```bash
claude -p "refactor AuctionBidService" --max-budget-usd 0.25
```
Stops before exceeding budget. Useful for automated scripts.

### Prompt Caching
Claude automatically caches your system prompt + CLAUDE.md across sessions. First request costs full price; subsequent requests reuse cache at ~10% of the cost. Cache TTL: 5 minutes in active conversation, up to 1 week for static content.

---

## 13. IDE Integration

### VS Code Extension

**Open Claude Code panel:** Click spark icon in top-right toolbar, or `Cmd+Esc`.

**Key features:**
- Side-by-side diff review with permission prompt
- `Option+K` — Insert `@file.php#10-25` reference with line range
- `Shift+Enter` — Multiline input
- Permission mode selector in prompt bar
- Session history (local + remote)
- Multiple conversations in tabs

**VS Code-specific settings** (VS Code settings, not `.claude/`):
```json
{
  "claudeCode.initialPermissionMode": "acceptEdits",
  "claudeCode.useCtrlEnterToSend": true,
  "claudeCode.autosave": true
}
```

### JetBrains (PhpStorm for this project)

**Open:** `Cmd+Esc` from editor, or Claude Code button in toolbar.

**Key features:**
- IDE lint/error diagnostics auto-shared with Claude
- `Cmd+Option+K` — Insert `@file#L1-99` reference
- Diff viewer integrated into IDE

**If ESC doesn't interrupt Claude:**
Settings → Tools → Terminal → uncheck "Move focus to the editor with Escape"

### CLI Inside IDE Terminal
Both IDEs have integrated terminals. Use `claude` CLI there for features not in the panel:
- Worktrees (`claude -w`)
- Custom flags
- MCP management (`claude mcp add`)
- Sub-agent spawning

### Connect IDE to External Terminal Session
```
/ide   # In Claude Code CLI → connects to open VS Code/JetBrains
```
Gives the CLI session file navigation and diff viewing through the IDE.

---

## Quick Reference — "What do I use for X?"

| Goal | Tool |
|------|------|
| Always-on coding standard | `CLAUDE.md` or `.claude/rules/` with `@` import |
| Standard only for Blade files | `.claude/rules/foo.md` with `paths: [*.blade.php]` |
| Standard only for Auction module | `Modules/Auction/CLAUDE.md` |
| Personal preference (no commit) | `CLAUDE.local.md` or memory |
| Repeatable multi-step workflow | Skill in `.claude/commands/` |
| Block a dangerous command | Hook with exit code 2 |
| Auto-run lint after edits | `PostToolUse` hook |
| Search 200+ files without filling context | Explore sub-agent |
| Query DB without copy-pasting | MCP MySQL server |
| Reduce permission prompts | `permissions.allow` in `.claude/settings.json` |
| Resume yesterday's session | `claude -c` or `-r session-name` |
| Check cost mid-session | `/cost` |
| Free context without losing rules | `/compact` |
| Undo Claude's last file change | `/rewind` or `Esc Esc` |
| Run task with budget cap | `claude -p "..." --max-budget-usd 0.50` |
