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

1. `/loop <count> <prompt>` parses `<count>` (integer `>= 1`) and `<prompt>`.
2. Each pass spawns a fresh `self` subagent (`Workspace: "inherit"`, `Model: "inherit"`) one at a time, waiting for iteration `k` to finish before starting `k + 1`.
3. Context is cleared between passes: every iteration runs in a new conversation with zero prior chat history and receives the exact original `<prompt>`. State persists purely on disk and in git.

## Validate & Install

```bash
# Validate strictly
agy plugin validate --strict ~/projects/development_tools/agy-ralph-loop

# Install globally (~/.gemini/config/plugins/agy-ralph-loop)
agy plugin install ~/projects/development_tools/agy-ralph-loop

# Verify installation
agy plugin list
```
