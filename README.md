# Ralph Loop (`agy-ralph-loop`)

Minimal Ralph Wiggum loop plugin for Google Antigravity (App, CLI, and IDE). Runs a prompt `N` times sequentially, starting every iteration in a brand-new, cleared context window while sharing workspace files on disk.

## Usage

```text
/loop <count> <prompt>
```

### Examples

```text
/loop 5 Run the test suite, fix one failing test, and commit the fix
```

```text
/loop 3 Pick the next unchecked task in TODO.md, implement and verify it, mark it done, and commit
```

## How It Works

1. `/loop <count> <prompt>` parses `<count>` (integer `>= 1`), `<prompt>`, and the main thread's `Conversation ID`.
2. Each pass spawns a fresh `self` subagent (`Workspace: "inherit"`, `Model: "inherit"`) one at a time, waiting for iteration `k` to send `[LOOP_COMPLETE]` (after all edits, background tasks, and verifications settle) before starting `k + 1`.
3. **Live main-thread telemetry:** While an iteration works, it sends concise `[LOOP_UPDATE]` messages back to the main thread via `send_message` when selecting its target, working through test/build snags or pivots, and running verifications—plus `[LOOP_WAITING]` guards when a long command backgrounds and `[LOOP_COMPLETE]` when the pass settles—so the main thread streams real-time progress and hands off reliably between iterations.
4. Context is cleared between passes: every iteration runs in a new conversation with zero prior chat history and receives the exact original `<prompt>`. State persists strictly on disk and in git.

## Validate & Install

```bash
# Validate strictly
agy plugin validate --strict ~/projects/development_tools/agy-ralph-loop

# Install globally (~/.gemini/config/plugins/agy-ralph-loop)
agy plugin install ~/projects/development_tools/agy-ralph-loop

# Verify installation
agy plugin list
```
