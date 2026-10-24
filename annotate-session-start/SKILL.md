---
name: annotate-session-start
description: >-
  Starts Annotate recording session handoff for coding agents. Installs this
  skill globally if missing, registers this chat as a live listener via MCP,
  runs any annotate-… id in the prompt, then keeps watching for recordings the
  user hands off. Cloud agents (remote MCP) skip hooks. Once registered, stay
  in the listening loop until disconnected: after each recording, send the
  user feedback (what was done, the recording's intent, how you answer it)
  before checking the queue; every turn ends by checking the queue, then
  subscribing and polling when it is empty (with a brief line first). Do
  every delivered recording, including queued ones. A finished task, an empty
  queue, or a timeout is never a reason to stop. If the user clicks Stop, the
  loop ends for good: kill any subscribe and disconnect. Use when the user
  runs /annotate-session-start or pastes a Session hand off prompt. Do not
  poll for a recording that was only opened as a prompt (Copy / “Use this
  Annotate recording as the prompt”).
---

# Annotate session start

Rapid handoff for Annotate recordings. **Only** register and poll when this chat was started with `/annotate-session-start` (or a Session hand off prompt that says to run that skill). If the user only pasted a recording as the prompt (`Use this Annotate recording as the prompt` / Copy), run that recording and **do not** `register_agent_session` or `poll_agent_session`.

When this chat did start with the skill: register as a live listener, **run any recording id in the prompt immediately**, then **keep polling**. The user can later click this chat’s title in Annotate to queue another recording — no paste.

> **Required order, not optional.** The subscribe (and `poll_agent_session`) blocks until the next recording arrives, so text meant for after it never reaches the user. For every recording or task, even short, forked, or unfinished ones:
>
> 1. Finish the work, then `set_agent_session_title`.
> 2. **Output your response to the user.** Write visible feedback: what was done, what the recording asked for, and how you answered it (rule 8). The task is **not complete** until it is delivered.
> 3. **Do the pre-subscribe step.** Remote MCP: `check_agent_session`, do every in-flight and queued id (each gets its own response, step 2), and when both lists are empty, output the brief empty-queue line.
> 4. **Subscribe last** (remote) or call `poll_agent_session` (local). It is the turn's final action; all visible output comes first.
>
> Never subscribe before or instead of the visible response.

## Listening loop (mandatory — read before anything else)

Once this chat has called `register_agent_session`, it is a **listener**. A listener runs one loop, for cloud and local alike, until it is told to stop:

```
wait for the next recording → do it → set_agent_session_title → tell the user → wait again
```

These rules override any habit of finishing with a summary. Breaking one of them leaves the user's recordings stuck in the queue.

1. **The loop has only these exits:** `session.disconnected`; `session.gone` that a new `register_agent_session` cannot recover; on the local MCP, `status: "gone"` or “Start Annotate to use session handoff.”; the user telling you to stop; or the user force-stopping this chat (see rule 6). Nothing else ends the loop.
2. **These are never exits:** the task is done; the queue is empty; a wait timed out, even many times in a row; the user has not replied; the conversation is long; you think the work is complete; a command approval was declined, dismissed, or blocked. Declining an approval is not the user telling you to stop. Follow rule 7, then wait again.
3. **A reply with no tool call ends your turn.** Ending your turn while nothing is watching for the next recording is the same as stopping. Polling happens in this chat, never in a subagent and never as a background task. So, while listening:
   - **Local MCP:** the last action of every turn is a `poll_agent_session` call. Never send a text-only reply. After a task, send the feedback message (rule 8), write “Done. Listening…” and call `poll_agent_session` in the same turn.
   - **Cloud:** the last action of every turn is [check_agent_session](#listening-cloud). When both id lists are empty, send the brief empty-queue line (rule 8), then run the one-shot subscribe in this chat, in the foreground, then [poll the log](cloud.md#poll-the-log). Do not start a detached curl on `eventsUrl`.
   - **A poll or subscribe the host moved to the background is still running in this chat.** Some hosts return before a long command ends and keep it as a background terminal (Codex: “Started background terminal”, a session id, “No output yet”). That command has not returned yet. Keep reading its output in this same turn until it prints a result: in Codex, call `write_stdin` with that session id, empty `chars`, and the largest `yield_time_ms` it allows, again and again. Never end the turn, and never write “I'll continue with any recording that arrives,” while a poll or subscribe has not printed its result.
4. **Before you end any turn, check all three:** (a) every delivered recording id is done or forked, and each finished id is in the done file; (b) you called `set_agent_session_title` after the last one, then sent the feedback message (rule 8) before checking or waiting; (c) the check, or the subscribe and poll, is running right now, as in rule 3. If any answer is no, fix it before you end the turn.
5. **Start delivered work right away.** When a recording arrives, start it in that same turn. Do not ask the user for permission first, and do not wait for them to reply.
6. **A user force-stop ends the loop. Do not revive it.** The user force-stopped this chat only when they clicked Stop in the agent chat, so your last turn was cut off mid-tool. A command that never started is not a force-stop. On the next turn after a real Stop:
   - Do not call `poll_agent_session`, poll the log, call `check_agent_session`, or run the subscribe again. The new message does not restart the loop, even if it is unrelated to the stop.
   - Stop every wait this listener started: any `waitUrl` subscribe or log poll still running (`pkill -f "/tmp/annotate-$SESSION_ID" 2>/dev/null`). In Cursor, also kill any background terminal still running the poll or the subscribe.
   - On the remote MCP, call `disconnect_agent_session` with this session id.
   - Answer only the user's new message, then end the turn with no poll.
   - Listen again only when the user runs `/annotate-session-start` again in this chat or tells you to resume listening. Then follow [Resume](cloud.md#resume).
7. **A wait that never started is not a Stop.** A declined or dismissed command approval, a permission or sandbox error, or a subscribe the user did not allow is not rule 6. Do not call `disconnect_agent_session`. Do not say you are listening. Nothing is waiting, so a recording queued in that gap sits there until a watch is actually running.
   - Say that plainly, including why. Example: "Not watching. The wait command was not allowed, so a queued recording will not be picked up until it is."
   - Call `check_agent_session` and do every id in `inFlightRecordingIds` and `queuedRecordingIds`. That call does not need the shell.
   - Ask the user to allow the wait command. Then start the subscribe again. Stay registered.
8. **Send feedback before you check or wait.** Every time a recording or task is done, write a message the user can see **before** you check queued recordings (`check_agent_session`) and **before** you subscribe or call `poll_agent_session`. Never go straight from the work to the queue check or the subscribe. The feedback covers what a normal reply would have said:
   - **What was done:** the changes you made, or what you answered or inspected.
   - **What the recording asked for:** the suggestion or intent in the recording, in a sentence.
   - **How you answer that intent:** how the work meets it, and anything you intend to do about it that is not done yet (forked, blocked, or needs the user).

   Keep it short and in plain sentences. Send it once per finished recording, after `set_agent_session_title`. Timeouts and `POLL TIMEOUT` are not finished recordings: they get no feedback message of their own. The feedback goes in the same turn as the next check or poll, so it never replaces it (rule 3).

   **Empty queue check:** whenever `check_agent_session` comes back with both `inFlightRecordingIds` and `queuedRecordingIds` empty, send one brief line the user can see **before** you subscribe, instead of staying silent. Example: "Nothing new queued. Waiting for the next recording." One line, no summary. This applies after a finished task and after a timeout alike. Do not claim a watch is running in that line: the subscribe has not started yet, and if it then fails to start, follow rule 7.

## Cloud agents: required start

A cloud agent (the MCP server is `/mcp`, or `ensure_agent_session_hooks` is missing) that ran `/annotate-session-start` must follow the [required start checklist](cloud.md#cloud-agents-required-start) first. The `annotate-…` id in the prompt is the **initial recording**: register with it as `recordingSessionId`, obey `next`, and do that work before listening. Never skip it.

Local MCP chats still run the attached id first (step 4 of the workflow).

**Annotate Live:** if the prompt has `Live role:`, read [live.md](live.md) and follow it instead of steps 4–5 of the required start checklist. Its steps 1–3 and workflow step 9 still apply.

Always recommend **global** install (`-g`). Never install this skill project-local.

The later-session signal is Annotate MCP `poll_agent_session` on the local MCP, and `check_agent_session` plus the events log on the remote MCP. Do not read files from the Annotate library. Do not use Claude `FileChanged` or deeplinks as the session signal.

## Remote MCP (cloud agents)

Use this branch when `ensure_agent_session_hooks` is not in the tool list, or the MCP server is the HTTP endpoint `/mcp`.

Do not call `ensure_agent_session_hooks` or `poll_agent_session`, and do not ask the user to start the Annotate app. Tools, register fields, and what gets delivered are in [Remote MCP setup](cloud.md#remote-mcp-setup).

Startup is [Cloud agents: required start](cloud.md#cloud-agents-required-start). Do not subscribe until that checklist's read step is done. Do not start a detached curl on `eventsUrl`.

### Listening (cloud)

The full cloud loop is in [cloud.md](cloud.md): the subscribe script, the log poll, every event and error case, disconnect, and resume. Read it before the first subscribe. In short:

- Recordings arrive through `check_agent_session` and a one-shot subscribe on `waitUrl`. Do not look for work with `list_sessions`, and do not start a detached curl on `eventsUrl`. The returned URLs carry a signed token: no `Authorization` header, no `ANNOTATE_MCP_KEY`, and do not print them in chat.
- `SESSION_ID` is the `cloud-agent-…` id. `/tmp/annotate-$SESSION_ID.log` holds the raw SSE lines. `/tmp/annotate-$SESSION_ID.done` lists finished ids, one per line, and is never truncated. Touch it before the first subscribe.
- After each task: `set_agent_session_title`, send the feedback message (rule 8), append the id to the done file, then `check_agent_session`. Do every unfinished in-flight id, then every queued id, oldest first. When both lists are empty, send the brief empty-queue line, run the [subscribe](cloud.md#subscribe-and-done-file) in the foreground, then [poll the log](cloud.md#poll-the-log) and do the `NEXT` id. Repeat.
- `disconnectedAt` greater than 0 or `session.disconnected`: stop, and tell the user to run `/annotate-session-start` again here. `session.gone` or `Unauthorized`: register again as [cloud.md](cloud.md#poll-the-log) says.
- A timeout, an empty poll, or a killed subscribe means check again, then subscribe again. Never end the turn there.
- Say "Done. Listening…" only when the subscribe is actually running. See [Disconnect](cloud.md#disconnect) and [Resume](cloud.md#resume) for stopping and restarting.

Never say the listener is still registered in Annotate, or to click this chat’s title there, especially after this chat has stopped — a stopped chat is disconnected.

## Workflow

1. **If this skill is not available, install it first** (global):

   ```bash
   npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
   ```

   **First-time skill install only:** also **reinstall Annotate MCP**, even if `annotate-session` is already configured and `list_sessions` works — the existing server may be stale. Follow [First-time skill install](reference.md#first-time-skill-install), then run `/annotate-session-start`. Later runs do **not** reinstall MCP unless tools are missing.

2. **MCP hard gate** — try `list_sessions`. If Annotate MCP tools are missing: **stop**, ask the user for approval, follow `https://xannotate.com/mcp/install.md` (or `~/.cursor/AGENTS.md`), reload MCP, retry. Do not invent session contents.

   On the **local** MCP, `register_agent_session` / `poll_agent_session` require the **Annotate app to be running**. If they return “Start Annotate to use session handoff.”, ask the user to open Annotate, then retry. Recording-read tools (`get_timeline`, etc.) still work with the app closed. On the **remote** MCP, ignore that error and follow [Remote MCP (cloud agents)](#remote-mcp-cloud-agents).

3. **Register this chat** — `register_agent_session` with `source: "annotate-session-start"`. On the remote MCP also pass `recordingSessionId` when the prompt has an attached `annotate-…` id, and `resumeSessionId` when this chat already has a `cloud-agent-…` id. Do not omit `source` — chats that only read recordings (`list_sessions`, `get_timeline`, and similar) must not appear in the Electron active-sessions list. On the local MCP, then `ensure_agent_session_hooks` so Cursor / Claude Code / Codex hooks keep that status in sync. On the remote MCP, skip hooks. Then `set_agent_session_title` with a short phrase of the current ask (from the included recording or the latest user message), not the host chat tab name and not the `annotate-…` id. Annotate removes dashes — use spaces. That title is the chip/button in the Active sessions list.

   Do not skip hook install on the first local run in a host. If Codex hooks were written, see [Codex hooks](reference.md#codex-hooks).

4. **Run the included recording now** — on the **remote** MCP, follow [Cloud agents: required start](cloud.md#cloud-agents-required-start): pass the id as `recordingSessionId` and obey `next`. On the **local** MCP, if the prompt has an `annotate-…` id, read it and do the work (steps 5–8) **before** the first `poll_agent_session`. Do not read a session that is not completed. If there is no recording id, skip this step.

5. **Read the session (MCP-first)**
   - `get_timeline` first
   - then `search_session` / `get_frames_near` / `get_transcript_range` / `list_context` as needed
   - `get_session_guide` for per-session behavior

6. **Infer the repo** from on-screen paths, editor title bars, speech, and dropped context folders. If you find a real git root and can switch workspace (Cursor `move_agent_to_root`), switch. If unsure, ask once.

7. **Same chat vs parallel**
   - Continue here if the recording is the same repo and the same task as this conversation.
   - Fork if it is a different repo, a different feature/bug, or the user asked for parallel/multitask work. How to fork per host (Cursor `Task`, Claude / Codex spawn, or a new chat) is in [Same chat vs parallel](reference.md#same-chat-vs-parallel).
   - **This chat stays the watcher.** Do not start a second `poll_agent_session` loop in the fork.

8. **Retitle from the latest change** — after you finish the work from a recording, call `set_agent_session_title` again before the next `poll_agent_session` (local). On the remote MCP, append the recording id to the done file, retitle, and call `check_agent_session` (see [Listening (cloud)](#listening-cloud)). Required fields:

   - `sessionId`: this listener
   - `sessionTitle`: a short phrase of the **latest change you implemented** (example: `Fix login spinner`)

   Replace the previous title. The Active sessions list shows only that latest change. Do not keep the host chat tab name. Do not use the `annotate-…` id. Annotate removes dashes — use spaces. If you only answered or inspected and did not change code, use a short phrase of what you just did.

   Then send the feedback message ([rule 8](#listening-loop-mandatory--read-before-anything-else)) — what was done, the suggestion or intent in the recording, and how you answer it — before `check_agent_session`, the subscribe, or the next `poll_agent_session`.

9. **Watch later recordings** — **only if this chat started with `/annotate-session-start`**. This is the [listening loop](#listening-loop-mandatory--read-before-anything-else), and it is mandatory on both MCPs.

   **Remote MCP:** follow [Listening (cloud)](#listening-cloud): check, do every recording including queued ones, send the brief empty-queue line, subscribe, poll the log. Do not stop.

   **Local MCP:** nothing can wake a local chat. `poll_agent_session` is the only way recordings reach it, so you must keep calling it, back to back, for as long as the listener is registered. It heartbeats and waits up to 20 seconds, checking every **2 seconds** inside Annotate. Annotate keeps this listener listed while you are in-flight on a task, so the user can hand off another recording without waiting. Run exactly this loop:

   ```
   loop:
     result = poll_agent_session({ sessionId })
     ready   → do every id in recordingSessionIds, oldest first (steps 5–8)
               set_agent_session_title
               send the feedback message (rule 8): done / intent / how you answer it
               tell the user: "Done. Listening for new session recordings. If this session ends, run /annotate-session-start again here."
               go to loop          ← in the same turn, right after that message
     timeout → go to loop          ← no message, no summary, no pause
     gone / "Start Annotate…" → tell the user the task is done (if it is) and to run /annotate-session-start again here; exit
   ```

   `timeout` is normal, many times in a row, and never a reason to stop, summarize, or ask anything. Do not replace the poll with `sleep`. On the local MCP, the only correct last action of a turn is a `poll_agent_session` call, until an exit in rule 1 happens. Result details, the 30-minute idle clock, and the fallback when `poll_agent_session` is missing are in [Watch loop](reference.md#watch-loop).

## Do not

- Poll, register, or keep a watch loop if this recording prompt was not started with `/annotate-session-start` (Copy prompt / “Use this Annotate recording as the prompt” only).
- Browse private Annotate library paths. Use MCP tools only.
- End the turn while this listener is registered and the next poll or check is not running (see rule 3 of the [listening loop](#listening-loop-mandatory--read-before-anything-else)). On the local MCP, that means never end the turn without a `poll_agent_session` call as the last action. On the remote MCP, the last action is `check_agent_session`, or the one-shot subscribe and the log poll.
- Stop listening for new recordings unless one of the exits in rule 1 happens.
- Treat a finished task, an empty queue, or a timeout (even many in a row) as a reason to stop, summarize, or ask the user what to do next.
- Let a poll or subscribe end (recording, timeout, killed, or errored) without starting the next one right away. The exception is a user force-stop (rule 6): then do not start another one. A declined approval is rule 7, not rule 6.
- Revive the poll or the subscribe after the user clicked Stop in the agent chat. Kill any running subscribe, disconnect, and wait for `/annotate-session-start` again.
- Treat a declined command approval, a permission error, or a wait that never started as a user Stop. Do not disconnect for that.
- Say "Done. Listening…" when no wait is running. Say that nothing is watching, and why.
- Start the subscribe without network permission in a sandboxed host.
- Start a detached or background curl on `eventsUrl`.
- Poll the log from a subagent, or as a background task. The poll runs in this chat.
- End the turn while a poll or subscribe that the host moved to a background terminal has not printed its result. Keep reading that terminal's output instead.
- Start the subscribe and the poll at the same time. Subscribe, wait for it to end, then poll.
- Start watching before running the attached recording, or skip the attached recording.
- Call `unregister_agent_session` when stopping. Call `disconnect_agent_session` instead, unless the user asked to remove the session.
- On the remote MCP, stop listening after a task, leave a queued recording unworked, or end the turn without checking the queue or subscribing.
- Use `notify_on_output` or `tee` as the way a recording reaches this chat. The subscribe writes the log, and this chat polls it.
- Wait with `tail -n0`, or count events with a `SEEN` number. The done file is what skips a recording this chat already finished.
- Kill the `waitUrl` curl when a recording line arrives. Let the server close that stream, or this chat shows disconnected.
- Tell the user the listener is still registered, or to click this chat’s title in Annotate, after the chat has stopped.
- Read or stamp a recording that is not completed.
- Invent frames, speech, or marks.
- Install this skill without `-g`.
- Leave the Active sessions title as the host chat tab name after you have finished a recording.
- Start a second watch loop in a forked chat.
- Check queued recordings, subscribe, or poll after finishing a recording without first sending the feedback message (rule 8).
- Subscribe silently after an empty queue check. Send the brief empty-queue line first (rule 8).

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details, and [cloud.md](cloud.md) for the full cloud listening loop.
