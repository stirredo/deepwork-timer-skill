# Auto-logging Claude Code work time to Deep Work Timer (proposal — NOT installed)

Idea: while Claude Code works on a task, keep a `dwt` session open and let hooks
flush elapsed time to deepworktimer.com automatically, so focus stats accumulate
without manual `dwt log` calls.

## Recommended design: Stop hook with a minimum-elapsed threshold

- You (or Claude) run `dwt start <task_id>` once when work on a tracked task begins.
- A `Stop` hook fires after each assistant turn. It checks the local session file
  and, only if at least N minutes have elapsed, runs `dwt log` (which posts the
  pomodoro and clears the session) and immediately re-starts the session on the
  same task. Short turns are skipped, so you get one pomodoro per ~N minutes of
  real work instead of one per turn.
- This composes well with the API's `ExtendsPreviousPomodoroRule`: elapsed wall
  time since the previous log can never overlap the previous pomodoro, so the
  flushes always validate.

### settings.json snippet (add to `~/.claude/settings.json`)

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/bin/bash -c 's=\"$HOME/.deepworktimer/session\"; dwt=\"$HOME/.claude/skills/deepwork-timer/bin/dwt\"; [ -f \"$s\" ] || exit 0; task=$(sed -n \"s/^TASK_ID=//p\" \"$s\"); start=$(sed -n \"s/^STARTED_AT=//p\" \"$s\"); now=$(date -u +%s); begin=$(date -j -u -f \"%Y-%m-%dT%H:%M:%SZ\" \"$start\" +%s 2>/dev/null || echo \"$now\"); el=$((now-begin)); [ \"$el\" -ge 600 ] && { \"$dwt\" log >/dev/null 2>&1 && \"$dwt\" start \"$task\" >/dev/null 2>&1; }; exit 0'"
          }
        ]
      }
    ]
  }
}
```

Notes on the snippet:
- `600` = flush threshold in seconds (10 min). Raise to 1500 for strict
  25-minute pomodoros.
- It exits 0 unconditionally so a network failure never blocks Claude's turn.
- `date -j -u -f` is the macOS form; on Linux use `date -u -d "$start" +%s`.

## Alternative: SessionEnd hook (one pomodoro per Claude Code session)

If per-turn flushing feels noisy, log once when the Claude Code session ends:

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/bin/bash -c '[ -f \"$HOME/.deepworktimer/session\" ] && \"$HOME/.claude/skills/deepwork-timer/bin/dwt\" log >/dev/null 2>&1; exit 0'"
          }
        ]
      }
    ]
  }
}
```

Trade-off: a crash or long-idle session inflates the logged time (elapsed wall
time, not active time). The Stop-hook variant caps each flush at the threshold
interval, which tracks reality better.

## Why not auto-start?

Starting a session automatically (e.g. on `UserPromptSubmit`) would need a way
to map the conversation to a Deep Work Timer task id. Better to keep `dwt start`
explicit: when the user says "track this", Claude picks/creates the task via
`dwt add` / `dwt tasks` and runs `dwt start <id>`.
