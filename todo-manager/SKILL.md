---
name: todo-manager
description: Manage personal todos, chores, and recurring adulting tasks stored as Markdown files in a GitHub repository. Use this skill whenever the user mentions tasks, chores, todos, household responsibilities, gardening, errands, or asks what they should do today/this weekend. Also use when the user wants to log that they completed something, add a new task, ask what's due, plan their day or week, or reports context that should be remembered (e.g., "I don't have the car today"). Trigger proactively even if the user doesn't say "todo" explicitly — phrases like "what should I work on", "I have a free afternoon", or "I just finished X" all warrant this skill.
---

# Todo Manager Skill

Manages personal todos stored as individual Markdown files in a GitHub repository, under a configurable directory.

---

## Configuration

This skill requires the following configuration values before it can operate:

| Key | Description | Example |
|---|---|---|
| `GITHUB_OWNER` | GitHub username or org that owns the todos repo | `alice` |
| `GITHUB_REPO` | Repository name | `todos` |
| `GITHUB_BRANCH` | Branch to read/write | `main` |
| `TODO_DIR` | Directory within the repo where todo files live | `managed` |

### Config Resolution (in order)

1. **Session/project context** — If these values are present in your system prompt, project instructions, or agent configuration, use them directly. Do not prompt the user.
2. **Conversation context** — If the user has already provided them earlier in the conversation, use those.
3. **Prompt the user** — If config is missing, ask for all required values at once (one message, not one at a time). Then use them for the remainder of the session.

When prompting for config, include this notice:

> These values are needed each session unless saved somewhere persistent. Depending on your setup, you can store them in a Claude Project's instructions, a VS Code custom agent config, a GitHub Copilot Space system prompt, or a similar surface. See the [todo-manager README](https://github.com/tomthorogood/agentic-skills/blob/main/todo-manager/README.md) for details.

The global learnings file is always at `{TODO_DIR}/context.md` within the configured repo.

---

## GitHub Access

**This skill requires the GitHub MCP connector.** If GitHub MCP tools are unavailable, stop and tell the user: "GitHub MCP is not available — cannot proceed." Do not fall back to manual file editing instructions.

All reads and writes use the configured `GITHUB_OWNER`, `GITHUB_REPO`, `GITHUB_BRANCH`, and `TODO_DIR`.

---

## File Format

Each todo is a single Markdown file at `{TODO_DIR}/<slug>.md`. The slug is lowercase, hyphenated (e.g., `mow-lawn.md`).

```markdown
---
title: Mow the lawn
cadence: every 2 weeks
last_performed: 2024-05-01
next_expected: 2024-05-15
estimated_duration: 45 minutes
urgency: medium
seasonality: spring through fall; no mowing in winter; light prep in late February
requires: [lawnmower, car for fuel if needed]
learnings:
  - Works best done before noon to avoid heat
  - Skip if rain is expected within 24 hours
---

Optional freeform notes here.
```

### Field Rules

- **cadence**: Human-readable (e.g., "weekly", "every 3 months", "once", "as needed"). If unclear, ask.
- **last_performed**: `YYYY-MM-DD` or `null`
- **next_expected**: `YYYY-MM-DD` or `null`. Derive from `last_performed + cadence` when possible.
- **estimated_duration**: Use natural units — minutes, hours, or days. Be specific (e.g., "2 hours" not "long").
- **urgency**: One of `low`, `medium`, `high`, `critical`. Derive from context unless stated. Consider overdue status, season, and consequences of delay.
- **seasonality**: Derive from task context unless stated. Use natural language. If a task has no seasonal constraint, omit the field or write `year-round`.
- **requires**: List of dependencies inferred or stated (tools, vehicle, weather, etc.). Update from user context.
- **learnings**: Bullet list. Append new entries; never delete existing ones unless the user explicitly asks.

---

## Global Learnings File

`{TODO_DIR}/context.md` stores preferences and learned context that apply across all todos. Read this file at the start of any session involving planning or suggestions.

Append to it when the user reveals something durable and general:
- Scheduling constraints ("I don't drive on Sundays")
- Resource availability ("I only have the car on weekdays")
- Physical constraints ("I can't do heavy lifting alone")
- Preferences ("I prefer outdoor tasks in the morning")

Do not append transient context (e.g., "I'm tired today").

Format entries as bullet points with a derived date:
```markdown
- [2024-05-10] Does not have access to the car on Sundays.
- [2024-05-12] Prefers outdoor tasks completed before noon.
```

---

## Core Workflows

### 1. Adding or Updating a Todo

When the user describes a task (new or existing):

1. Check if a matching file already exists in `{TODO_DIR}/`.
2. If new: derive a slug, populate all fields, ask about cadence if unclear.
3. If existing: update only the changed fields. Never overwrite `learnings` — append.
4. Commit with a brief message: `"add: mow-lawn"` or `"update: mow-lawn — marked complete"`.

**Ask about cadence if** the user hasn't stated it and it isn't obvious. One question, concisely:
> "How often does this need to happen?"

Do not ask for fields you can reasonably derive.

### 2. Marking a Task Complete

When the user says they did something:

1. Set `last_performed` to today's date.
2. Derive and set `next_expected` from cadence (or set to `null` if one-off).
3. Append any relevant learnings from the user's description.
4. Commit the update.
5. Confirm briefly — one line.

If the user mentions a constraint while completing ("I had to borrow the neighbor's ladder"), add it to `requires` and/or `learnings`.

### 3. Daily / Session Suggestions

**Only offer suggestions when explicitly prompted.** Example triggers:
- "What should I do today?"
- "I have 3 hours Saturday morning, what's most important?"
- "What's due this week?"

When prompted:

1. Read `{TODO_DIR}/context.md` for global constraints.
2. Read all todo files (or as many as needed via directory listing + selective fetch).
3. Score and rank tasks using all of the following, weighted together:
   - **Overdue status**: Days past `next_expected` (higher = more urgent)
   - **Explicit urgency**: `critical` > `high` > `medium` > `low`
   - **Seasonal fit**: Is this task in-season right now? Out-of-season tasks deprioritized.
   - **Time fit**: Does `estimated_duration` fit within the user's stated available time?
4. Present a short, prioritized list. Include estimated duration and a one-line reason for each.
5. Do not present more than ~5–7 suggestions unless asked.

Format suggestions as a simple numbered list. No headers, no fluff.

### 4. Capturing Learnings Mid-Conversation

If the user says something that reveals a durable constraint or preference:
- Update the relevant todo's `learnings` or `requires` fields.
- If it's global, append to `{TODO_DIR}/context.md`.
- Do this silently unless a commit confirmation is needed. Minimize interruption.

---

## Behavioral Rules

- **Do not volunteer suggestions** unless explicitly asked.
- **Do not ask multiple questions at once.** If clarification is needed, ask one thing.
- **Do not over-explain updates.** A one-line confirmation is sufficient.
- **Derive, don't ask**, when the answer is reasonably inferrable from context.
- **Prefer seamless commits.** The user can review history in GitHub at any time.
- **Seasonality is derived by default.** Use the current date and task context to determine it. Only ask if genuinely ambiguous.
- **Respect the user's edits.** If a field in the repo differs from what you'd expect, assume the user changed it intentionally.

---

## Urgency Derivation Guide

Use this to derive urgency when not stated:

| Signal | Urgency |
|---|---|
| Overdue by 2+ cadence cycles | `high` or `critical` |
| Overdue by 1 cycle | `medium` |
| Due within this week | `medium` |
| Not yet due | `low` |
| Has downstream consequences (e.g., pest damage, safety) | Bump up one level |
| One-off with no deadline | `low` unless user states otherwise |

---

## Seasonality Reference (Northern Hemisphere defaults)

Adjust based on learnings if user location is known.

- **Gardening / outdoor planting**: March–October active; February for prep; November–January dormant
- **Lawn care**: March–November; peak April–September
- **Snow / ice removal**: December–February (weather-dependent)
- **Gutter cleaning**: Spring (April) and Fall (October–November)
- **HVAC filter / furnace service**: Fall (before heating season) and Spring (before cooling season)
- **General indoor tasks**: Year-round
