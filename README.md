# deepwork-timer — Claude Code skill

Lets Claude Code track tasks and pomodoro work sessions on
[deepworktimer.com](https://deepworktimer.com) via the `dwt` CLI
(login, tasks, sessions, time logging, stats).

## Install on a new machine

```bash
git clone git@github.com:stirredo/deepwork-timer-skill.git ~/.claude/skills/deepwork-timer
chmod +x ~/.claude/skills/deepwork-timer/bin/dwt
```

Then authenticate once:

```bash
~/.claude/skills/deepwork-timer/bin/dwt login <email>
# or paste an existing Sanctum token:
~/.claude/skills/deepwork-timer/bin/dwt token <raw-token>
```

Credentials live in `~/.deepworktimer/` (never in this repo).
Claude Code picks the skill up automatically from `~/.claude/skills/`.

See `SKILL.md` for the full command reference and `HOOKS.md` for the
auto-logging hook proposal.

## Update

```bash
git -C ~/.claude/skills/deepwork-timer pull
```
