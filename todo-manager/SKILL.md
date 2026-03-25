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
| `LOCATION` | User's location, used to apply accurate seasonal context | `Seattle, WA, USA` |

### Config Resolution (in order)

1. **Session/project context** — If these values are present in your system prompt, project instructions, or agent configuration, use them directly. Do not prompt the user.
2. **Conversation context** — If the user has already provided them earlier in the conversation, use those.
3. **Prompt the user** — If config is missing, ask for all required values at once (one message, not one at a time). Then use them for the remainder of the session.

When prompting for config, include this notice:

> These values are needed each session unless saved somewhere persistent. Depending on your setup, you can store them in a Claude Project's instructions, a VS Code custom agent config, a GitHub Copilot Space system prompt, or a similar surface. See the [todo-manager README](https://github.com/tomthorogood/agentic-skills/blob/main/todo-manager/README.md) for details.

The global learnings file is always at `{TODO_DIR}/context.md` within the configured repo.

### Seasonality and Location

Use `LOCATION` to apply accurate seasonal context when scoring or describing tasks. A city and country/region is sufficient. If `LOCATION` is not provided, fall back to Northern Hemisphere defaults and note the assumption.

Do not use `LOCATION` for any purpose other than seasonal reasoning (e.g., do not infer timezone, language, or cultural preferences from it).

---

## GitHub Access

At the start of every session, determine whether the GitHub MCP connector is available.

- **If GitHub MCP is available**: operate normally. Read from and write to the repo directly.
- **If GitHub MCP is unavailable**: enter offline mode. Do not stop. Inform the user once:

  > "GitHub MCP is not available. I'll keep a pending changes log this session. When you're back in a connected context, sync it to apply the changes."

  Then proceed to record all intended changes in the pending changes log (see below). Do not repeat the offline notice for every action.

All reads and writes in connected mode use the configured `GITHUB_OWNER`, `GITHUB_REPO`, `GITHUB_BRANCH`, and `TODO_DIR`.

---

## Pending Changes Log (Offline Mode)

When GitHub MCP is unavailable, maintain a running log of all changes that would have been committed. Present this log inline in the conversation, updated after each action.

### Format

```markdown
# Pending Changes

> Generated: 2025-03-25. Sync these changes when GitHub MCP is available.

## update: mow-lawn
- last_performed: 2025-03-25
- next_expected: 2025-04-08
- learnings: [append] Skipped bagging due to dry conditions.

## add: clean-gutters
- title: Clean gutters
- cadence: every 6 months
- estimated_duration: 2 hours
- urgency: medium
- seasonality: spring and fall
- requires: [ladder]

## delete: fix-fence
- reason: one-time task completed
```

### Rules

- Each entry uses a header matching the intended commit message format (`add:`, `update:`, `delete:`).
- List only the fields that would change, not the full file.
- For `learnings`, note whether the entry is an append or a replacement.
- Do not silently drop changes — every intended write must appear in the log.
- At the end of the session, present the complete log and remind the user to sync it.

### Syncing Pending Changes

When the user asks to sync (or when a session begins with MCP available and a pending log is provided):

1. Apply each entry in the log to the corresponding file in the repo.
2. Commit each change with its logged message.
3. Confirm each applied change briefly — one line per item.
4. Note any conflicts (e.g., a file was modified in the repo since the log was generated) and ask the user how to resolve.

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
- **seasonality**: Derive from task context and `LOCATION` unless stated. Use natural language. If a task has no seasonal constraint, omit the field or write `year-round`.
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

In offline mode, changes to `context.md` are included in the pending changes log under the header `update: context`.

---

## Core Workflows

### 1. Adding or Updating a Todo

When the user describes a task (new or existing):

1. Check if a matching file already exists in `{TODO_DIR}/` (skip in offline mode — work from session context).
2. If new: derive a slug, populate all fields, ask about cadence if unclear.
3. If existing: update only the changed fields. Never overwrite `learnings` — append.
4. In connected mode: commit with a brief message: `"add: mow-lawn"` or `"update: mow-lawn — marked complete"`.
5. In offline mode: add the change to the pending log.

**Ask about cadence if** the user hasn't stated it and it isn't obvious. One question, concisely:
> "How often does this need to happen?"

Do not ask for fields you can reasonably derive.

### 2. Marking a Task Complete

When the user says they did something:

1. Set `last_performed` to today's date.
2. If the task has `cadence: once` or is otherwise a one-time task:
   - **Connected mode**: delete the file and commit with `"delete: <slug> — one-time task completed"`.
   - **Offline mode**: add a `delete:` entry to the pending log.
   - **If file deletion is not available**: provide a direct link to the file and prompt the user to delete it manually.
3. For recurring tasks: derive and set `next_expected` from cadence, append any learnings, and commit the update (or log it in offline mode).
4. Confirm briefly — one line.

If the user mentions a constraint while completing ("I had to borrow the neighbor's ladder"), add it to `requires` and/or `learnings` before deleting or updating.

### 3. Daily / Session Suggestions

**Only offer suggestions when explicitly prompted.** Example triggers:
- "What should I do today?"
- "I have 3 hours Saturday morning, what's most important?"
- "What's due this week?"

When prompted:

1. Read `{TODO_DIR}/context.md` for global constraints (connected mode only; use session context in offline mode).
2. Read all todo files (or as many as needed via directory listing + selective fetch).
3. Score and rank tasks using all of the following, weighted together:
   - **Overdue status**: Days past `next_expected` (higher = more urgent)
   - **Explicit urgency**: `critical` > `high` > `medium` > `low`
   - **Seasonal fit**: Is this task in-season right now, given `LOCATION` and current date? Out-of-season tasks deprioritized.
   - **Time fit**: Does `estimated_duration` fit within the user's stated available time?
4. Present only the tasks the user should act on. Do not mention excluded tasks or explain why they were omitted.
5. Do not present more than ~5–7 suggestions unless asked.

Format suggestions as a simple numbered list. No headers, no fluff.

### 4. Capturing Learnings Mid-Conversation

If the user says something that reveals a durable constraint or preference:
- Update the relevant todo's `learnings` or `requires` fields.
- If it's global, append to `{TODO_DIR}/context.md`.
- In offline mode, add to the pending log.
- Do this silently unless a commit or log confirmation is needed. Minimize interruption.

---

## Behavioral Rules

- **Do not volunteer suggestions** unless explicitly asked.
- **Do not ask multiple questions at once.** If clarification is needed, ask one thing.
- **Do not over-explain updates.** A one-line confirmation is sufficient.
- **Derive, don't ask**, when the answer is reasonably inferrable from context.
- **Prefer seamless commits.** The user can review history in GitHub at any time.
- **Seasonality is derived by default.** Use `LOCATION` and the current date. Only ask if genuinely ambiguous.
- **Respect the user's edits.** If a field in the repo differs from what you'd expect, assume the user changed it intentionally.
- **Offline mode is not a degraded state.** Log changes faithfully and keep the session productive.
- **Only surface actionable output.** When presenting suggestions, list only what the user should do. Do not explain why tasks were deprioritized, deferred, or excluded.

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

## Seasonality Reference

Use `LOCATION` to determine hemisphere and local climate. The defaults below assume Northern Hemisphere temperate climate; adjust for Southern Hemisphere (invert months) or tropical/arid climates (use wet/dry seasons instead) as appropriate.

- **Gardening / outdoor planting**: March–October active; February for prep; November–January dormant
- **Lawn care**: March–November; peak April–September
- **Snow / ice removal**: December–February (weather-dependent)
- **Gutter cleaning**: Spring (April) and Fall (October–November)
- **HVAC filter / furnace service**: Fall (before heating season) and Spring (before cooling season)
- **General indoor tasks**: Year-round
