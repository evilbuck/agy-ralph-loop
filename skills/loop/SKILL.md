---
name: loop
description: >-
  Run a Ralph Wiggum loop via /loop <count> <prompt>, executing <prompt>
  sequentially <count> times with a fresh, cleared context window on every
  iteration.
---

# Ralph Loop (`/loop`)

Execute `<prompt>` sequentially `<count>` times, giving every iteration a brand-new, cleared context window while sharing the active workspace on disk.

**Recursion guard:** If your prompt already contains `[Ralph Loop iteration — fresh context]`, you are inside an active iteration—execute the task directly and never invoke `/loop` or spawn further loop iterations.

## Steps

1. **Parse `<count>` and `<prompt>`**
   - Read the arguments following `/loop`:
     - `<count>`: first whitespace-delimited token; must be an integer `>= 1`.
     - `<prompt>`: all remaining text after `<count>` (preserving multiline content; unwrap a single matching outer pair of `"` or `'` quotes only if it wraps the entire prompt).
   - If `<count>` is missing, not an integer `>= 1`, or `<prompt>` is empty, reply with:
     `Usage: /loop <count> <prompt> (example: /loop 5 Fix the next failing test and commit)`
     and stop immediately without calling tools.

2. **Launch iteration `1` in a fresh context and end your turn**
   - Call `invoke_subagent` **once**:
     - `TypeName`: `"self"`
     - `Role`: `"Ralph Loop 1/<count>"`
     - `Workspace`: `"inherit"`
     - `Model`: `"inherit"`
     - `Prompt`:
       ```text
       <prompt>

       ---
       [Ralph Loop iteration — fresh context] Complete this single pass autonomously end-to-end in the workspace without asking follow-up questions. When done, reply with a 1-3 line summary of changes and verification status.
       ```
   - Output `Running loop 1/<count>...` and **end your turn immediately** (call no further tools; Antigravity resumes you automatically when the subagent finishes).

3. **Advance sequentially with cleared context (`k = 2 .. <count>`)**
   - On wakeup from iteration `k - 1`:
     - Record `[Loop <k-1>/<count>](conversation://<conversationId>)` and its brief outcome.
     - Spawn iteration `k` with a **new** `invoke_subagent` call (`TypeName: "self"`, `Role: "Ralph Loop <k>/<count>"`, `Workspace: "inherit"`, `Model: "inherit"`) using the **exact same `Prompt`** constructed in Step 2.
     - Keep prior iterations' chat history, summaries, and reasoning out of the new subagent's `Prompt` and never reuse a previous `conversationId` via `send_message`. Progress transfers strictly through workspace files and git state.
     - Continue through all `<count>` iterations even when an iteration reports test failures or partial progress; abort early only on an unrecoverable platform/quota error or user cancellation.
     - Output `Loop <k-1>/<count> complete. Running loop <k>/<count>...` and **end your turn immediately**.

4. **Report completion (`k == <count>`)**
   - When iteration `<count>` finishes, output a compact summary listing `[Loop 1/<count>](conversation://<id>)` through `[Loop <count>/<count>](conversation://<id>)` with each iteration's outcome.
