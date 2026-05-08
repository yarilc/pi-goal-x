# pi-goal

A pi extension that adds a thin long-running goal loop to pi. It gives you a `/goal` command, agent-visible goal tools, local markdown persistence, hook reminders, and autonomous checkpoint continuation until the agent marks the goal complete.

## Install

Install from GitHub:

```bash
pi install https://github.com/lulucatdev/pi-goal.git
```

Or, from a local checkout:

```bash
pi install .
```

To try the extension once without installing it globally:

```bash
pi -e .
```

After installing, start pi normally in any project and use `/goal` in the TUI.

## Quick Start

Start a goal and let the agent continue autonomously:

```text
/goal improve benchmark coverage for the parser
```

Check status:

```text
/goal status
```

Ask the agent to revise the goal prompt:

```text
/goal tweak focus on parser edge cases before adding broader benchmarks
```

Pause or resume autonomous work:

```text
Press Esc while the agent is running
/goal resume
```

Replace or clear the current goal:

```text
/goal replace migrate the auth module
/goal clear
```

There is intentionally no `/goal complete` command. The user controls create, resume, replace, and clear; pressing Esc pauses the active goal. The agent marks the goal complete only by calling `update_goal` when the objective is actually done.

## Command Reference

### `/goal <objective>`

Creates a new active goal. If an unfinished goal already exists, pi asks for confirmation before replacing it.

Examples:

```text
/goal write tests for the payment retry flow
/goal --no-auto keep this migration goal in context but wait for my next instruction
```

### `/goal status`

Shows the active goal, status, auto-continue setting, and local file path.

### `/goal tweak <instructions>`

Sends a normal agent-visible message asking the agent to update the active goal file. The extension does not mutate the prompt directly.

Use this when the goal is directionally right but needs refinement:

```text
/goal tweak preserve the original API and only refactor internals
```

The agent is instructed to:

1. Read the active goal file.
2. Edit only the `# Goal Prompt` section.
3. Avoid marking the goal complete just because the prompt changed.
4. Continue working under the revised prompt.

### `/goal resume`

Resumes a paused goal and queues another continuation if auto-continue is enabled.

### `/goal replace <objective>`

Archives the current unfinished goal, then starts a new active goal.

### `/goal clear`

Archives the current unfinished goal and removes it from the active session state. This is user-controlled; the agent cannot clear goals.

## Flags

- `--no-auto` or `--no-start`: create the goal and keep it in context, but do not automatically send continuation prompts.
- `--auto` or `--start`: explicitly enable autonomous continuation. This is the default.

Older `--tokens`, `--token-budget`, and `--max-turns` flags are accepted for compatibility, but they are ignored. A goal runs until it is complete, the user presses Esc to pause it, or the user clears or replaces it.

## Agent Tools

The extension exposes three tools to the model:

- `get_goal`: read the current goal, status, auto-continue setting, and file paths.
- `create_goal`: create a goal only when the user explicitly asks the agent to set one.
- `update_goal`: mark the active goal `complete` when the objective is actually achieved.

`create_goal` and `update_goal` run sequentially to avoid concurrent state mutations. `update_goal` refuses stale in-flight runs if the active goal changed while the agent was working.

`get_goal` and `update_goal` are only exposed while a goal is active. When no goal exists, or the goal is paused or complete, the extension hides them so unrelated turns do not get nudged into goal bookkeeping.

## How Autonomous Continuation Works

When a goal is active and auto-continue is enabled, pi injects goal context into the system prompt. The objective is wrapped as untrusted user data so it stays task content, not extension-level control text. After the agent stops, resumes, or auto-compacts, the extension queues a hidden compact checkpoint message asking the agent to decide whether the goal is complete. If complete, the agent calls `update_goal`; otherwise it immediately takes the next concrete step.

Each checkpoint includes a goal id. If an old checkpoint survives compaction, reload, goal replacement, or repeated aborts, the context hook rewrites it into a hidden stale-checkpoint notice before it reaches the model. Only the latest checkpoint for the current active goal is allowed through. That keeps the loop simple: there are no token budgets, turn caps, or synthetic paused states, only active/paused/complete plus checkpoint reminders.

Autonomous continuation stops when:

- the agent calls `update_goal` with `status=complete`;
- the user presses Esc while the agent is running;
- the user runs `/goal clear`.

## Local Files

Active goals are written as editable markdown files under `.pi/goals/`:

```text
.pi/goals/active_goal_2026050711200332_<goal-id>.md
```

Archived goals are written under `.pi/goals/archived/`:

```text
.pi/goals/archived/goal_2026050710232343_<goal-id>.md
```

Each file starts with JSON metadata, followed by an editable prompt section:

```markdown
# Goal Prompt

The current goal prompt lives here.

## Progress

- Status: running
- Auto-continue: on
```

The extension treats lifecycle metadata as extension-owned and rereads only the `# Goal Prompt` section from disk before writing progress. This prevents `/goal tweak` edits from being overwritten by stale in-memory state while keeping status, file paths, and archive transitions controlled by the extension.

For safety, goal file paths are constrained to `.pi/goals/` and `.pi/goals/archived/`. The extension rejects absolute paths, path traversal, NUL bytes, symlinked goal paths, and metadata-provided paths outside the allowed directories.

## Recommended Workflow

1. Start with a concrete objective: `/goal migrate auth tests to the new helper API`.
2. Use `/goal status` when you want to inspect progress.
3. Use `/goal tweak ...` when you want to change direction without bypassing the agent.
4. Press Esc before manual intervention or risky operations.
5. Let the agent call `update_goal` only when the goal is actually complete.
6. Use `/goal clear` to stop tracking the current goal, or `/goal replace ...` to start a new one.

## Development

Install dependencies and type-check:

```bash
npm install
npm run check
```

Preview the package contents:

```bash
npm pack --dry-run
```

## Notes

This mirrors the main Codex design split: the user controls goal creation, resume, clear, and replacement, and Esc pauses active work; the model can only mark the current active goal complete. The extension deliberately avoids turn caps, token counters, and budget-driven status. Its job is just to keep the goal visible to the model and restart the next checkpoint when the previous turn stops.
