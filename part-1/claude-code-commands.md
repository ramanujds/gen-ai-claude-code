# Claude Code — Complete Notes & Terminal Commands

## What Is Claude Code?

Claude Code is an agentic coding tool that lives in your terminal, understands your codebase, and helps you code faster by executing routine tasks, explaining complex code, and handling git workflows — all through natural language commands.

It works with all your CLI tools alongside any IDE, and integrates with GitHub, GitLab, and your command line tools to handle the entire workflow — reading issues, writing code, running tests, and submitting PRs.

---

## Installation & Setup

```bash
# Install globally via npm
npm install -g @anthropic-ai/claude-code

# Authenticate with your Anthropic account
claude auth login

# Or login with Anthropic Console (API billing)
claude auth login --console

# Check auth status
claude auth status

# Update to latest version
claude update
```

You need a paid Claude.ai subscription (Pro or Max) or API access to use Claude Code. The free plan does not include terminal access.

Claude Code works on macOS, Linux, and Windows.

---

## Core CLI Commands

You can start sessions, pipe content, resume conversations, and manage updates with these commands:

| Command | What it does |
|---|---|
| `claude` | Start an interactive session |
| `claude "query"` | Start a session with an initial prompt |
| `claude -p "query"` | One-shot query — runs and exits (great for scripting) |
| `cat file \| claude -p "query"` | Pipe file content into Claude |
| `claude -c` | Continue the most recent conversation |
| `claude -c -p "query"` | Continue last session non-interactively |
| `claude -r "session-name" "query"` | Resume a session by ID or name |
| `claude update` | Update to the latest version |
| `claude auth login` | Sign in to your account |
| `claude auth logout` | Sign out |
| `claude auth status` | Show authentication status |
| `claude mcp` | Configure MCP servers |
| `claude agents` | List all configured sub-agents |

---

## Slash Commands (Inside a Session)

The `/help` command shows all available slash commands, including your custom commands from `.claude/commands/` and `~/.claude/commands/` directories, as well as any commands from connected MCP servers.

### Essential Slash Commands

| Command | What it does |
|---|---|
| `/help` | List all available slash commands |
| `/init` | Generate a `CLAUDE.md` file for your project |
| `/clear` | Reset the context window — fresh start |
| `/cost` | Show token usage and cost for this session |
| `/bug` | Report a bug directly to Anthropic |
| `/exit` | End the session |
| `/add-dir <path>` | Add an additional directory to the working context |
| `/model` | Switch models mid-session |
| `/insights` | Compile a detailed HTML usage report from your history |

### Planning & Workflow Commands

| Command | What it does |
|---|---|
| `/plan` | Have Claude analyze before acting (read-only first pass) |
| `/review` | Run a code review on current changes |
| `/diff` | Show pending changes |
| `/checkpoint` | Save a checkpoint of the current state |

---

## Important CLI Flags

Customize Claude Code's behavior with these command-line flags. `claude --help` does not list every flag, so a flag's absence from `--help` does not mean it is unavailable.

### Model & Effort

```bash
# Set the effort level for reasoning depth
claude --effort low       # Fast, lightweight tasks
claude --effort medium    # Default
claude --effort high      # More thorough
claude --effort max       # Maximum reasoning (slowest, most thorough)

# Use a specific model
claude --model opus       # Claude Opus (most capable)
claude --model sonnet     # Claude Sonnet (balanced, default)
claude --model haiku      # Claude Haiku (fastest)

# Fallback model if default is overloaded
claude -p --fallback-model sonnet "query"
```

### Session Control

```bash
# Continue last session
claude --continue

# Resume a named session
claude -r "auth-refactor" "Continue where we left off"

# Fork a session (create new ID instead of reusing)
claude --resume abc123 --fork-session

# Resume sessions linked to a pull request
claude --from-pr 123
```

### Directory & Context

```bash
# Add extra directories Claude can read and edit
claude --add-dir ../lib ../shared

# Connect to IDE automatically
claude --ide

# Enable Chrome browser integration
claude --chrome
```

### System Prompt Customization

```bash
# Replace the system prompt entirely
claude --system-prompt "You are a senior Go engineer"

# Load system prompt from a file
claude --system-prompt-file ./my-prompt.txt

# Append to the default system prompt
claude --append-system-prompt "Always write tests"

# Append from file
claude --append-system-prompt-file ./extra-rules.txt
```

### Permissions & Safety

```bash
# Skip permission prompts (use with caution in CI/CD only)
claude --dangerously-skip-permissions

# Allow specific tools without prompting
claude --allowedTools "Bash(git log *)" "Bash(git diff *)" "Read"

# Disable specific tools entirely
claude --disallowedTools "Edit" "Write"

# Disable all slash commands
claude --disable-slash-commands
```

### Non-Interactive / Scripting Mode

```bash
# One-shot print mode (ideal for CI/CD)
claude -p "Analyze this code for security issues"

# Output as JSON (includes cost, duration, turn count)
claude -p "query" --output-format json

# Limit number of turns in a session
claude -p "query" --max-turns 5

# Minimal/bare mode — skip auto-discovery for faster startup
claude --bare -p "query"
```

---

## In-Session Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Shift + Tab` | Cycle permission modes (default → auto → bypassPermissions) |
| `Ctrl + C` | Cancel current generation |
| `Ctrl + L` | Clear screen |
| `↑ / ↓` | Navigate prompt history |

---

## Referencing Files & Running Shell Commands

```bash
# Reference a file in your prompt using @
"Fix the bug in @src/auth/login.ts"

# Reference an entire directory
"Review all files in @src/components/"

# Run shell commands directly with ! (bypasses conversational mode)
!git status
!npm test
!ls -la
```

Using `!` bypasses Claude's conversational mode, which saves tokens compared to asking Claude to run the command for you.

---

## CLAUDE.md — Project Memory

`CLAUDE.md` is a Markdown file Claude reads automatically at the start of every session. It is your project's persistent instruction set.

```bash
# Generate one automatically
/init
```

```markdown
# CLAUDE.md example

## Project Overview
This is a Node.js REST API using Express and PostgreSQL.

## Conventions
- Use TypeScript strict mode
- All functions must have JSDoc comments
- Tests go in __tests__/ alongside the source file

## Commands
- `npm run dev` to start dev server
- `npm test` to run tests
- `npm run lint` to lint

## Important Files
- src/config/db.ts — database connection
- src/middleware/auth.ts — authentication logic
```

There are two scopes: **Global** (`~/.claude/CLAUDE.md`) applies to all projects, and a project-level `CLAUDE.md` applies only to that project.

---

## MCP — Connecting External Tools

MCP is one of Claude Code's most important extensions. With it, you can directly integrate external tools and services.

```bash
# Add an MCP server (HTTP transport)
claude mcp add --transport http github https://mcp.github.com

# Add a local MCP server
claude mcp add --transport stdio myserver ./path/to/server

# List configured MCP servers
claude mcp list

# Remove an MCP server
claude mcp remove github

# Start with MCP debug logging
claude --debug "mcp" "query"

# Expose Claude Code itself as an MCP server (for other agents)
claude mcp serve
```

Since early 2026, Claude Code uses Tool Search (lazy loading) for MCP tools by default, reducing context usage by approximately 95%, as tools are loaded on demand rather than all at once.

---

## Sub-Agents

Sub-agents are specialized Claude instances with their own context windows and personas. You define them as Markdown files:

```markdown
---
name: reviewer
description: Use for thorough code reviews
model: sonnet
color: orange
---

You are an expert code reviewer. Focus on security, performance, and maintainability.
Always output findings as a numbered list with severity levels.
```

```bash
# List configured sub-agents
claude agents

# Start a session using a specific agent
claude --agent reviewer

# Define agents dynamically via JSON flag
claude --agents '{"reviewer":{"description":"Reviews code","prompt":"You are a code reviewer"}}'
```

---

## Automation & CI/CD Patterns

```bash
# Analyze then fix in linked sessions
SESSION=$(claude -p "Analyze the test failures" --output-format json | jq -r '.session_id')
claude -p "Now fix the failures you identified" --session-id "$SESSION"

# Pipe git diff into Claude for review
git diff | claude -p "Review this diff for bugs and security issues"

# Error handling in scripts
RESULT=$(claude -p "query" --output-format json)
SUBTYPE=$(echo "$RESULT" | jq -r '.subtype')
if [ "$SUBTYPE" = "error" ]; then
  echo "Claude Code failed"
  exit 1
fi

# Pipe logs for analysis
cat server.log | claude -p "Summarize the errors and suggest fixes"
```

---

## Project Management

```bash
# Preview what would be deleted for a project
claude project purge ~/work/my-repo --dry-run

# Delete all Claude Code state for a project (transcripts, logs, history)
claude project purge ~/work/my-repo

# Delete state for all projects
claude project purge --all

# Generate a long-lived OAuth token for CI
claude setup-token
```

---

## Key Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Your API key (for Console/API billing users) |
| `CLAUDE_CODE_EFFORT_LEVEL` | Default effort level (`low` / `medium` / `high` / `max`) |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | Directory for debug logs |
| `CLAUDE_CODE_SIMPLE` | Set by `--bare` flag; disables auto-discovery |

Variables can be set permanently in your `settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EFFORT_LEVEL": "medium"
  }
}
```

---

## Supported Models

Claude Code works with the Opus 4.7, Sonnet 4.6, and Haiku 4.5 models. Enterprise users can run Claude Code using models in existing Amazon Bedrock or Google Cloud Vertex AI instances.

| Model | Best for |
|---|---|
| `claude-opus-4-7` | Complex architecture, hardest problems |
| `claude-sonnet-4-6` | Everyday coding tasks (default) |
| `claude-haiku-4-5` | Fast, lightweight tasks and scripting |

---

## Quick Reference Cheatsheet

```bash
# Start fresh
claude

# One-shot (scripting)
claude -p "query"

# Continue last session
claude -c

# High-effort session with Opus
claude --model opus --effort high

# Pipe a file
cat main.py | claude -p "Find all bugs"

# Add extra directories
claude --add-dir ../shared ../lib

# Non-interactive with JSON output
claude -p "query" --output-format json --max-turns 3

# Fast bare mode for CI
claude --bare -p "Run lint checks"

# Generate project memory file
claude  →  /init

# Reset context
/clear

# Check cost
/cost
```
