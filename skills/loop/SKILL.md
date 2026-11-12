---
name: loop
description: >-
  Run a Ralph Wiggum loop via /loop <count> <prompt>, executing <prompt>
  sequentially <count> times with a fresh, cleared context window on every
  iteration and streaming milestone and problem-solving updates back to the
  main thread.
---

# Ralph Loop (`/loop`)

Execute `<prompt>` sequentially `<count>` times, giving every iteration a brand-new, cleared context window while sharing the active workspace on disk and relaying live milestone + problem-solving updates to the main thread.

**Recursion guard:** If your prompt contains `[Ralph Loop iteration — fresh context]`, you are inside an active iteration—execute the task directly following the telemetry and turn-boundary contract in your prompt without invoking `/loop` or spawning loop iterations.

## Steps

1. **Parse `<count>`, `<prompt>`, and `<parentConversationId>`**
   - Read the arguments following `/loop`:
     - `<count>`: first whitespace-delimited token; must be an integer `>= 1`.
     - `<prompt>`: all remaining text after `<count>` (preserving multiline content; unwrap a single matching outer pair of `"` or `'` quotes only if it wraps the entire prompt).
   - Read `<parentConversationId>` from your own `Conversation ID` in `<user_information>`.
   - If `<count>` is missing, not an integer `>= 1`, or `<prompt>` is empty, reply with:
     `Usage: /loop <count> <prompt> (example: /loop 5 Fix the next failing test and commit)`
     and stop immediately without calling tools.

2. **Build the shared iteration prompt and launch iteration `1`**
   - Construct the exact `Prompt` below (substituting `<prompt>` and `<parentConversationId>` only—keep all chat history and prior-iteration summaries out of it):
     ```text
     <prompt>

     ---
     [Ralph Loop iteration — fresh context]
     Parent Conversation ID: <parentConversationId>

     Complete this single pass autonomously end-to-end in the workspace without asking follow-up questions.

     Live main-thread telemetry (`send_message` to `<parentConversationId>`):
     Calling `send_message` in a turn suppresses Antigravity's automatic idle notice for that turn, so **all** progress, wait, and completion signals must be sent explicitly via `send_message(Recipient="<parentConversationId>", Message="...")`:
     - **Target (`[LOOP_UPDATE]`):** right after initial inspection, send `[LOOP_UPDATE] Target: ...` stating what concrete item or problem this pass is tackling.
     - **Problem-solving (`[LOOP_UPDATE]`):** send `[LOOP_UPDATE] Problem-solving: ...` once you diagnose the root cause / plan your edit, and again whenever a test, build, or check fails, an assumption breaks, or you hit a snag and pivot.
     - **Verification (`[LOOP_UPDATE]`):** send `[LOOP_UPDATE] Verification: ...` when running your verification suite (and if a check fails). Fold the final passing check + commit result into `[LOOP_COMPLETE]` rather than sending a separate `[LOOP_UPDATE]` right before finishing.
     - **Background wait (`[LOOP_WAITING]`):** if a `run_command` call backgrounds (`Tool is running as a background task`) and you must end a turn to wait for it, send `[LOOP_WAITING] <command>` via `send_message` and end your turn.
     - **Completion (`[LOOP_COMPLETE]` — mandatory final tool call):** only after all edits, background tasks, verifications, and commits for this pass have settled, call `send_message(Recipient="<parentConversationId>", Message="[LOOP_COMPLETE] <1-3 line summary of changes, verification status, and any remaining blockers>")`, then immediately end your turn with `[LOOP_COMPLETE]` and no further tool calls.
     ```
   - Call `invoke_subagent` **once**:
     - `TypeName`: `"self"`
     - `Role`: `"Ralph Loop 1/<count>"`
     - `Workspace`: `"inherit"`
     - `Model`: `"inherit"`
     - `Prompt`: the constructed prompt above.
   - Output `**Loop 1/<count>** started ([subagent](conversation://<subagentId>)) — inspecting workspace...` and **end your turn immediately** (call no further tools).

3. **Relay live updates and gate sequential handoffs (`k = 1 .. <count>`)**
   - Ignore any late message or idle notice referencing an already-completed subagent ID (`!= <activeSubagentId>`).
   - When resumed by incoming messages for active iteration `k` (`<activeSubagentId>`), if multiple messages arrived together, output any `[LOOP_UPDATE]` lines in order first, then apply `[LOOP_WAITING]` or `[LOOP_COMPLETE]` in the same turn:
   - **Live milestone or problem-solving update** (`sender=<activeSubagentId>` containing `[LOOP_UPDATE]`, with no `[LOOP_COMPLETE]` in the wakeup):
     - Strip `[LOOP_UPDATE]` and output directly on the main thread:
       `**[Loop <k>/<count>]** <update>`
     - **End your turn immediately** without calling tools or spawning the next iteration (iteration `k` is still running).
   - **Background-command wait** (`[LOOP_WAITING]` from `<activeSubagentId>`, or `Subagent <activeSubagentId> has gone idle` whose response starts with `[LOOP_WAITING]`):
     - Strip `[LOOP_WAITING]` and output `**[Loop <k>/<count>]** Waiting on background command: <detail>`, then **end your turn immediately** without advancing `k` (Antigravity wakes `<activeSubagentId>` automatically when its background task finishes).
   - **Iteration `k` complete** (`sender=<activeSubagentId>` containing `[LOOP_COMPLETE]`, or fallback `Subagent <activeSubagentId> has gone idle` without `[LOOP_WAITING]`):
     - Strip `[LOOP_COMPLETE]` to get `<summary>` and mark `<activeSubagentId>` completed.
     - If `k < <count>`:
       - Spawn iteration `k + 1` via a **new** `invoke_subagent` call (`TypeName: "self"`, `Role: "Ralph Loop <k+1>/<count>"`, `Workspace: "inherit"`, `Model: "inherit"`) using the **exact same `Prompt`** constructed in Step 2. Never reuse a prior `conversationId` via `send_message`, and keep prior iterations' chat history out of `Prompt`; state transfers strictly through workspace files and git.
       - Continue through all `<count>` iterations even when a pass reports test failures or partial progress; abort early only on an unrecoverable platform/quota error or explicit user cancellation.
       - Output:
         `**Loop <k>/<count> complete** ([transcript](conversation://<activeSubagentId>)): <summary>`
         `**Loop <k+1>/<count>** started ([subagent](conversation://<nextSubagentId>)) — inspecting workspace...`
       - **End your turn immediately**.
     - If `k == <count>`, proceed to Step 4.

4. **Report completion (`k == <count>`)**
   - Output the final pass completion line followed by a compact summary listing `[Loop 1/<count>](conversation://<id>)` through `[Loop <count>/<count>](conversation://<id>)` with each iteration's outcome and verification status.
