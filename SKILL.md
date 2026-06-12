---
name: deepwork-timer
description: track tasks and focus sessions on deepworktimer.com — use whenever the user mentions dwt, tracking work time, pomodoros, focus sessions, Deep Work Timer tasks/projects/skills, or when starting substantive work that should be time-tracked
---

# Deep Work Timer

Manage the user's tasks and log focused work time on their Deep Work Timer
account (https://deepworktimer.com) via the bundled CLI:

```
~/.claude/skills/deepwork-timer/bin/dwt
```

This file is authoritative for the CLI's behavior — trust it over exploring
the script. All commands print friendly errors; a 401 means the token is
missing/expired — see Onboarding below.

## Sessions are SERVER-SIDE (important)

`dwt start` creates a session on the server, not just locally:

- **One active session per account.** Starting while another machine (or the
  phone PWA) has a session running TAKES IT OVER — the old segment is closed
  and attributed, and `dwt start` prints a `took over` warning. Check
  `dwt status` before starting if unsure, and always surface takeovers.
- The session has a fixed duration (the user's pomodoro length, default 25m)
  and an `ends_at`. Past `ends_at` it is **overdue**, not recorded: nothing
  lands in stats until `dwt log` completes it (or `dwt abandon` discards it).
- At `ends_at` the server sends a Web Push to the user's subscribed devices.

## Commands

| Command | What it does |
|---|---|
| `dwt login <email> [-p <pw>]` | Get and store an API token (`~/.deepworktimer/token`). |
| `dwt token <raw-token>` | Save a pasted API token directly (verifies it). |
| `dwt whoami` | Authenticated user (name, email, timezone, pomodoro settings). |
| `dwt projects` | List projects (id, name). |
| `dwt tasks [--project <id>]` | List tasks; without `--project`, the unassigned ones. Subtasks indented. |
| `dwt add "<desc>" [--project <id>] [--estimate <seconds>]` | Create a task. `#skill` words in the description become skill tags — but ONLY skills that already exist on the account attach (exact name match; the CLI warns when a `#tag` matched nothing). Existing skill names use underscores (e.g. `product_strategy`); `product-development` (id 587) exists for Claude-supervised work. Create missing skills via `POST /api/skills` with `{"skillName": "...", "parent_id": <optional>}`. |
| `dwt done <task_id>` | Mark a task completed. |
| `dwt start <task_id> [--minutes <n>]` | Start a SERVER session on the task (may take over — see above). `--minutes` sets a custom length (1–240; default is the account's pomodoro length). |
| `dwt switch <task_id>` | Re-point the running session at another task; one pomodoro, honestly split per segment. |
| `dwt abandon` | End the running session, logging nothing. |
| `dwt log` | Complete the running server session → records the pomodoro with its per-task splits (server computes them). `--task <id> --minutes <n>` logs a fixed direct pomodoro instead. |
| `dwt status` | Active session (task, remaining, segments, machine label) + today's total. |
| `dwt stats` | Today / 7d / 30d / 365d / all-time, best day, daily average. |
| `dwt map [<task_id>] [path]` | Map a working directory to a task for the multi-session arbiter (below). No args lists mappings; `--remove [path]` deletes; path defaults to the current directory; longest-prefix match wins. |
| `dwt statusline-install` | Wrap this machine's Claude Code statusline so a live `🍅 task · mm:ss left` segment is appended. Idempotent; only edits `~/.claude/settings.json` — every machine keeps its own statusline. |
| `dwt hooks-install` | Add a UserPromptSubmit hook that injects one line of live session state (`[dwt] …`) into Claude's context on every prompt. This is what powers the tracking loop below — act on those lines without being asked. |

## Tracking Claude Code work (the default behavior)

The user wants work done *with* Claude Code tracked as their own skill
practice — supervising a coding session trains `#product-development` even
when Claude does the execution. Follow this loop:

1. **When substantive work begins**, propose a tracking line the user can
   tweak in one reply — description, `#skills`, estimate, project:

   > Track as: `Mobile redesign & product direction #product-development`
   > @2h in project "Deep Work Timer" — ok, or edit?

   On approval: `dwt add ...` then `dwt start <id>`. Never silently start
   sessions.
2. **While working**, the statusline shows the live session. When the
   conversation's focus shifts to a different effort, suggest `dwt switch`
   to an existing task or propose a new one (same one-line format).
3. **When the session goes overdue or work wraps**, run `dwt log`, report
   what was logged, and offer to start the next one if work continues.
4. Prefer the CLI over raw API calls — including custom session lengths
   (`dwt start <id> --minutes <n>`).

## Multiple Claude Code windows (the arbiter)

The account still has ONE session — that models the user's singular
attention, and totals must stay wall-clock true (the server rejects
overlapping pomodoros). When the user runs several Claude Code windows at
once, the `dwt-context` hook arbitrates: each prompt is treated as the live
signal of where the user's attention is. If the prompting window's cwd is
mapped (`dwt map <task_id> <path>`) and the active segment points at a
different task, the hook switches the segment to this window's task
(60s debounce, lock shared across windows). Sequential segments then split
each pomodoro honestly across every workstream the user actually steered.

What Claude should do:

- On first substantive work in a project, check `dwt map` — if the workdir
  is unmapped, propose mapping it to the tracking task being created
  (`dwt map <task_id>`). One mapping per project dir; subdirs inherit
  unless mapped more specifically.
- When the hook line says the arbiter switched or debounced, just continue —
  no action needed; that is the system working.
- Per-task times become fractional shares of real time. This is by design:
  supervising three agents for 25 minutes is 25 minutes of attention,
  split — never 75 minutes logged.

## Config

`~/.deepworktimer/config` — `KEY=VALUE` lines. Only key: `BASE_URL`
(default `https://deepworktimer.com`). Point it at `http://localhost` to use a
local dev stack. `DWT_CONFIG_DIR` env var relocates the whole config dir
(useful for testing without touching real credentials). Statusline cache:
`~/.deepworktimer/statusline-cache`.

## Onboarding (first use / 401 errors)

The user (stirredo@gmail.com) signs in to deepworktimer.com via GitHub OAuth.
If `dwt login` fails for an OAuth-only account (no password):

- **Path A — set a password:** use the password-reset flow on
  https://deepworktimer.com (Forgot password → email link → set password), then
  `dwt login <email>`.
- **Path B — paste a token:** create one in Settings → API Tokens on the web
  app and run `dwt token <raw-token>`.

After auth: `dwt whoami` to confirm, then offer `dwt statusline-install`.

## API quirks worth knowing

- Task creation field is `task_description`, NOT `description` (updates use `description`).
- `estimated_time` is denominated in **minutes** API-side; the CLI's `--estimate`
  takes seconds and converts. POST /api/tasks ignores `estimated_time`, so the
  CLI applies it with a follow-up PATCH.
- POST /api/pomodoro enforces two rules: the sum of `effort_time` must not
  exceed `time`, and `last_pomodoro.created_at + time` must be in the past —
  i.e. you cannot log 25 minutes twice within the same real 25-minute window
  ("Invalid pomodoro time given"). If a log fails with that message, wait or
  log a shorter duration. A 429 means rate-limited: retry later, never drop.
- All protected routes require a verified email (`verified` middleware) —
  unverified accounts get 403s even with a valid token.
- Stats "today" boundaries use the account's profile timezone.
