# todo-manager

An agentic skill for managing personal todos, chores, and recurring tasks stored as Markdown files.

## How It Works

The agent reads and writes individual `.md` files in a storage location you control. Each file represents one task, with structured frontmatter (cadence, urgency, seasonality, learnings, etc.) and optional freeform notes. A compact `index.md` file is maintained automatically so planning queries only require a single read, regardless of how many todos you have.

## Storage

This skill works with any file storage backend the agent has access to. On first use, the agent will inspect available tools, present the options it finds, and ask you to choose one. If nothing suitable is available, it will search for connectors you could add.

Supported storage types include (but are not limited to):

- **GitHub repository** — version history, accessible from any agent surface, easy to edit manually
- **Google Drive** — familiar interface, easy to browse and share
- **Local filesystem** — fast and private, available in desktop/CLI agent contexts
- **Any other MCP-connected file storage** — the agent will detect and offer what's available

If no storage is reachable, the agent falls back to a session log and shows you what to save when you're done.

## Setup

### 1. Create a storage location

Create a folder or directory where your todos will live. For example:

- A GitHub repo (`my-username/todos`) with a `managed/` directory
- A Google Drive folder called `My Todos`
- A local directory like `~/todos`

### 2. Connect the relevant tool or MCP server

The agent needs access to tools that can read and write files in your chosen storage. How you connect these depends on your agent environment:

#### Claude.ai / Cowork

Enable connectors via the tools or connectors menu (GitHub, Google Drive, etc.). The agent will detect what's connected automatically.

#### Claude Desktop

Add the relevant MCP server to your config file at:

`(macOS): ~/Library/Application Support/Claude/claude_desktop_config.json`

Example for GitHub (replace `$GITHUB_PAT` with a [personal access token](https://github.com/settings/tokens)):

```json
{
  "mcpServers": {
    "github": {
      "command": "/opt/homebrew/bin/npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_PAT"
      }
    }
  }
}
```

#### VS Code agents

`CMD+Shift+P` > `MCP: Open user configuration` and add the relevant server. Example for GitHub:

```json
"github": {
  "type": "http",
  "url": "https://api.githubcopilot.com/mcp/",
  "headers": {
    "X-MCP-Toolsets": "git,issues,repos,pull_requests"
  }
}
```

### 3. Provide configuration

The skill needs two values to operate:

| Key | Description | Example |
|---|---|---|
| `TODO_DIR` | Where todo files are stored | `managed` (GitHub dir), `My Todos` (Drive folder), `~/todos` (local path) |
| `LOCATION` | Your location, for seasonal task reasoning | `Seattle, WA, USA` |

Depending on your backend, you may also need:

| Key | Description |
|---|---|
| `GITHUB_OWNER` / `GITHUB_REPO` / `GITHUB_BRANCH` | For GitHub backends |
| `GDRIVE_FOLDER_ID` | For Google Drive backends (optional; the agent can search by name) |

**If you don't provide these in advance**, the agent will ask at the start of the session. You'll need to re-provide them each session unless you save them somewhere persistent.

### Persisting configuration (recommended)

Depending on your agent surface, you can store these values so you never have to type them again:

- **Claude.ai Projects** — add to the project's custom instructions
- **VS Code custom agents** — add to the agent's system prompt in `.vscode/agents/`
- **GitHub Copilot Spaces** — add to the space system prompt
- **Any other agent surface** — add to whatever system prompt or context file that surface supports

Example snippet to paste into your project/agent instructions:

```
todo-manager config:
TODO_DIR: managed
LOCATION: Austin, TX, USA
STORAGE_BACKEND: github
GITHUB_OWNER: alice
GITHUB_REPO: todos
GITHUB_BRANCH: main
```

## Features

- Add and update tasks with structured metadata
- Mark tasks complete (auto-updates `last_performed` and `next_expected`)
- Prioritized suggestions based on overdue status, urgency, seasonality, and available time
- Seasonality reasoning adapts to your location (hemisphere, climate)
- Per-task learnings that accumulate over time
- Global context file for cross-task preferences and constraints
- Planning index (`index.md`) for fast suggestions regardless of todo list size
- Works with any file storage backend — GitHub, Google Drive, local, and more

## File Format

Each todo is a Markdown file with YAML frontmatter:

```markdown
---
title: Mow the lawn
cadence: every 2 weeks
last_performed: 2025-03-01
next_expected: 2025-03-15
estimated_duration: 45 minutes
urgency: medium
seasonality: spring through fall
requires: [lawnmower]
learnings:
  - Works best before noon
---
```

A compact `index.md` is maintained automatically alongside your todo files. It summarizes all todos in a single table for fast planning reads — you don't need to manage it manually.
