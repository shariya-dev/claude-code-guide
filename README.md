# Claude Code — Complete Feature Guide

> Reference for every Claude Code concept, when to use each, and how to structure your project.
> Examples use a Laravel eCommerce project (6Valley) for context.

---

## Mental Model First: The Six Building Blocks

Before diving into features, understand what each layer is *for*:

| Building Block | Purpose | Invocation |
|---|---|---|
| **Rules** | Passive knowledge Claude always applies | Automatic |
| **Skills** | Multi-step workflows you invoke on demand | `/skill-name` |
| **Built-in Commands** | Control Claude Code's own session state | `/clear`, `/cost`, etc. |
| **Hooks** | Automatic enforcement at lifecycle events | Triggered by events |
| **MCP Servers** | Live access to external systems | Via tools |
| **Sub-agents** | Isolated workers for parallel/large tasks | Spawned on demand |

**Decision rule:**

```
Always-on knowledge?           → Rule
On-demand workflow?            → Skill
Auto-runs without asking?      → Hook
External system access?        → MCP Server
Parallel/large output task?    → Sub-agent
Session control?               → Built-in slash command
```

---

## Table of Contents

1. [CLI Usage](#1-cli-usage)
2. [CLAUDE.md — Project Instructions](#2-claudemd)
3. [Rules System](#3-rules-system)
4. [Memory System](#4-memory-system)
5. [Skills — Custom Slash Commands](#5-skills--custom-slash-commands)
6. [Built-in Slash Commands](#6-built-in-slash-commands)
7. [Hooks](#7-hooks)
8. [MCP Servers](#8-mcp-servers)
9. [Sub-agents](#9-sub-agents)
10. [Settings & Permissions](#10-settings--permissions)
11. [Keyboard Shortcuts](#11-keyboard-shortcuts)
12. [Token & Cost Management](#12-token--cost-management)
13. [IDE Integration](#13-ide-integration)
14. [Effective Prompting](#14-effective-prompting)

---

## 1. CLI Usage

### What & Why
The `claude` CLI invokes Claude Code outside an IDE. Supports interactive sessions, one-shot print mode, session continuity, and scripting.

### Invocation Modes

| Mode | Command | Use Case |
|---|---|---|
| Interactive | `claude` | Normal development work |
| With prompt | `claude "fix the auction controller"` | Start with a task |
| Print (non-interactive) | `claude -p "summarize changes"` | CI scripts, piping output |
| Continue last | `claude -c` | Resume where you left off |
| Resume specific | `claude -r "session-name"` | Go back to named session |

**Print mode in CI:**
```bash
git diff HEAD~1 | claude -p "review this diff for security issues" --output-format json
```

**Continue after closing terminal:**
```bash
claude -c   # Picks up the last conversation
```

### Key Flags

```bash
--model sonnet              # Use a specific model (cheaper)
--model opus                # Use most capable model
--add-dir ../Modules/AI     # Include another directory in context
--permission-mode acceptEdits       # Auto-accept file edits
--permission-mode plan              # Read-only planning mode
--permission-mode bypassPermissions # Skip all prompts (use carefully)
--max-turns 10              # Limit how many steps Claude can take
--max-budget-usd 0.50       # Stop before spending $0.50
--verbose                   # Show tool calls and reasoning
--system-prompt "..."       # Override system prompt
```

**Automated task with budget cap:**
```bash
claude -p "refactor the AuctionController to use the repository pattern" \
  --max-budget-usd 0.30 \
  --permission-mode acceptEdits
```

**Add module context to a session:**
```bash
claude --add-dir Modules/Auction "why is the bid placement failing?"
```

### Worktrees (Parallel Work)
```bash
claude -w feature-payment    # Start in isolated git worktree
claude -w bugfix-auction     # Parallel session on same repo
```
Each worktree is independent — no conflicts between parallel Claude sessions.

---

## 2. CLAUDE.md

### What & Why
CLAUDE.md is the instruction file Claude reads at the start of every session. Defines project context, coding standards, commands, and rules. Replaces writing context in every prompt.

### Loading Hierarchy

```
/Library/Application Support/ClaudeCode/CLAUDE.md   → Managed (org-wide)
~/.claude/CLAUDE.md                                  → User (all your projects)
./CLAUDE.md  or  ./.claude/CLAUDE.md                → Project (committed, shared)
./CLAUDE.local.md                                    → Local (gitignored, personal)
Modules/Auction/CLAUDE.md                            → Module-level (loads when working there)
```

**Loading rule:** More specific = higher priority. Nested module CLAUDE.md only loads when Claude reads files in that directory.

### When to Use Each Level

| Level | Use For |
|---|---|
| Project `CLAUDE.md` | Team-wide conventions, architecture, commands |
| `CLAUDE.local.md` | Personal preferences, local paths, secrets |
| `~/.claude/CLAUDE.md` | Global habits across all your projects |
| `Modules/X/CLAUDE.md` | Module-specific rules (only loads in-module) |

### `@` Directive — Import Files

```markdown
@.claude/rules/translate.md       # Import a rule file
@.claude/rules/response-style.md  # Import response style
@README.md                        # Import README into context
```

**Why:** Keeps CLAUDE.md short while pulling focused rule files into context.

### Size Guideline
Keep under 200 lines. Beyond that, adherence drops and token cost rises.

**Bad:**
```markdown
## Architecture
[500 lines of architecture docs...]
```

**Good:**
```markdown
## Architecture
Flow: Controller → Service → Repository → Model
Full docs: AI_MODEL_PROJECT_CONTEXT.md (read on demand)
```

### Exclude Specific CLAUDE.md Files
`.claude/settings.local.json`:
```json
{
  "claudeMdExcludes": ["**/SomeModule/CLAUDE.md"]
}
```

---

## 3. Rules System

### What Rules Are For

Rules are **passive, always-on knowledge** — they shape how Claude thinks about your project without you having to say it every time. Rules are *declarative*: "this is how things are."

**Use rules for:**
- Coding standards ("always use the repository pattern")
- Architecture constraints ("never query AuctionProduct directly")
- API response formats ("all responses must include `auction_status`")
- Naming conventions ("services go in `app/Services/`, not controllers")

**Do NOT use rules for:** Multi-step procedures or workflows — those are [skills](#5-skills--custom-slash-commands).

### Structure
```
.claude/rules/
├── translate.md          # Blade translation (always loaded via @)
├── response-style.md     # Output format rules
├── api-conventions.md    # REST API patterns
└── frontend/
    └── vue-patterns.md   # Vue 2 component rules (path-scoped)
```

### Scoped Rules (Only Load for Matching Files)

```markdown
---
paths:
  - "resources/views/**/*.blade.php"
  - "resources/js/**/*.vue"
---

# Frontend-only rules
These rules only apply when editing Blade or Vue files.
```

**Why path-scoping matters:** A Blade translation rule doesn't need to load when you're editing a migration. Path-scoped rules save tokens and reduce irrelevant context.

### Always-loaded vs. On-demand

| Type | How to Configure | When It Loads |
|---|---|---|
| `@.claude/rules/foo.md` in CLAUDE.md | Direct import | Every request |
| Path-scoped `---paths:---` | Automatic | When Claude reads matching files |
| No paths + no import | Orphaned | Never — must be imported |

### Example — Module-specific rule

`.claude/rules/auction-api.md`:
```markdown
---
paths:
  - "Modules/Auction/**/*"
---

# Auction API Conventions
- All auction API responses MUST include `auction_status` field
- Use AuctionRepository — never query AuctionProduct directly
- Bid validation lives in AuctionBidService, not controllers
```

This only loads when editing files inside `Modules/Auction/`.

### Writing Strong Rules

Rules must be **imperative, specific, and structured as conditions** — not prose Claude has to interpret.

| Weak (suggestive) | Strong (imperative) |
|---|---|
| "You should probably use..." | "Use X" |
| "Try to avoid..." | "Never do X" |
| "Be careful about payment flows..." | "When editing payment flows: ALWAYS wrap in DB transaction, NEVER deduct before verifying auction is open" |

**Three-layer structure for every rule file:**
```
1. WHAT  — one-line statement of the rule
2. HOW   — specific, actionable steps or patterns
3. EXCEPTIONS — what NOT to do, edge cases
```

### Practical Rule Examples

#### Example 1 — Always-on Rule: Repository Pattern
**Problem:** Developers (and Claude) query Eloquent models directly in controllers, bypassing the repository layer and making the codebase inconsistent.
**Solution:** A project-wide rule imported via `@` in CLAUDE.md — loads on every request.

`.claude/rules/repository-pattern.md`:
```markdown
# Repository Pattern

## WHAT
All database access MUST go through repository classes.
Never query Eloquent models directly in controllers or services.

## HOW
- Controllers → call Service methods only
- Services    → call Repository methods only
- Repositories → use Eloquent models internally

✅ Correct (in AuctionBidService.php):
  $bid = $this->auctionBidRepository->findByIdWithUser($bidId);

❌ Wrong — never do this (in any controller or service):
  $bid = AuctionBid::with('user')->find($bidId);

## EXCEPTIONS
- Seeders and migrations may query models directly
- Console commands for local dev/debug only may skip the repo layer
- Never skip in production-facing code under any circumstance
```

`CLAUDE.md` import:
```markdown
@.claude/rules/repository-pattern.md
```

---

#### Example 2 — Path-Scoped Rule: Payment Flow Safety
**Problem:** Payment and wallet logic is high-risk — a missing DB transaction or a balance deduction before verifying auction status causes real money loss.
**Solution:** A path-scoped rule that only loads when Claude is editing Auction or Payment files.

`.claude/rules/payment-flow-safety.md`:
```markdown
---
paths:
  - "Modules/Auction/**/*.php"
  - "Modules/Payment/**/*.php"
  - "app/Services/Wallet*.php"
---

# Payment & Wallet Flow Safety

## WHAT
All wallet operations MUST be atomic and audited. Any mistake causes real money loss.

## HOW
ALWAYS wrap in a DB transaction:
  DB::transaction(function () use ($bid, $amount) {
      $this->walletRepository->deduct($bid->user_id, $amount);
      $this->auctionBidRepository->markAsPaid($bid->id);
      AuditLogger::log('bid_payment', $bid->id, $amount);
  });

ALWAYS verify auction is OPEN before deducting balance.
On ANY failure: refund via WalletService::refund() and log with AuditLogger.

## EXCEPTIONS
- NEVER deduct balance before verifying auction status
- NEVER update wallet balance outside a DB transaction
- NEVER swallow exceptions — always re-throw after refund
- NEVER skip AuditLogger on payment operations, even in tests
```

This file is **not** imported in CLAUDE.md. It loads **automatically** only when Claude touches a matching file path.

---

#### Example 3 — Path-Scoped Rule: API Response Format
**Problem:** Different developers return API responses in different shapes — some use `AuctionResource`, others build arrays manually, causing inconsistent frontend integration.
**Solution:** A path-scoped rule for all Auction API controllers and routes.

`.claude/rules/auction-api-response.md`:
```markdown
---
paths:
  - "Modules/Auction/Http/**/*.php"
  - "routes/rest_api/v1/auction*.php"
---

# Auction API Response Conventions

## WHAT
All Auction API responses MUST use AuctionResource. Never build response arrays manually.

## HOW
✅ Always:
  return response()->json(['status' => true, 'data' => new AuctionResource($auction)]);

❌ Never:
  return response()->json(['id' => $auction->id, 'title' => $auction->title]);

Required fields in every auction response (enforced by AuctionResource):
- auction_status: "open" | "closed" | "cancelled"
- bid_count: int
- current_price: float (null if no bids)
- ends_at: ISO 8601 datetime

## EXCEPTIONS
- Internal admin-only endpoints may use simplified responses — document with // admin-only
- Webhook payloads use WebhookAuctionPayload, not AuctionResource
```

---

## 4. Memory System

### What & Why
Auto memory is Claude's persistent notepad across sessions. Stores things Claude learns about you and your project so you don't repeat yourself.

### File Structure
```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md             # Index — first 200 lines load at startup
├── feedback_no_coauthor.md
└── user_preferences.md
```

### Memory Types

| Type | Purpose | Example |
|---|---|---|
| `user` | Who you are, skill level | "Senior Laravel developer, new to Vue" |
| `feedback` | How Claude should behave | "Never add Co-Authored-By in commits" |
| `project` | Ongoing decisions & deadlines | "Merge freeze starts 2026-05-10" |
| `reference` | Where to find things | "Bug tracker is Linear project AUCTION" |

### Memory File Format
```markdown
---
name: No Co-Authored-By in commits
type: feedback
---

Never include "Co-Authored-By" when committing via /push.

**Why:** Automated tags clutter commit history.
**How:** Commit message = one short subject line. No body, no trailers.
```

### Rules vs. Memory — When to Use Which

| | Rules (`.claude/rules/`) | Memory (`~/.claude/.../memory/`) |
|---|---|---|
| Visible in IDE | Yes | No |
| Version controlled | Yes | No |
| Shared with team | Yes | No |
| Use for | Project standards | Personal behavioral feedback |

**Guideline:** Team standard → `.claude/rules/`. Personal correction ("Claude keeps doing X wrong for me") → memory.

### `/memory` Command
Opens memory editor. Toggle auto memory on/off, view/edit all topic files.

---

## 5. Skills — Custom Slash Commands

### What Skills Are For

Skills are **multi-step workflows packaged into a single `/command`**. They are *procedural*: "do these steps in order."

**Use skills for:**
- Repeatable checklists you run before deployment
- Multi-step audits or reviews
- Tasks that require specific tool permissions pre-approved
- Team workflows you want enforced consistently

**Do NOT use skills for:** Always-on standards or knowledge — those are [rules](#3-rules-system).

**Do NOT use skills for:** Things that should auto-trigger — those are [hooks](#7-hooks).

### Key Distinction: Skills vs. Rules

```
Rule  = "Always use the repository pattern"     → passive, always applied
Skill = "/audit-module Auction"                 → active, invoked on demand
```

### File Locations

| Scope | Path | Shared With Team |
|---|---|---|
| Project | `.claude/commands/<name>.md` | Yes (git) |
| Project (new format) | `.claude/skills/<name>/SKILL.md` | Yes (git) |
| User (all projects) | `~/.claude/skills/<name>/SKILL.md` | No |

### Example — `/deploy-check` Skill

`.claude/commands/deploy-check.md`:
```markdown
Run the following pre-deployment checklist:

1. Run `php artisan config:cache` — verify no errors
2. Run `php artisan route:cache` — verify no errors
3. Check `git status` — warn if uncommitted changes exist
4. Run `php artisan migrate --pretend` — list pending migrations
5. Check `.env` for `APP_DEBUG=false` and `APP_ENV=production`

Report: pass/fail for each item, list any warnings.
```

Usage: `/deploy-check` — runs the full checklist instantly.

### Skill with Arguments

`.claude/commands/seed-module.md`:
```markdown
Seed the $ARGUMENTS module with test data:
1. Run `php artisan db:seed --class=$ARGUMENTS`Seeder
2. Verify 5+ records exist in the corresponding table
3. Report row counts
```

Usage: `/seed-module Auction` → seeds the Auction module.

### Skill Frontmatter Options

```yaml
---
description: When to suggest this skill automatically
argument-hint: [branch] [environment]   # Autocomplete hint
allowed-tools: Bash(php artisan *) Read  # Pre-approve tools
model: sonnet                           # Use cheaper model for this skill
---
```

### When to Create a Skill

Create a skill when:
- You type the same multi-step instruction more than 3× a week
- The workflow is hard to remember or must be done in a specific order
- The task requires specific tool permissions pre-approved
- You want team members to follow the same process

### Practical Skill Examples

#### Example 1 — `/deploy-check` (Safety Checklist)
**Problem:** Before every deployment you need to run config cache, check .env, and verify pending migrations — but you sometimes forget steps under pressure.
**Solution:** A skill that runs the full checklist in one command.

`.claude/commands/deploy-check.md`:
```markdown
---
description: Run before every production deployment to verify safe-to-release
allowed-tools: Bash(php artisan *) Bash(git *) Read
---

Run ALL steps below in order. Stop and report if any step fails.

Step 1 — Config & Route Cache
  php artisan config:cache && php artisan route:cache && php artisan view:cache
  Expected: no errors. If errors appear, stop and report the exact message.

Step 2 — Uncommitted Changes
  git status
  If uncommitted files exist, list them and warn before continuing.

Step 3 — Pending Migrations
  php artisan migrate --pretend
  List all migrations that would run. If none: "No pending migrations."

Step 4 — Environment Check
  Read .env and verify: APP_DEBUG=false, APP_ENV=production,
  QUEUE_CONNECTION is not "sync".

Step 5 — Final Report
  Print a ✅/❌/⚠️ summary table for all checks.
```

**Usage:** `/deploy-check`

---

#### Example 2 — `/audit-module [ModuleName]` (Pre-Review Audit)
**Problem:** Before merging a feature branch you want to catch architectural violations (direct model queries, missing auth middleware, unindexed foreign keys) without reading every file manually.
**Solution:** A skill that searches the module and produces a compact violation report.

`.claude/commands/audit-module.md`:
```markdown
---
description: Audit a module for architectural violations before code review or merge
argument-hint: [ModuleName]
allowed-tools: Bash(grep *) Bash(find *) Read
model: sonnet
---

Audit Modules/$ARGUMENTS/ for violations. Report every match as "file:line — issue".
Do NOT make any edits — report only.

Check 1 — Direct Model Queries (bypasses repository):
  grep -rn "::where\|::find\|::all\|::first" Modules/$ARGUMENTS/ --include="*.php" \
    | grep -v "Repository\|Test\|Seeder"

Check 2 — Missing Auth Middleware:
  Read all route files in Modules/$ARGUMENTS/Routes/.
  Flag any route missing "auth:api" middleware.

Check 3 — Missing DB Indexes on Foreign Keys:
  Read all migration files in Modules/$ARGUMENTS/Database/Migrations/.
  Flag any foreign() column without a corresponding ->index() call.

Output as:
  ## Audit Report — Modules/$ARGUMENTS
  ### ❌ Direct Model Queries (N found)
  ### ❌ Missing Auth Middleware (N found)
  ### ⚠️  Missing Indexes (N found)
  Total violations: N — recommend fixing before merge.
```

**Usage:** `/audit-module Auction` — audits `Modules/Auction/`. Takes an argument.

---

#### Example 3 — `/make-feature [Module] [Name]` (Feature Scaffolding)
**Problem:** Creating a new feature requires 6+ files in the right locations with consistent boilerplate. Doing it manually takes 20 minutes and often misses a file.
**Solution:** A skill that scaffolds everything in one command.

`.claude/commands/make-feature.md`:
```markdown
---
description: Scaffold a complete feature (Controller, Service, Repository, Resource, Routes)
argument-hint: [ModuleName] [FeatureName]
allowed-tools: Bash(php artisan *) Write Edit
---

Parse $ARGUMENTS as: ModuleName FeatureName (e.g., "Auction BidRefund").
Create the following files following existing conventions in the project:

1. Controller  → Modules/{Module}/Http/Controllers/V1/Customer/{Feature}Controller.php
   Inject {Feature}Service. Stub: index, show, store, update, destroy.
   Each method calls the service and returns a Resource response.

2. Service     → Modules/{Module}/Services/{Feature}Service.php
   Inject {Feature}Repository. Stub methods matching the controller.

3. Repository  → Modules/{Module}/Repositories/{Feature}Repository.php
   Extend BaseRepository, implement {Feature}RepositoryInterface.

4. Interface   → Modules/{Module}/Contracts/{Feature}RepositoryInterface.php
   Declare all repository method signatures.

5. Resource    → Modules/{Module}/Http/Resources/{Feature}Resource.php
   toArray() returning empty array with // TODO: define fields comment.

6. Route       → Append to Modules/{Module}/Routes/api.php:
   Route::apiResource('{feature-kebab}', {Feature}Controller::class);

After creating all files: list every path created and print the service
provider binding snippet the user must add manually.
```

**Usage:** `/make-feature Auction BidRefund` — creates all 6 files for BidRefund inside the Auction module.

---

## 6. Built-in Slash Commands

### What Built-in Commands Are For

Built-in commands control **Claude Code's own session state** — not your project. They're Claude Code's UI controls.

**Key distinction:** Built-in commands control the *tool*. Skills are workflows you define for your *project*.

### Most Useful Built-ins

| Command | When to Use It |
|---|---|
| `/clear` | Switching to a completely different task |
| `/compact [instructions]` | Context filling up mid-session |
| `/cost` or `/usage` | Checking spend on a long session |
| `/model sonnet` | Switch to cheaper model mid-session |
| `/model opus` | Switch to most capable model |
| `/memory` | View/edit personal memory files |
| `/init` | Auto-generate CLAUDE.md from codebase scan |
| `/doctor` | Diagnose configuration issues |
| `/status` | Show active settings, model, permission mode |
| `/thinking` | Toggle extended thinking (deeper reasoning) |
| `/mcp` | Manage MCP servers + OAuth |
| `/todo` | Show/manage task list |
| `/rewind` | Undo Claude's last action |
| `/vim` | Toggle vim keybindings |

### Practical Scenario Examples

**`/compact` — Context filling up mid-task**

Scenario: You've been debugging the auction payment flow for 45 minutes. Claude's responses are getting slower — the context is full of old messages and intermediate file reads.
```
/compact Focus only on the AuctionBidPaymentService refactor — ignore earlier debugging
```
Claude summarizes everything into a compact note, re-injects CLAUDE.md, and continues with fresh context on the thing that matters. You don't lose the thread — you drop the noise.

---

**`/clear` — Switching to a completely different task**

Scenario: You just finished the bid placement bug. Now you're moving to the seller dashboard — totally unrelated. You don't want auction payment context bleeding in.
```
/clear
```
Then start fresh. **Difference from `/compact`:** `/compact` keeps a focused summary. `/clear` wipes everything. Use `/clear` when topics are unrelated.

---

**`/cost` — Before starting a heavy task**

Scenario: You've been in a session for 2 hours on a large refactor. Before asking Claude to scan 200+ files, you want to know how much you've already spent this session.
```
/cost
```
Shows current session token count and dollar estimate. If you're at $0.40 with a $0.50 budget, use `/compact` before the next large task.

---

**`/model sonnet` — Switch models mid-session**

Scenario: You used Opus for a complex architecture decision. Now you just need Claude to generate 10 CRUD stubs — no deep reasoning needed.
```
/model sonnet
```
Sonnet handles repetitive code generation at a fraction of the cost. Switch back with `/model opus` when you return to complex design work.

---

**`/rewind` — Claude made a wrong edit**

Scenario: You asked Claude to "clean up" AuctionBidService and it deleted a method you still needed.
```
/rewind
```
Or press `Esc Esc`. Restores all files Claude touched in its last response. **Note:** Only undoes the last Claude action. For older edits, use `git restore` instead.

---

**`/thinking` — Complex design problem**

Scenario: You need Claude to design the refund flow for cancelled auctions — it involves wallets, bid states, notification queues, and multiple edge cases. You want Claude to reason before proposing.
```
/thinking
```
Then ask your question. Extended thinking makes Claude work through the problem step-by-step before answering. Slower and more tokens, but significantly better on complex design questions.

---

**`/memory` — Saving a persistent preference**

Scenario: Claude keeps using `response()->json()` directly in controllers instead of your `ApiResponse::success()` helper. You've corrected it twice and don't want to do it a third time.
```
/memory
```
Add a new memory entry:
```markdown
---
name: Use ApiResponse helper in controllers
type: feedback
---
Always use ApiResponse::success($data) and ApiResponse::error($message, $code).
Never use response()->json() directly in controllers.
```
This persists across all future sessions automatically.

---

## 7. Hooks

### What Hooks Are For

Hooks run shell commands **automatically at specific lifecycle events** — without you invoking them. They enforce rules, automate reactions, and block dangerous actions.

**Use hooks for:**
- Blocking dangerous commands *every time* Claude tries them (not just sometimes)
- Auto-running lint/tests after file edits
- Logging every Bash command Claude executes
- Notifying you when Claude finishes a long task

### Key Distinction: Hooks vs. Skills

```
Skill = You invoke it on demand           → "/deploy-check"
Hook  = Fires automatically every time   → blocks rm -rf every single time it's attempted
```

If you'd forget to run a check manually → use a hook. If you invoke it deliberately → use a skill.

### Lifecycle Events

| Event | When It Fires | Use It For |
|---|---|---|
| `PreToolUse` | Before Claude calls any tool | Blocking dangerous actions, logging |
| `PostToolUse` | After tool completes | Auto-lint, auto-test, auto-format |
| `UserPromptSubmit` | When you send a message | Input validation |
| `Stop` | When Claude finishes responding | Notifications |
| `SessionStart` | When session begins | Health checks, context loading |
| `SessionEnd` | When session ends | Cleanup, summary logging |
| `FileChanged` | When a file is changed | Triggering file watchers |

### Hook Exit Codes

| Exit Code | Behavior |
|---|---|
| `0` | Success — continue |
| `2` | **Block the action** — stderr sent to Claude as explanation |
| Other | Continue — stderr shown to user |

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
            "command": ".claude/hooks/validate.sh"
          }
        ]
      }
    ]
  }
}
```

### Practical Hook Examples

#### Example 1 — Block Dangerous Artisan Commands (PreToolUse)
**Problem:** Claude sometimes suggests `migrate:fresh` or `db:wipe` during debugging sessions. These destroy ALL database data. You need this blocked *every single time* — not just when you remember to ask.
**Why a hook and not a rule?** A rule might be ignored. Exit code 2 *physically prevents* Claude from running the command.

`.claude/hooks/block-dangerous-artisan.sh`:
```bash
#!/bin/bash
# $CLAUDE_TOOL_INPUT = the full bash command Claude is about to run

BLOCKED="migrate:fresh|migrate:reset|db:wipe"

if echo "$CLAUDE_TOOL_INPUT" | grep -qE "$BLOCKED"; then
    echo "🚫 BLOCKED: This command destroys all database data." >&2
    echo "Blocked command: $CLAUDE_TOOL_INPUT" >&2
    echo "" >&2
    echo "Safe alternatives:" >&2
    echo "  php artisan migrate         (run pending migrations only)" >&2
    echo "  php artisan migrate:status  (check migration state)" >&2
    echo "If you genuinely need a reset, ask the user for explicit confirmation." >&2
    exit 2   # Exit code 2 = block + send stderr to Claude as explanation
fi

exit 0  # Allow everything else
```

`.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/hooks/block-dangerous-artisan.sh" }]
      }
    ]
  }
}
```

---

#### Example 2 — Auto-run Linter After File Edits (PostToolUse)
**Problem:** Claude edits PHP files but doesn't always run Laravel Pint afterward. You catch style violations manually during review — wasted time.
**Why a hook?** If you rely on remembering to ask Claude to lint after every edit, you'll forget. The hook makes it automatic and invisible.

`.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'if echo \"$CLAUDE_TOOL_INPUT\" | grep -q \"\\.php\"; then ./vendor/bin/pint --test 2>&1 || true; fi'"
          }
        ]
      }
    ]
  }
}
```
Pint runs silently after every PHP file edit. Lint errors appear in Claude's context automatically.

---

#### Example 3 — Desktop Notification When Claude Finishes (Stop)
**Problem:** Long Claude tasks (refactors, audits) take 2–5 minutes. You switch to another window and miss when it's done — then lose 10 minutes before you come back.
**Why a hook?** You can't manually trigger a notification at the end — you don't know when it'll end. Only a `Stop` hook can do this automatically.

`.claude/settings.json`:
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude finished a task\" with title \"Claude Code\" sound name \"Glass\"' 2>/dev/null || notify-send 'Claude Code' 'Task finished' 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```
Works on macOS (osascript) and Linux (notify-send). The `|| true` prevents the hook from failing on the other OS.

---

#### Example 4 — Audit Log of All Bash Commands (PreToolUse)
**Problem:** You need a complete record of every command Claude ran during a session for a security review. You can't watch every command manually during a long refactor.

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
            "command": "echo \"$(date '+%Y-%m-%d %H:%M:%S') | $CLAUDE_TOOL_INPUT\" >> ~/.claude/logs/bash-audit.log"
          }
        ]
      }
    ]
  }
}
```
Every Bash command Claude runs — with timestamp — is written to `~/.claude/logs/bash-audit.log`. Exit code 0 (no blocking).

---

#### Example 5 — Dev Server Health Check at Session Start (SessionStart)
**Problem:** You start a Claude session and ask it to test an API endpoint, but the dev server isn't running. Claude wastes 3 tool calls before figuring this out.

`.claude/settings.json`:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'curl -sf http://localhost:8000/api/health > /dev/null 2>&1 || echo \"⚠️ WARNING: Dev server not responding at localhost:8000. Run: php artisan serve\" >&2'"
          }
        ]
      }
    ]
  }
}
```
If the server isn't running, Claude sees the warning immediately at session start and knows to mention it before attempting API calls.

---

```
"Bash"              # Exact match
"Edit|Write"        # Multiple tools (regex OR)
"^mcp__"            # Any MCP tool
"Bash(git push *)"  # Specific bash command pattern
""  or omit         # Match everything
```

---

## 8. MCP Servers

### What MCP Is For

MCP (Model Context Protocol) gives Claude **live access to external systems** — databases, APIs, file systems, GitHub, Slack. Without MCP, Claude can only read/edit local files and run bash commands. With MCP, Claude interacts directly with external services.

**The core problem MCP solves:** Without it, you copy-paste data between Claude and external tools. With it, Claude gets direct access.

### When to Use MCP

Use MCP when:
- Claude needs to **read live data** (DB records, GitHub issues, API responses)
- Claude needs to **act on external systems** (create PRs, update tickets, send messages)
- You're tired of copy-pasting results into Claude's context

Use Bash instead when: The external access is a one-off command and the structured input/output of MCP isn't worth the setup.

### MCP vs. Bash

```
Bash: Claude runs `mysql -e "SELECT..."` — output is raw text, Claude parses it
MCP:  Claude calls a typed tool with structured input/output — cleaner, safer, repeatable
```

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
claude mcp add --scope project my-server ...  # .mcp.json (committed to git)
claude mcp add --scope user my-server ...     # ~/.claude.json (personal)
claude mcp add --scope local my-server ...    # Not shared (default)
```

### Useful MCP Servers for Laravel Projects

| Server | What It Gives Claude | Install |
|---|---|---|
| `github` | Create PRs, review issues, manage releases | `claude mcp add github --transport http https://mcp.github.com` |
| `filesystem` | Expanded file operations | `claude mcp add filesystem -- npx @modelcontextprotocol/server-filesystem .` |
| `mysql` | Query the database directly | Custom server |

### Practical MCP Examples

#### Example 1 — MySQL: Direct Database Investigation
**Problem:** A bug report says "some auction bids are stuck in pending payment." Without MCP you have to: write the query → run it in TablePlus → copy the results → paste into Claude → repeat for every follow-up query. With MCP, Claude investigates directly.

**Setup (one-time):**
```bash
npm install -g mysql-mcp-server

claude mcp add mysql --scope project \
  --env DB_HOST=127.0.0.1 \
  --env DB_PORT=3306 \
  --env DB_NAME=6valley \
  --env DB_USER=root \
  --env DB_PASS=secret \
  -- npx mysql-mcp-server
```

**What you can now say to Claude:**
```
Check the auction_bids table for any bids placed in the last 48 hours
where payment_status is NULL. Show bid_id, user_id, amount, and created_at.
```
Claude runs the query, reads the result, and immediately follows up — no copy-pasting.

```
Find all auctions where status = 'closed' but there are still bids
with payment_status = 'pending'. These are stuck payments needing refunds.
```

```
After running the migration, confirm that the minimum_bid column exists
on auction_products and show 3 sample rows to verify the data migrated correctly.
```

**Safety:** Use read-only DB credentials for investigation. Only grant write access in dev/staging when Claude needs to fix data directly.

---

#### Example 2 — GitHub: PR and Issue Management
**Problem:** During development you constantly switch between terminal and GitHub browser to check issues, create PRs, and add comments. Each context switch costs 5–10 minutes.

**Setup (one-time):**
```bash
claude mcp add github --transport http https://mcp.github.com --scope user
# Then: /mcp → GitHub → Connect (OAuth — no token needed)
```

**What you can now say to Claude:**
```
List all open issues assigned to me labeled 'auction-module', sorted by updated date.
```

```
Create a PR for the current branch targeting main.
Title: "Add rate limiting to auction bid placement API"
Description: summarize the changes from the last 5 commits.
Labels: auction-module, backend
Request review from: @team-backend
```

```
I just fixed issue #247 (bid placement 500 error).
Add a comment on that issue explaining the fix was merged in this PR.
```

**Reference GitHub resources directly in your prompts:**
```
@github:issue://247    # Attach issue context to your message
@github:pr://88        # Attach a PR diff to your message
```
Type `@` in Claude Code to see all available GitHub resources.

---

```bash
claude mcp list         # List all configured servers
claude mcp get github   # Show server details
claude mcp remove name  # Remove a server
```

Or use `/mcp` inside Claude Code for GUI + OAuth management.

### MCP Resources (`@` mentions)

```
@github:issue://123      # Reference a GitHub issue
@filesystem:./README.md  # Reference a file via MCP
```

Type `@` in the prompt to see all available resources.

---

## 9. Sub-agents

### What Sub-agents Are For

Sub-agents are **separate Claude instances** spawned from your main session to handle isolated, large, or parallel tasks. They protect your main context window from verbose output.

### The Core Problem Sub-agents Solve

**Without sub-agent:**
```
You: "Find all direct DB::table() calls in Modules/Auction/"
Claude: [reads 244 files, fills your context with raw file contents]
→ Context is clogged. Subsequent prompts are slower and more expensive.
```

**With sub-agent:**
```
You: "Spawn an Explore agent: find all direct DB::table() calls in Modules/Auction/"
Sub-agent: [reads 244 files in its own isolated context]
Sub-agent: "Found 7 instances: file:line ..."
→ Your context receives only the compact report.
```

### When to Use Sub-agents

Use a sub-agent when:
- The task produces **massive output** (searching hundreds of files)
- The task is **independent** of your current work (can run in parallel)
- You want to **protect your main context** from exploratory, verbose work
- You need **parallel execution** of two unrelated tasks

### Built-in Agent Types

| Type | Specialization | Use When |
|---|---|---|
| `Explore` | Read-only search | "Find all places X is called" |
| `Plan` | Architecture design | "Design the bid refund flow" |
| `general-purpose` | All tools | Complex multi-step tasks |
| `claude-code-guide` | Claude Code questions | "How does MCP work?" |

### Foreground vs. Background

| | Foreground | Background |
|---|---|---|
| Blocks your session | Yes | No |
| Result shown | Full response | Summary only |
| Use when | You need results before continuing | Independent parallel task |

### Practical Sub-agent Examples

#### Example 1 — Explore Agent: Large Codebase Search
**Problem:** You need to find every place `AuctionProduct` is queried directly across 244 files. Reading all files in your main session would fill the context window and slow every subsequent response for the rest of the session.

**What to type:**
```
Spawn an Explore agent with this task:

Search all files in Modules/Auction/ for direct Eloquent model queries
that bypass the repository pattern. Find any ::where(, ::find(, ::first(, ::all(
called on model classes — not inside files ending in Repository.php or Test.php.

For each match: report file path, line number, and the exact line.
Return as: "file:line — code snippet"
Do not read full file contents, only identify matching lines.
```

**What you get back:** A 10-line compact report. Your context window receives the summary only — not 244 file reads.

---

#### Example 2 — Parallel Agents: Two Independent Audits
**Problem:** Before a code review you want a route auth audit AND a migration index audit. Running them sequentially wastes time; running them in your main session pollutes context with both outputs.

**What to type:**
```
Run these two tasks in parallel using separate agents.
Report both results before making any changes.

Agent 1 — Route Auth Audit:
Read all route files in Modules/Auction/Routes/ and list every route
missing auth:api middleware. Format: "METHOD /path — file:line"

Agent 2 — Migration Index Audit:
Read all files in Modules/Auction/Database/Migrations/ and list every
foreign key column with no corresponding ->index() call.
Format: "column_name — migration_file:line"
```

Both agents run simultaneously. Your context gets two compact reports — no file dumps.

---

#### Example 3 — Worktree Agent: Safe Experiment
**Problem:** You want to try a major refactor (splitting `AuctionBidService` into 3 smaller services) but aren't sure if it'll work, and you don't want to mess up your current branch if it fails.

**What to type:**
```
Spawn a sub-agent in a new worktree (isolation: worktree) with this task:

Refactor AuctionBidService.php by extracting its three responsibilities
into separate services:
  - AuctionBidValidationService  (all validation methods)
  - AuctionBidPaymentService     (all payment/wallet methods)
  - AuctionBidNotificationService (all notification methods)

Update all callers to inject the correct new service.
Run: php artisan test --filter=AuctionBid

If tests pass: "Refactor successful — ready to review."
If tests fail: list which tests failed and why.
```

If the experiment produces no good result, the worktree auto-cleans. Your working branch is completely untouched.

---

#### Example 4 — Background Agent: Generate Docs Without Blocking
**Problem:** You need API documentation for every endpoint in the Auction module. This would produce 500+ lines of output — too much to have in your main session's context when you're mid-task.

**What to type:**
```
Spawn a background general-purpose agent with this task:

Read all controller files in Modules/Auction/Http/Controllers/V1/.
For each public method mapped to a route, generate a markdown doc block:
  - Method + URL (infer from route names)
  - Parameters (from Request classes)
  - Response format (from Resource classes)
  - Auth required: yes/no

Write the output to docs/api/auction-module.md.
When done, report: "Done — X endpoints documented in docs/api/auction-module.md"
```

You continue working. When the agent finishes, you get a one-line confirmation — and the full output went straight to the file, not your context.

---

#### When NOT to Use a Sub-agent
- Simple, focused tasks (1–3 files) — just do it in your main session
- Tasks where you need to interact mid-way — you can't redirect a running agent
- When the task depends on context from your current conversation — agents don't share your session history

---

## 10. Settings & Permissions

### What & Why
Settings control model, permissions, hooks, env vars, and behavior. Permissions control which tools Claude can use without prompting you.

### Settings File Hierarchy

```
Managed (IT)                        → always wins, can't override
  ↓
Command-line flags
  ↓
.claude/settings.local.json         (local, not committed)
  ↓
.claude/settings.json               (project, committed)
  ↓
~/.claude/settings.json             (user, all projects) ← lowest priority
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

| Mode | Auto-approved | Still Prompts For |
|---|---|---|
| `default` | Nothing | Everything |
| `acceptEdits` | All file edits | Bash, web |
| `plan` | Nothing (read-only) | Everything (view only) |
| `bypassPermissions` | Everything | Nothing |

**Switch during session:** `Shift+Tab` cycles through modes.

### Permission Rule Syntax

```json
"allow": [
  "Bash(npm *)",                    // All npm commands
  "Bash(git log *)",                // git log only
  "Read(./src/**)",                 // Read anything in src/
  "Edit(./Modules/**)",             // Edit anything in Modules/
  "WebFetch(domain:packagist.org)"  // Specific domain only
]
```

**Evaluation order:** Deny → Ask → Allow (first match wins).

### What Goes Where

| Setting | `settings.json` (commit) | `settings.local.json` (local only) |
|---|---|---|
| Allowed artisan commands | Yes | |
| Denied dangerous commands | Yes | |
| Personal API keys | | Yes |
| Preferred model | | Yes |
| Team-wide hooks | Yes | |

### `/status` — Check Active Config

```
/status
```

Shows which settings files are active, current model, permission mode, and the source of each setting.

---

## 11. Keyboard Shortcuts

### Core Shortcuts

| Shortcut | Action |
|---|---|
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
|---|---|
| `Ctrl+A` | Jump to start of line |
| `Ctrl+E` | Jump to end of line |
| `Ctrl+K` | Delete to end of line |
| `Ctrl+W` | Delete previous word |
| `Ctrl+U` | Delete to start of line |
| `Alt+B` | Move back one word |
| `Alt+F` | Move forward one word |

### Multiline Input

| Method | How |
|---|---|
| Quick | `\ + Enter` |
| macOS | `Option+Enter` |
| Universal | `Shift+Enter` |

### `@` and `!` Shortcuts

- `@filename` — Attach a specific file to your message (fuzzy search)
- `!command` — Run a shell command directly (e.g., `!git status`)
- `/command` — Run a built-in command or skill

---

## 12. Token & Cost Management

### What Consumes Tokens Per Request

| Source | Est. Tokens | Controllable? |
|---|---|---|
| System prompt | ~3,000–4,500 | No |
| CLAUDE.md + rules | ~200–500 | Yes — keep lean |
| Memory (MEMORY.md) | ~200–300 | Yes — keep index short |
| MCP tool schemas | ~800–1,200 | Yes — disable unused servers |
| Conversation history | Grows | Yes — `/compact` or `/clear` |
| `@Directory` directives | ~5,000–30,000 | **Yes — never use `@Directory`** |

### Reduce Per-Request Cost

1. **Never `@Directory` in CLAUDE.md** — reading an entire directory on every request is catastrophic
2. **Use nested CLAUDE.md** — `Modules/Auction/CLAUDE.md` only loads when you're working in that module
3. **Path-scope rules** — `.claude/rules/frontend.md` with `paths: [*.blade.php]` skips on backend tasks
4. **Disable unused MCP servers** — each adds schema tokens
5. **Keep CLAUDE.md under 200 lines** — adherence drops and cost rises beyond that
6. **Use `sonnet` for most tasks** — use `opus` only for complex architecture decisions

### Mid-Session Cost Control

```
/cost          # Check current session cost
/compact       # Summarize + free context (preserves CLAUDE.md)
/clear         # Full reset (start fresh)
/model sonnet  # Switch to cheaper model mid-session
```

### Model Cost Comparison

| Model | Relative Cost | Best For |
|---|---|---|
| Haiku 4.5 | Cheapest | Sub-agents, search, simple tasks |
| Sonnet 4.6 | Mid | Most development tasks |
| Opus | Most expensive | Complex reasoning, architecture |

**Practical pattern:** Sonnet for development, Haiku sub-agents for large searches/audits, Opus only for architecture decisions.

### Budget Cap (Non-interactive)
```bash
claude -p "refactor AuctionBidService" --max-budget-usd 0.25
```

Stops before exceeding budget. Useful for automated scripts.

### Prompt Caching
Claude automatically caches system prompt + CLAUDE.md across sessions. First request costs full price; subsequent requests reuse cache at ~10% of the cost. Cache TTL: 5 minutes in active conversation, up to 1 week for static content.

---

## 13. IDE Integration

### VS Code Extension

**Open Claude Code panel:** Click spark icon in top-right toolbar, or `Cmd+Esc`.

Key features:
- Side-by-side diff review with permission prompt
- `Option+K` — Insert `@file.php#10-25` reference with line range
- Permission mode selector in prompt bar
- Session history (local + remote)
- Multiple conversations in tabs

**VS Code-specific settings:**
```json
{
  "claudeCode.initialPermissionMode": "acceptEdits",
  "claudeCode.useCtrlEnterToSend": true,
  "claudeCode.autosave": true
}
```

### JetBrains (PhpStorm)

**Open:** `Cmd+Esc` from editor or Claude Code button in toolbar.

Key features:
- IDE lint/error diagnostics auto-shared with Claude
- `Cmd+Option+K` — Insert `@file#L1-99` reference
- Diff viewer integrated into IDE

**If ESC doesn't interrupt Claude:**
Settings → Tools → Terminal → uncheck "Move focus to the editor with Escape"

### Connect IDE to External Terminal Session
```
/ide   # In Claude Code CLI → connects to open VS Code/JetBrains
```

---

## 14. Effective Prompting

### The Signal Formula

Every high-quality prompt has three elements:

```
[WHAT] + [CONTEXT] + [CONSTRAINT]
```

| Element | Weak | Strong |
|---|---|---|
| WHAT | "fix the bug" | "fix the null pointer in AuctionBidService@placeBid" |
| CONTEXT | (nothing) | "bid fails when user has no wallet balance record" |
| CONSTRAINT | (nothing) | "edit only the service, not the controller; no new methods" |

**Weak prompt:**
```
Fix the auction bid issue
```

**Strong prompt:**
```
AuctionBidService::placeBid() throws "Undefined index: balance" when 
the user's wallet record doesn't exist. Fix: return a validation error 
instead of crashing. Edit only AuctionBidService — don't touch the controller.
```

### Define Scope Explicitly

```
# Bad — Claude might rewrite the whole file
"Improve the AuctionController"

# Good — Claude changes exactly what you need
"In AuctionController::store(), add validation that auction_end_date 
is in the future. Don't change any other methods."
```

**Scope keywords:**
- `Only edit X` — restricts file scope
- `Don't change Y` — explicit exclusion
- `No new methods/classes` — architecture constraint
- `Minimal change` — prevents over-engineering

### Attach Files, Don't Describe Them

```
# Bad
"Update bid placement to match our API response format"

# Good
@Modules/Auction/Http/Resources/BidResource.php
Update AuctionBidService::placeBid() to return a response 
matching the BidResource structure above.
```

**Rules for `@` attachments:**
- Attach the specific file, not the whole module
- Attach the interface when asking Claude to implement it
- Attach the failing test when asking Claude to fix it

### Give Error Context, Not Just Symptoms

```
# Bad
"The auction bid isn't working"

# Good
"php artisan auction:process-bids throws:
  ErrorException: Undefined property: stdClass::$reserve_price
  in AuctionJobService.php line 87
  
reserve_price was renamed to minimum_bid in migration 2026_04_15.
Fix the reference in AuctionJobService — don't alter the migration."
```

### Ask for a Plan Before Complex Edits

```
Before editing anything: list the files you'll touch and what you'll 
change in each. Wait for my approval before proceeding.
```

Prevents expensive wrong-direction work on complex tasks.

### Intent Declaration Pattern

Start complex sessions with a declaration block:

```
GOAL: Add rate limiting to auction bid placement API
SCOPE: Modules/Auction/Http/Controllers/V1/CustomerAuctionController.php only
APPROACH: Use Laravel's throttle middleware, not custom logic
DONE WHEN: POST /auction/bid returns 429 after 5 requests/minute per user
DO NOT: Change bid logic, touch other controllers, add new middleware classes
```

Takes 10 seconds to write. Eliminates 3–4 correction cycles.

### Token-Efficient Prompt Patterns

**File + Instruction (not description):**
```
@app/Services/AuctionBidService.php
Extract the balance check logic (lines 45–67) into a private method 
`validateBuyerBalance()`. No other changes.
```

**Error + Known Cause + Fix Constraint:**
```
Error: "Call to undefined method AuctionBid::scopeActive()"
Cause: scope was removed in last refactor, use status='active' directly.
Fix: update all ->active() calls in AuctionBidService to ->where('status','active').
Don't add the scope back.
```

**Output-First Specification:**
```
I need a function that:
- Input: auction_id, user_id
- Returns: ['can_bid' => bool, 'reason' => string|null]
- Reads from: AuctionRepository, UserRepository
- No DB calls directly

Write only the function. Add it to AuctionEligibilityService.
```

### Anti-Patterns to Avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| "Fix it" with no context | Claude guesses wrong scope | Add file, line, error message |
| "Improve this code" | Claude rewrites things you didn't ask to touch | State exactly what to improve |
| Repeated corrections | 3× token cost vs. one precise prompt | Write the intent declaration first |
| Pasting 500 lines for context | Fills context unnecessarily | `@file` the specific file instead |
| "Make it better" after Claude goes wrong | Claude doesn't know what "better" means | State the specific failure explicitly |
| Long history on unrelated tasks | Old context bleeds into new work | `/clear` between unrelated tasks |

---

## Quick Reference — "What Do I Use for X?"

| Goal | Use |
|---|---|
| Always-on coding standard | `CLAUDE.md` or `.claude/rules/` with `@` import |
| Standard only for Blade files | `.claude/rules/foo.md` with `paths: [*.blade.php]` |
| Standard only for Auction module | `Modules/Auction/CLAUDE.md` |
| Personal preference (no commit) | `CLAUDE.local.md` or memory |
| Repeatable multi-step workflow | Skill in `.claude/commands/` |
| Block a dangerous command every time | Hook with exit code 2 |
| Auto-run lint after edits | `PostToolUse` hook |
| Search 200+ files without polluting context | Explore sub-agent |
| Query DB without copy-pasting | MCP MySQL server |
| Reduce permission prompts | `permissions.allow` in `.claude/settings.json` |
| Resume yesterday's session | `claude -c` or `-r session-name` |
| Check cost mid-session | `/cost` |
| Free context without losing rules | `/compact` |
| Undo Claude's last file change | `/rewind` or `Esc Esc` |
| Run automated task with budget cap | `claude -p "..." --max-budget-usd 0.50` |
| Run parallel tasks without polluting context | Two parallel sub-agents |
| Notify on session completion | `Stop` hook |
