# skills

Personal agent skills.

## l-herdr-peer-agents

Use Herdr as a lightweight peer-agent runtime with only two supported peers:

- `agy` — read-only research, repository exploration, review, and verification
- `pi` — bounded implementation, fixes, tests, and other tasks that may modify the working tree

The skill creates sibling Herdr panes, starts a peer with `herdr agent start`, submits work with `herdr agent prompt --wait`, and reads the result back into the main agent context.

### Install

```bash
npx skills add liaoyinglong/skills --skill l-herdr-peer-agents -g
```

Or install it only for the current project by omitting `-g`.

### Requirements

- Herdr installed and running
- the main coding agent itself is running inside Herdr (`HERDR_ENV=1`)
- Pi and/or Antigravity CLI available for Herdr to launch

The skill intentionally does not launch Claude, Codex, Gemini, or other Herdr agent kinds.
