# dwt × Claude Code automation (installed via `dwt hooks-install` / `dwt statusline-install`)

Two integrations keep Deep Work Timer ambient inside Claude Code:

## 1. Context hook (UserPromptSubmit) — awareness

`bin/dwt-context` injects one line into Claude's context on every prompt:

- `[dwt] no active session — …propose a tracking task`
- `[dwt] active session: '<task>' — 12:33 left …suggest dwt switch if focus moved`
- `[dwt] session OVERDUE by 47m on '<task>' — propose dwt log or dwt abandon`

Claude acts on these conversationally (see SKILL.md "Tracking Claude Code
work"): it proposes, the user approves — nothing is logged or switched
silently. Cache-backed (60s, shared with the statusline) so prompts are
never blocked.

## 2. Statusline segment — visibility

`bin/dwt-statusline-wrap` composes a `🍅 <task> · mm:ss left` segment with
whatever statusline the machine already uses.

## Why no auto-logging Stop hook

An earlier proposal auto-ran `dwt log` from a Stop hook. With server-side
sessions that is unsafe: sessions are shared across devices, and a hook on
one machine would complete a session the user is running from their phone.
Awareness + explicit suggestion (above) replaces it.
