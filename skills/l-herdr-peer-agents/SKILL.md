---
name: l-herdr-peer-agents
description: Delegate bounded coding tasks to Pi or Antigravity CLI (agy) peer agents running in sibling Herdr panes. Use for independent repository research, exploration, verification, review, or implementation when a separate agent context is useful. Requires the current agent to run inside Herdr.
---

# Herdr Peer Agents

Use Herdr as a lightweight peer-agent runtime instead of an in-process subagent framework.

This skill supports exactly two peer kinds:

- `agy` — default for read-only research, repository exploration, review, and verification.
- `pi` — default for implementation, fixes, tests, and other tasks that may modify the working tree.

Do not launch Claude, Codex, Gemini, or any other Herdr agent kind from this skill.

## Preconditions

Before controlling Herdr, verify that the current agent is running inside a Herdr-managed pane:

```bash
test "${HERDR_ENV:-}" = 1
```

If the check fails, stop and tell the user that this agent is not running inside Herdr. Do not try to control a focused Herdr session from outside it.

Use the existing workspace, current tab, and current working directory by default. Do not create a new workspace, tab, worktree, or alternate cwd unless the user explicitly asks for that topology.

## Routing

Honor an explicitly requested peer kind first. Otherwise choose by task shape.

### Use `agy` for

- repository research and codebase exploration
- locating implementations or tracing behavior
- independent verification of a conclusion
- reviewing a diff or proposed change
- comparing approaches or checking assumptions
- bounded read-only investigation that can run in parallel

AGY tasks are read-only by default. Include `Do not modify files.` in the prompt unless the user explicitly asks for writes; if writes are required, prefer `pi` instead.

### Use `pi` for

- implementing a bounded change
- fixing a bug
- writing or updating tests
- running an edit-test-fix loop
- work that needs the same project tooling and filesystem as the main agent

When both the main agent and a Pi peer share the same working tree, avoid overlapping edits. Either give the peer a non-overlapping file/module scope or serialize the work. Do not create a worktree automatically.

### Do not delegate when

- the task is trivial and cheaper to do directly
- the peer would need most of the main conversation to understand the task
- the task requires constant back-and-forth coordination
- a Pi peer would race with the main agent on the same files

## Default topology

Create a sibling pane in the current tab and preserve the caller's cwd. Keep focus on the caller pane.

Choose a sensible split direction. Prefer `right` for a wide pane and `down` when another vertical split would make panes too narrow.

```bash
herdr pane layout --pane "$HERDR_PANE_ID"
herdr pane split --current --direction right --cwd "$PWD" --no-focus
```

Use `down` instead of `right` when appropriate.

Read the new pane ID from `.result.pane.pane_id` in the JSON response. Do not guess pane IDs.

## Start a peer

Use a short, descriptive, unique live agent name matching `[a-z][a-z0-9_-]{0,31}`.

For AGY:

```bash
herdr agent start research --kind agy --pane <pane-id>
```

For Pi:

```bash
herdr agent start implementer --kind pi --pane <pane-id>
```

If the preferred name is already live, choose a short suffixed variant such as `research2` or `implement2`.

Do not pass model or provider arguments unless the user explicitly asks for them. Arguments intended for the underlying CLI go after `--`.

Reuse an existing idle peer when it already has the right role and task context instead of spawning duplicates.

## Prompt contract

Send a self-contained task. The peer should not need access to the main conversation to understand what to do.

Include:

1. the concrete goal
2. relevant paths, symbols, errors, or diff scope
3. whether file modifications are allowed
4. constraints and non-goals
5. the expected response format

For AGY, prefer prompts like:

```text
Investigate why <behavior> happens in this repository.
Do not modify files.
Trace the relevant code paths and return concise findings with file paths, symbols, and any uncertainty.
```

For Pi, prefer prompts like:

```text
Implement <bounded change> in the current repository.
Limit changes to <scope> and avoid unrelated refactors.
Run the relevant tests or checks.
Return a concise summary of files changed, checks run, and any remaining concerns.
```

## Submit, wait, and read

Use the agent control surface rather than raw pane input:

```bash
herdr agent prompt <name> "<prompt>" --wait --timeout 300000
herdr agent read <name> --source recent-unwrapped --lines 160
```

`agent prompt --wait` may settle at `idle`, `done`, or `blocked`.

- `done` or `idle`: read and integrate the result.
- `blocked`: read the peer output to understand what input or approval is required. Surface material questions to the user instead of guessing.
- `unknown`: do not assume completion. Read current output and wait again or inspect the agent state.

Keep responses concise so they remain readable from Herdr pane history.

If a read is truncated because the agent uses alternate-screen rendering, first ask the peer for a shorter summary. Only after a failed read, ask it to write the complete response to a temporary Markdown file and reply with that path, then read the file directly.

## Parallelism

Read-only AGY peers are safe to use for independent tasks in parallel.

Examples:

- one AGY peer traces authentication flow while the main agent inspects UI behavior
- two AGY peers independently verify different hypotheses
- an AGY peer reviews a change after the main agent implements it

Pi peers that modify the same working tree require stricter coordination. Do not run concurrent overlapping edits.

## Result handling

Treat peer output as evidence, not authority.

After reading a peer result:

1. extract the useful findings or changed-file summary
2. verify important claims against the repository when needed
3. integrate the result into the main task
4. mention meaningful disagreement or uncertainty

Do not blindly forward the raw peer transcript to the user.

## Lifecycle

Keep the sibling pane and peer alive by default so follow-up prompts can reuse its context. Do not steal focus from the user's current pane.

Only terminate, replace, or reorganize peers when the user asks, when the peer is unusable, or when cleanup is clearly required for the current task.
