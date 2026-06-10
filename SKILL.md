---
name: deepwork-timer
description: track tasks and pomodoro work sessions on deepworktimer.com — use when the user asks to track work time, start/log a pomodoro, manage their Deep Work Timer tasks, or review their focus stats
---

# Deep Work Timer

Manage the user's tasks and log focused work time on their Deep Work Timer account
(Laravel API at https://deepworktimer.com) via the bundled CLI:

```
~/.claude/skills/deepwork-timer/bin/dwt
```

Run it with its absolute path (it is not on PATH). All commands print friendly
errors; a 401 means the token is missing/expired — re-run onboarding below.

## Commands

| Command | What it does |
|---|---|
| `dwt login <email> [-p <pw>]` | POST /api/login, saves token to `~/.deepworktimer/token` (600). Password is prompted, read from stdin, or passed with `-p`. |
| `dwt token <raw-token>` | Save a pasted Sanctum API token directly (also verifies it). |
| `dwt whoami` | Show authenticated user (name, email, timezone, pomodoro settings). |
| `dwt projects` | List projects (id, name). |
| `dwt tasks [--project <id>]` | List tasks. Without `--project` it lists tasks not assigned to any project. Subtasks are indented. |
| `dwt add "<desc>" [--project <id>] [--estimate <seconds>]` | Create a task. |
| `dwt done <task_id>` | Mark a task completed. |
| `dwt start <task_id>` | Start a local work session (state in `~/.deepworktimer/session`; nothing is sent to the API yet). |
| `dwt log [--task <id>] [--minutes <n>]` | Log a pomodoro. With no args it closes the `dwt start` session using elapsed wall time on that task. `--minutes` overrides the duration; `--task` overrides the task. |
| `dwt status` | Active session (task + elapsed) and today's total focus time. |
| `dwt stats` | Summary: today / last 7 / 30 / 365 days / all time, best day, daily average. |

Typical flow when the user starts working on something:
1. `dwt tasks` (or `dwt projects` then `dwt tasks --project N`) to find the task id — or `dwt add "..."` to create one.
2. `dwt start <id>` when work begins.
3. `dwt log` when work ends (or `dwt log --minutes 25` to record a fixed pomodoro).
4. `dwt done <id>` when the task is finished.
5. `dwt stats` to review focus time.

## Config

`~/.deepworktimer/config` — `KEY=VALUE` lines. Only key: `BASE_URL`
(default `https://deepworktimer.com`). Point it at `http://localhost` to use a
local dev stack. `DWT_CONFIG_DIR` env var relocates the whole config dir
(useful for testing without touching real credentials).

## Onboarding (first use / 401 errors)

The user (stirredo@gmail.com) signs in to deepworktimer.com via GitHub OAuth and
has NO password set, so `dwt login` will not work until one of these is done:

- **Path A — set a password:** use the password-reset flow on
  https://deepworktimer.com (Forgot password → email link → set password), then
  `dwt login stirredo@gmail.com`.
- **Path B — paste a token:** obtain a Sanctum bearer token (e.g. from the
  browser session / API) and run `dwt token <raw-token>`.

## API quirks worth knowing

- Task creation field is `task_description`, NOT `description` (updates use `description`).
- `estimated_time` is denominated in **minutes** API-side; the CLI's `--estimate`
  takes seconds and converts. POST /api/tasks ignores `estimated_time`, so the
  CLI applies it with a follow-up PATCH.
- POST /api/pomodoro enforces two rules: the sum of `effort_time` must not
  exceed `time`, and `last_pomodoro.created_at + time` must be in the past —
  i.e. you cannot log 25 minutes twice within the same real 25-minute window
  ("Invalid pomodoro time given"). If a log fails with that message, wait or
  log a shorter duration.
- All protected routes require a verified email (`verified` middleware) —
  unverified accounts get 403s even with a valid token.
- Stats "today" boundaries use the account's timezone (America/Toronto).
