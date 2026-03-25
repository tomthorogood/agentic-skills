---
name: todo-manager
description: Manage personal todos, chores, and recurring adulting tasks stored as Markdown files. Use this skill whenever the user mentions tasks, chores, todos, household responsibilities, gardening, errands, or asks what they should do today/this weekend. Also use when the user wants to log that they completed something, add a new task, ask what's due, plan their day or week, or reports context that should be remembered (e.g., "I don't have the car today"). Trigger proactively even if the user doesn't say "todo" explicitly — phrases like "what should I work on", "I have a free afternoon", or "I just finished X" all warrant this skill.
---

# Todo Manager Skill

Manages personal todos stored as individual Markdown files, with a pluggable storage backend. On first use, the skill inspects available tools to discover storage options and asks the user to choose one.

---

## First Run: Storage Detection

Before doing anything else, check whether `STORAGE_BACKEND` and `TODO_DIR` are already configured (see Configuration below). If they are, skip this section entirely.

If not configured, run storage detection:

### Step 1 — Inspect available tools

Silently examine the tools available in the current session. Do not narrate this to the user. You are looking for toolsets that can perform the four storage operations this skill requires: **list**, **read**, **write**, and **delete** files in a persistent location.

Assess each available toolset against these criteria:

| Capability | What to look for |
|---|---|
| **List** | Can you enumerate files or items in a container (directory, folder, repo, collection)? |
| **Read** | Can you retrieve the full text content of a named file or document? |
| **Write** | Can you create or overwrite a named file or document with arbitrary text content? |
| **Delete** | Can you permanently remove a named file or document? |

A toolset qualifies as a **full backend** if it supports list, read, and write. Delete support is noted but not required — if unavailable, the skill will prompt the user for manual deletion when needed.

Characterize each qualifying toolset in plain terms (e.g., "a GitHub repository", "a Google Drive folder", "the local filesystem"). Do not refer to tool names or API internals when presenting options to the user.

### Step 2 — Present options

If one or more qualifying toolsets are found, present them to the user:

> I can store your todos using the following storage options I have access to right now:
>
> [list each option with a one-sentence description of what it means for the user — where files live, tradeoffs like version history, shareability, or device accessibility]
>
> Which would you like to use? If none of these work for you, I can search for other options you might be able to connect.

If the user asks for other options, or if no qualifying toolsets are found, proceed to Step 3.

### Step 3 — Search for additional options

If a `search_mcp_registry` tool (or equivalent connector/plugin registry search) is available, use it to find storage-capable integrations. Search for terms like "file storage", "drive", "notes", "documents", or the name of a specific service the user mentioned.

Present relevant results plainly:

> Here are some storage integrations you could connect to use with this skill:
>
> [list results with a brief description of each]
>
> If you connect one of these, I can set up your todos there.

If a `suggest_connectors` tool is available, call it to streamline the connection flow for the user.

### Step 4 — No options at all

If no qualifying toolsets are found and no registry search is available (or returns nothing useful):

> I don't have access to any persistent file storage right now. I can keep a session log of your todos and show you what to save when we're done — but changes won't persist automatically.
>
> If you can connect a file storage service (like a cloud drive, notes app, or version control system), I'll be able to save your todos there. Let me know if you'd like help finding an option.

Then enter offline mode and maintain the pending changes log for the session.

### Step 5 — Gather required details

Once the user has chosen a backend, ask for any details needed to locate or create the todo storage location (e.g., a folder name, repo name, or path). Ask for all required values in one message. Store them as `TODO_DIR` and any backend-specific config.

Include this notice when prompting:

> These values are needed each session unless saved somewhere persistent. You can store them in a Claude Project's instructions, an agent config file, or a similar surface. See the [todo-manager README](https://github.com/tomthorogood/agentic-skills/blob/main/todo-manager/README.md) for details.

---

## Configuration

| Key | Description | Example |
|---|---|---|
| `STORAGE_BACKEND` | A label identifying the active backend | `github`, `gdrive`, `local`, or any descriptive name |
| `TODO_DIR` | The storage location for todo files | A folder name, repo path, drive folder ID, local path, etc. |
| `LOCATION` | User's location for seasonal context | `Seattle, WA, USA` |

Additional backend-specific keys (e.g., repo owner, branch, folder ID) should be stored alongside these and referenced consistently throughout the session.

### Config Resolution (in order)

1. **Session/project context** — if present in system prompt, agent config, or project instructions, use directly. Do not prompt.
2. **Conversation context** — if the user provided them earlier this session, use those.
3. **First-run detection** — run the storage detection flow above.

### Seasonality and Location

Use `LOCATION` to apply accurate seasonal context when scoring or describing tasks. A city and country/region is sufficient. If not provided, fall back to Northern Hemisphere temperate defaults and note the assumption.

Do not use `LOCATION` for any purpose other than seasonal reasoning.

---

## Storage Operations

All reads and writes go through the active backend. The four logical operations are:

- **list** — enumerate all todo slugs in `TODO_DIR`
- **read(path)** — retrieve the full content of a file
- **write(path, content)** — create or overwrite a file
- **delete(path)** — permanently remove a file

Map these to whatever tools the active backend provides. If `delete` is unavailable, provide the user a direct reference to the file (link, path, or location) and prompt them to delete it manually. Log this as a pending action.

### Offline / No Backend

If the chosen backend becomes unavailable mid-session, enter offline mode. Inform the user once, then maintain a pending changes log (see below). Do not repeat the offline notice.

---

## Planning Index (`index.md`)

`{TODO_DIR}/index.md` is a compact planning summary of all todos. It is the **primary read target** for any planning or suggestion workflow — one read operation regardless of how many todos exist.

### Format

```markdown
# Todo Index

> Planning summary for all managed todos. Maintained by the todo-manager skill — do not edit manually.
> Last updated: YYYY-MM-DD

| slug | title | cadence | last_performed | next_expected | urgency | seasonality | estimated_duration |
|---|---|---|---|---|---|---|---|
| mow-lawn | Mow the lawn | every 2 weeks | 2024-05-01 | 2024-05-15 | medium | spring through fall | 45 minutes |
```

### Rules

- **Read `index.md` first** for all planning, scoring, and suggestion workflows. Do not fetch individual todo files unless you need `learnings` or freeform notes.
- **Update `index.md` on every write.** Whenever a todo file is created, updated, or deleted, update the corresponding row. Never let the index fall out of sync.
- **Deletions**: remove the row entirely.
- **New todos**: append a new row.
- The index does not include `learnings`, `requires`, or freeform notes — only fields needed for scoring and ranking.
- In offline mode, include index updates in the pending changes log.

---

## Pending Changes Log (Offline / Unavailable Backend)

When the storage backend is unavailable, maintain a running log of all intended changes. Present this log inline in the conversation, updated after each action.

### Format

```markdown
# Pending Changes

> Generated: 2025-03-25. Apply these when storage is available.

## update: mow-lawn
- last_performed: 2025-03-25
- next_expected: 2025-04-08
- learnings: [append] Skipped bagging due to dry conditions.

## update: index.md
- mow-lawn row: last_performed → 2025-03-25, next_expected → 2025-04-08

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

- Each entry uses a header matching the intended operation (`add:`, `update:`, `delete:`).
- List only the fields that would change, not the full file.
- Always include a corresponding `update: index.md` entry for any todo that is added, updated, or deleted.
- Do not silently drop changes — every intended write must appear in the log.
- At the end of the session, present the complete log and remind the user to sync it.

### Syncing

When the user asks to sync (or storage becomes available again):

1. Apply each entry to the corresponding file via the active backend.
2. Confirm each applied change briefly — one line per item.
3. Note any conflicts (e.g., a file was modified externally since the log was generated) and ask the user how to resolve.

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
- **seasonality**: Derive from task context and `LOCATION` unless stated. If no seasonal constraint, omit or write `year-round`.
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

In offline mode, changes to `context.md` go in the pending log under `update: context`.

---

## Core Workflows

### 1. Adding or Updating a Todo

1. Check `{TODO_DIR}/index.md` for a matching row.
2. If new: derive a slug, populate all fields, ask about cadence if unclear.
3. If existing: update only changed fields. Never overwrite `learnings` — append.
4. Write the todo file, then update `index.md`.
5. In offline mode: log both changes to the pending log.

**Ask about cadence only if** the user hasn't stated it and it isn't obvious:
> "How often does this need to happen?"

Do not ask for fields you can reasonably derive.

### 2. Marking a Task Complete

1. Set `last_performed` to today's date.
2. If `cadence: once` or a one-time task:
   - Delete the file (or prompt user if deletion isn't available) and remove its row from `index.md`.
   - In offline mode: log a `delete:` entry and an `update: index.md` entry.
3. For recurring tasks: derive `next_expected`, append any learnings, update the file and index row.
4. Confirm briefly — one line.

If the user mentions a constraint while completing, add it to `requires` and/or `learnings` first.

### 3. Daily / Session Suggestions

**Only offer suggestions when explicitly prompted.**

1. Read `{TODO_DIR}/context.md` for global constraints.
2. Read `{TODO_DIR}/index.md` to get all todos in one operation. Do not fetch individual files for planning.
3. Score and rank using:
   - **Overdue status**: days past `next_expected`
   - **Explicit urgency**: `critical` > `high` > `medium` > `low`
   - **Seasonal fit**: in-season given `LOCATION` and current date
   - **Time fit**: `estimated_duration` vs. stated available time
4. Present only actionable tasks — no more than 5–7 unless asked. Do not explain omissions.

Format as a simple numbered list. No headers, no fluff.

Fetch individual files only if the user asks for learnings, notes, or requirements on a specific task.

### 4. Capturing Learnings Mid-Conversation

When the user reveals a durable constraint or preference:
- Update the relevant todo's `learnings`/`requires`, and update its index row if any indexed fields changed.
- If global, append to `context.md`.
- Do this silently unless a write confirmation is needed.

---

## Behavioral Rules

- **Do not volunteer suggestions** unless explicitly asked.
- **Do not ask multiple questions at once.** One question at a time.
- **Do not over-explain updates.** A one-line confirmation is sufficient.
- **Derive, don't ask**, when the answer is reasonably inferrable.
- **Seasonality is derived by default.** Use `LOCATION` and current date. Only ask if genuinely ambiguous.
- **Respect the user's edits.** If a field in storage differs from what you'd expect, assume it was changed intentionally.
- **Offline mode is not a degraded state.** Log changes faithfully and keep the session productive.
- **Only surface actionable output.** Do not explain why tasks were excluded or deprioritized.
- **Fetch individual files on demand only.** Always use `index.md` for planning. Fetch full files only when writing or when the user needs detail.
- **Never hardcode storage assumptions.** Always derive available backends from the tools present in the session.

---

## Urgency Derivation Guide

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

Use `LOCATION` to determine hemisphere and local climate. Defaults below assume Northern Hemisphere temperate; invert months for Southern Hemisphere, or use wet/dry seasons for tropical/arid climates.

- **Gardening / outdoor planting**: March–October active; February for prep; November–January dormant
- **Lawn care**: March–November; peak April–September
- **Snow / ice removal**: December–February (weather-dependent)
- **Gutter cleaning**: Spring (April) and Fall (October–November)
- **HVAC filter / furnace service**: Fall (before heating season) and Spring (before cooling season)
- **General indoor tasks**: Year-round
