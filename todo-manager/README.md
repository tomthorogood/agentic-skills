# todo-manager

An agentic skill for managing personal todos, chores, and recurring tasks stored as Markdown files in a GitHub repository.

## How It Works

The agent reads and writes individual `.md` files in a GitHub repo you control. Each file represents one task, with structured frontmatter (cadence, urgency, seasonality, learnings, etc.) and optional freeform notes.

## Setup

### 1. Create a todos repository

Create a GitHub repo (public or private) and a directory inside it for your todos. Example: `my-username/todos`, directory `managed/`.

### 2. Connect GitHub MCP

This skill requires a GitHub MCP connector in your agent environment. 

### Claude 

#### Web client

In Claude.ai, you can try enabling the GitHub connector via the tools menu. 

#### Desktop

Create a [PAT]() with read/write permissions on your TODO repo, and add the following configuration to

(MacOS): `/Users/$USER/Library/Application Support/Claude/claude_desktop_config.json`

Make sure to replace `$GITHUB_PAT` with your PAT.
```
  "mcpServers": {
    "github": {
      "command": "/opt/homebrew/bin/npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_PAT"
      }
    }
  }
```

### VS Code agents

`CMD+Shift+P` > `MCP: Open user configuration` 

and add the following entry:

```
"github": {
			"type": "http",
			"url": "https://api.githubcopilot.com/mcp/",
			"headers": {
				"X-MCP-Toolsets": "actions,context,copilot,copilot_spaces,dependabot,discussions,git,github_support_docs_search,issues,labels,notifications,orgs,projects,pull_requests,repos"
			}
		}
```

Edit down the "Toolsets" list to only the capabilities you plan to use.

VS Code should prompt you to authenticate when you run the server for the first time.

### 3. Provide configuration

The skill needs four values:

| Key | Description | Example |
|---|---|---|
| `GITHUB_OWNER` | GitHub username or org | `alice` |
| `GITHUB_REPO` | Repository name | `todos` |
| `GITHUB_BRANCH` | Branch to read/write | `main` |
| `TODO_DIR` | Directory for todo files | `managed` |

**If you don't provide these in advance**, the agent will ask for them at the start of the session. You'll need to re-provide them each session unless you save them somewhere persistent.

### Persisting configuration (recommended)

Depending on your agent surface, you can store these values so you never have to type them again. Some examples:

- **Claude.ai Projects** — Add them to the project's custom instructions
- **VS Code custom agents** — Add to the agent's system prompt in `.vscode/agents/`
- **GitHub Copilot Spaces** — Add to the space system prompt
- **Any other agent surface** — Add to whatever system prompt or context file that surface supports

Example snippet to paste into your project/agent instructions:

```
todo-manager config:
GITHUB_OWNER: alice
GITHUB_REPO: todos
GITHUB_BRANCH: main
TODO_DIR: managed
```

## Features

- Add and update tasks with structured metadata
- Mark tasks complete (auto-updates `last_performed` and `next_expected`)
- Prioritized suggestions based on overdue status, urgency, seasonality, and available time
- Per-task learnings that accumulate over time
- Global context file for cross-task preferences and constraints

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
