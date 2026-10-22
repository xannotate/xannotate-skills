---
name: annotate-session-start
description: >-
  Starts Annotate recording session handoff for coding agents. Installs this
  skill globally if missing, registers this chat as a live listener via MCP,
  runs any annotate-… id in the prompt, then keeps watching for recordings the
  user hands off. Cloud agents (remote MCP) skip hooks. Once registered, stay
  in the listening loop until disconnected: every turn ends by checking the
  queue with check_agent_session, then subscribing to waitUrl and polling the
  log when it is empty, and every delivered recording is done, including
  anything already queued. A finished task, an empty queue, or a timeout is
  never a reason to stop. If the user clicks Stop in the agent chat, the loop
  ends for good: do not revive polling, kill any running subscribe, and
  disconnect. Use when the user runs /annotate-session-start or pastes a
  Session hand off prompt. Do not poll for a recording that was only opened
  as a prompt (Copy / “Use this Annotate recording as the prompt”) without
  that skill.
---

# Annotate session start

Rapid handoff for Annotate recordings. **Only** register and poll when this chat was started with `/annotate-session-start` (or a Session hand off prompt that says to run that skill). If the user only pasted a recording as the prompt (`Use this Annotate recording as the prompt` / Copy), run that recording and **do not** `register_agent_session` or `poll_agent_session`.

When this chat did start with the skill: register as a live listener, **run any recording id in the prompt immediately**, then **keep polling**. The user can later click this chat’s title in Annotate to queue another recording — no paste.

## Listening loop (mandatory — read before anything else)

Once this chat has called `register_agent_session`, it is a **listener**. A listener runs one loop, for cloud and local alike, until it is told to stop:

```
wait for the next recording → do it → set_agent_session_title → tell the user → wait again
```

These rules override any habit of finishing with a summary. Breaking one of them leaves the user's recordings stuck in the queue.

1. **The loop has only these exits:** `session.disconnected`; `session.gone` that a new `register_agent_session` cannot recover; on the local MCP, `status: "gone"` or “Start Annotate to use session handoff.”; the user telling you to stop; or the user force-stopping this chat (see rule 6). Nothing else ends the loop.
2. **These are never exits:** the task is done; the queue is empty; a wait timed out, even many times in a row; the user has not replied; the conversation is long; you think the work is complete; a command approval was declined, dismissed, or blocked. Declining an approval is not the user telling you to stop. Follow rule 7, then wait again.
3. **A reply with no tool call ends your turn.** Ending your turn while nothing is watching for the next recording is the same as stopping. Polling happens in this chat, never in a subagent and never as a background task. So, while listening:
   - **Local MCP:** the last action of every turn is a `poll_agent_session` call. Never send a text-only reply. After a task, write “Done. Listening…” and call `poll_agent_session` in the same turn.
   - **Cloud:** the last action of every turn is [check_agent_session](#listening-cloud). When both id lists are empty, run the one-shot subscribe in this chat, in the foreground, then [poll the log](#poll-the-log). Do not start a detached curl on `eventsUrl`.
   - **A poll or subscribe the host moved to the background is still running in this chat.** Some hosts return before a long command ends and keep it as a background terminal (Codex: “Started background terminal”, a session id, “No output yet”). That command has not returned yet. Keep reading its output in this same turn until it prints a result: in Codex, call `write_stdin` with that session id, empty `chars`, and the largest `yield_time_ms` it allows, again and again. Never end the turn, and never write “I'll continue with any recording that arrives,” while a poll or subscribe has not printed its result.
4. **Before you end any turn, check all three:** (a) every delivered recording id is done or forked, and each finished id is in the done file; (b) you called `set_agent_session_title` after the last one; (c) the check, or the subscribe and poll, is running right now, as in rule 3. If any answer is no, fix it before you end the turn.
5. **Start delivered work right away.** When a recording arrives, start it in that same turn. Do not ask the user for permission first, and do not wait for them to reply.
6. **A user force-stop ends the loop. Do not revive it.** The user force-stopped this chat only when they clicked Stop in the agent chat, so your last turn was cut off mid-tool. A command that never started is not a force-stop. On the next turn after a real Stop:
   - Do not call `poll_agent_session`, poll the log, call `check_agent_session`, or run the subscribe again. The new message does not restart the loop, even if it is unrelated to the stop.
   - Stop every wait this listener started: any `waitUrl` subscribe or log poll still running (`pkill -f "/tmp/annotate-$SESSION_ID" 2>/dev/null`). In Cursor, also kill any background terminal still running the poll or the subscribe.
   - On the remote MCP, call `disconnect_agent_session` with this session id.
   - Answer only the user's new message, then end the turn with no poll.
   - Listen again only when the user runs `/annotate-session-start` again in this chat or tells you to resume listening. Then follow [Resume](#resume).
7. **A wait that never started is not a Stop.** A declined or dismissed command approval, a permission or sandbox error, or a subscribe the user did not allow is not rule 6. Do not call `disconnect_agent_session`. Do not say you are listening. Nothing is waiting, so a recording queued in that gap sits there until a watch is actually running.
   - Say that plainly, including why. Example: "Not watching. The wait command was not allowed, so a queued recording will not be picked up until it is."
   - Call `check_agent_session` and do every id in `inFlightRecordingIds` and `queuedRecordingIds`. That call does not need the shell.
   - Ask the user to allow the wait command. Then start the subscribe again. Stay registered.

## Cloud agents: required start

Do this first when this chat is a cloud agent (the MCP server is `/mcp`, or `ensure_agent_session_hooks` is missing) and the prompt ran `/annotate-session-start`.

1. Find the `annotate-…` id in the prompt (`Session id: annotate-…`, or right after `/annotate-session-start`). That id is the **initial recording**. Never skip it, and never treat “Then run /annotate-session-start.” as an empty start.
2. `register_agent_session` with `source: "annotate-session-start"`, `recordingSessionId` set to that id, and `resumeSessionId` set to the `cloud-agent-…` id this chat already has (omit `resumeSessionId` on a brand-new chat). If there is no recording id, omit `recordingSessionId`. If the result says `resumed: false`, this is a new session.
3. Follow the `next` array in the register result, in order. When the result includes `inFlightRecordingIds` or `queuedRecordingIds`, call `get_timeline` for each and do that work before listening. Skip an in-flight id this chat already finished.
4. When `next` says the recording is ready, call `get_timeline` with that id and do the work **before** listening. When `next` says it is still uploading, do not read it; listening delivers it as `handoff.ready`.
5. `set_agent_session_title` with a short phrase of what you did.
6. Touch the done file (`touch /tmp/annotate-$SESSION_ID.done`, where `SESSION_ID` is this chat's `cloud-agent-…` id). Then follow [Listening (cloud)](#listening-cloud). Do not start a detached curl on `eventsUrl`.
7. You are now in the [listening loop](#listening-loop-mandatory--read-before-anything-else). Do every recording `check_agent_session` or the poll returns, including anything already queued. After each one, retitle, then check again. Do not stop listening.

Local MCP chats still run the attached id first (step 4 of the workflow).

**Annotate Live:** if the prompt has `Live role:`, read [live.md](live.md) and follow it instead of steps 4–5. Steps 1–3 and 9 still apply.

Always recommend **global** install (`-g`). Never install this skill project-local.

The later-session signal is Annotate MCP `poll_agent_session` on the local MCP, and `check_agent_session` plus the events log on the remote MCP. Do not read files from the Annotate library. Do not use Claude `FileChanged` or deeplinks as the session signal.

## Remote MCP (cloud agents)

Use this branch when `ensure_agent_session_hooks` is not in the tool list, or the MCP server is the HTTP endpoint `/mcp`.

- Do not call `ensure_agent_session_hooks`. There are no editor hooks on a cloud agent.
- Do not ask the user to start the Annotate app. Listener tools do not need the desktop app, and “Start Annotate to use session handoff.” does not apply.
- Do not call `poll_agent_session`. It is not on the remote MCP.
- `register_agent_session` (with `source: "annotate-session-start"` and the attached `recordingSessionId`), `check_agent_session`, `set_agent_session_title`, `set_agent_session_status`, and `disconnect_agent_session` go through MCP. Register returns `eventsUrl`, `waitUrl`, `broadcastUrl`, `inFlightRecordingIds`, `queuedRecordingIds`, and `next`. Follow [Cloud agents: required start](#cloud-agents-required-start). `eventsUrl` is returned but this skill does not use it. `check_agent_session` is the queue read. It does not take recordings off the queue. Listening is that check, then a one-shot subscribe on `waitUrl`.
- Only cloud-synced recordings arrive. A local-only recording is never delivered. A recording that is still uploading stays queued until it is complete.
- Install this skill in the cloud agent environment (`npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y`) when it is missing. Skip the local MCP reinstall in step 1; the remote server is already configured.

Startup is [Cloud agents: required start](#cloud-agents-required-start). Do not subscribe until that checklist's read step is done. Do not start a detached curl on `eventsUrl`.

### Listening (cloud)

Recordings arrive through `check_agent_session` and one server-sent events subscribe on `waitUrl`. Do not look for new work with `list_sessions`. Do not start a detached curl on `eventsUrl`. The subscribe only writes a log. This chat polls that log itself, in the foreground, after the subscribe ends. A subagent must not poll for you.

An open subscribe means this chat is **connected**. Closing it early shows **Disconnected** in Annotate. While you are working a recording, Annotate shows **processing**. Retitling marks it **idle**. `check_agent_session` heartbeats the lease between subscribes.

`eventsUrl`, `waitUrl`, and `broadcastUrl` already carry a signed token for this listener. Do not add an `Authorization` header, and do not look for or ask the user for `ANNOTATE_MCP_KEY`. `eventsUrl` is returned but not used. Use `waitUrl` and `broadcastUrl` exactly as returned (quote them; they contain `&`). Do not print them in chat.

`SESSION_ID` is the `cloud-agent-…` id. Two files, both under `/tmp`:

- `LOG=/tmp/annotate-$SESSION_ID.log` — raw SSE lines, append-only. Truncate it only when you subscribe with a new `waitUrl` (the first start, `session.gone`, or `Unauthorized`).
- `DONE=/tmp/annotate-$SESSION_ID.done` — one finished `annotate-…` id per line. Append an id after you finish that recording. Never truncate it. This replaces any count of events you have seen, so a restart and a `redelivered` replay both skip work this chat already finished.

**You must keep listening.** Stopping after one task, or ending the turn without the check (or the subscribe and poll), leaves the queue stuck and is a bug.

The log is not the queue, because nothing is appending to it until you subscribe. After each task:

1. `set_agent_session_title` for the task you just finished. That clears it from the replay list.
2. `check_agent_session` with this chat's `sessionId`. It heartbeats the lease and does not take recordings off the queue.
3. `disconnectedAt` greater than 0 means this chat was disconnected. Stop, the same as `session.disconnected`. "Agent session is not registered." means the row is gone: treat it as `session.gone`.
4. Do every id in `inFlightRecordingIds`, then every id in `queuedRecordingIds`, that is not already in the done file. One at a time, oldest first (`get_timeline`, steps 5–8). Skip an in-flight id this chat already finished. After each one, `echo "<id>" >> "$DONE"`, `set_agent_session_title`, and call `check_agent_session` again. A recording queued while you worked shows up on that next check.
5. When both lists are empty, call `subscribe_agent_session` with this chat's `sessionId` and append the returned text to the log. `event: timeout` is not a recording and is not an empty queue: call `check_agent_session` and do any unfinished ids before calling it again. If that tool is missing or the call errors, call `check_agent_session` first and do every unfinished in-flight id, then every queued id. Only when both lists are empty, subscribe once with the curl below. Then [poll the log](#poll-the-log) right away and do the `NEXT` id. Then repeat from step 1.

Touch `$DONE` before the first subscribe (`touch /tmp/annotate-$SESSION_ID.done`).

The subscribe writes every line to the log and ends only after a new recording (or `session.gone`, `session.disconnected`, or `Unauthorized`). `: open`, `: keepalive`, and `event: timeout` are not recordings, so they do not end it. Set `timeoutSec=540` on `waitUrl` (replace `1800`) so the server closes with `event: timeout` inside the 9-minute budget. Let curl run until the server closes. Do not exit on the data line and do not kill curl: `waitUrl` closes itself after one event, and hanging up early shows this chat as disconnected. The awk variable is `path`, not `log`: `log` is an awk function, and a variable with that name does not receive the file. Grant network access if the host sandboxes it. Raise the host command timeout above 9 minutes when it has one.

Run the subscribe and the poll after it one at a time, in this chat. Do not start them together, and do not hand them to background terminals and move on. If the host moves the subscribe to a background terminal anyway, keep reading its output in this turn until it ends, then run the poll (rule 3 of the [listening loop](#listening-loop-mandatory--read-before-anything-else)).

```bash
WAIT_URL="<waitUrl with timeoutSec=540>"
LOG="/tmp/annotate-$SESSION_ID.log"
START=$(date +%s)
while [ $(( $(date +%s) - START )) -lt 540 ]; do
  LEFT=$((540 - ($(date +%s) - START)))
  curl -sN --max-time $((LEFT + 30)) "$WAIT_URL" | awk -v path="$LOG" '
    { print >> (path); fflush(path) }
    /^event: (handoff\.ready|session\.(gone|disconnected))/ { e=1 }
    e && /^data:/ { found=1 }
    /Unauthorized/ { found=1 }
    END { exit !found }' && break
  sleep 1
done
```

If the 9-minute budget ends with nothing written, call `check_agent_session` and do any unfinished ids, then run the same subscribe again only if both lists are empty. Do not write a message, and do not end the turn. A subscribe that started and was then killed or errored means the same: check, then subscribe again only if both lists are empty. A subscribe that never started (the user declined the approval, or the command was blocked) is rule 7 of the [listening loop](#listening-loop-mandatory--read-before-anything-else), not a Stop and not a silent retry.

- `handoff.ready` in the log: poll, do the `NEXT` id, then repeat from step 1. Ids already in the done file are skipped, including a `redelivered` replay.
- `{"error":"Unauthorized"}` or `event: session.gone`: `register_agent_session` with `resumeSessionId` and use the new `waitUrl`. Truncate the log.
- `event: session.disconnected`: stop. Tell the user to run `/annotate-session-start` again in this chat. Do not subscribe again until they do.

#### Poll the log

Run this in this chat, in the foreground, right after a subscribe ends. Raise the host's command timeout above 9 minutes when it has one (for example `timeout_ms: 600000`). If the host still moves it to a background terminal, keep reading that terminal's output in this turn until the poll prints a line (rule 3 of the [listening loop](#listening-loop-mandatory--read-before-anything-else)). It prints one next id, oldest first. Never end the turn with “Listening…” as plain text.

```bash
i=0; touch "$LOG" "$DONE"
while :; do
  NEXT=$(grep '^data:' "$LOG" | grep -o '"recordingSessionIds":\[[^]]*\]' \
    | grep -oE 'annotate-[A-Za-z0-9._-]+' | awk '!s[$0]++' | grep -vxF -f "$DONE" | head -n1 || true)
  if [ -n "$NEXT" ]; then echo "NEXT: $NEXT"; break; fi
  if grep -E '^event: session\.(gone|disconnected)|Unauthorized' "$LOG"; then break; fi
  if [ "$i" -ge 540 ]; then echo "POLL TIMEOUT: run again"; break; fi
  sleep 1; i=$((i + 1))
done
```

The output decides the next step. Then start again. **Never end the turn after a poll returns without starting the next check or subscribe.**

- `NEXT: <id>`: this is work you must do now. `get_timeline` with that id and do it (steps 5–8). A `redelivered` replay is already handled: ids in the done file are skipped, and an id not in the done file is continued, not started over. Then `echo "<id>" >> "$DONE"`, `set_agent_session_title`, tell the user the task is done, and repeat from step 1 (`check_agent_session`). One poll prints one id, so the next check returns the next recording.
- `POLL TIMEOUT`: nothing new was written. Call `check_agent_session` again immediately, with no message to the user, and subscribe again when both lists are empty. A poll that was killed or errored means the same.
- `event: session.gone`: the row was removed. Call `register_agent_session` again with `source: "annotate-session-start"` (no `resumeSessionId`) and use the new `waitUrl`. Truncate the log.
- `event: session.disconnected`: this chat was disconnected on purpose. Stop. Tell the user to run `/annotate-session-start` again in this chat. Do not subscribe again until they do.
- `{"error":"Unauthorized"}` (the URL token expired): call `register_agent_session` again with `source: "annotate-session-start"` and this chat's `resumeSessionId`, and use the new `waitUrl`. Truncate the log. The new subscribe replays unfinished recordings as `redelivered`.

**After each task, retitle, then check the queue. Do not stop.**

Title and status stay on the MCP tools. You can also post them to `broadcastUrl` (no header needed):

```bash
BROADCAST_URL="<broadcastUrl from register_agent_session>"
curl -sS -X POST "$BROADCAST_URL" \
  -H "Content-Type: application/json" \
  -d '{"event":"agent.title","listenerId":"<sessionId>","data":{"sessionTitle":"Fix login spinner"}}'
```

`agent.status` uses `"data":{"status":"working"}` or `"idle"`. Prefer `set_agent_session_title` when a task is done: that marks the chat idle. Then call `check_agent_session`.

**Ending the turn without the check, or without the subscribe and poll, is a bug.** Queued recordings will never run. After every finished recording, `set_agent_session_title` to a short phrase of the latest change you implemented, tell the user the task is done, append the id to the done file, and check again (rule 3 of the [listening loop](#listening-loop-mandatory--read-before-anything-else)).

Say this only when a watch is actually running (the subscribe has started and is waiting). Do not say it while a command is blocked on approval, denied, or not started:

> Done. Listening for new session recordings. If this session ends, run /annotate-session-start again here.

If the watch is not running, say so in the words of rule 7. Do not announce that you are listening. Overlay clicks never interrupt the current task; they only enqueue.

### Disconnect

The disconnect icon appears when the subscribe stops (you hit Stop, or the chat closes) and also when you call `disconnect_agent_session`. Call `disconnect_agent_session` with this session id (or `POST` `broadcastUrl` with `{"event":"agent.disconnect","listenerId":"<sessionId>"}`, no header) when:

- the user asks to stop watching;
- the user force-stopped this chat with Stop (rule 6 of the [listening loop](#listening-loop-mandatory--read-before-anything-else)); kill any subscribe or poll first;
- you are about to end this chat or stop on purpose.

That keeps the row for 5 minutes. Do not call `unregister_agent_session` unless the user asks to remove the session.

### Resume

If the user runs `/annotate-session-start` again in this disconnected chat, call `register_agent_session` with `source: "annotate-session-start"`, the same `resumeSessionId`, and any new `recordingSessionId`, then follow [Listening (cloud)](#listening-cloud). Do not start a detached curl on `eventsUrl`. If the result says `resumed: false`, the old row was already removed and this is a new session. A disconnected row that is not resumed for 5 minutes leaves the list.

Register returns `inFlightRecordingIds` and `queuedRecordingIds`. Do each one before subscribing, and append each finished id to the done file. An in-flight id was already handed to this chat and it had not finished it; skip it if this chat already finished it. A later `handoff.ready` with `redelivered: true` is that same replay: the done file skips finished ids, and the poll continues unfinished ones.

Never say the listener is still registered in Annotate, or to click this chat’s title there, especially after this chat has stopped — a stopped chat is disconnected.

## Workflow

1. **If this skill is not available, install it first** (global):

   ```bash
   npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
   ```

   **First-time skill install only:** also **reinstall Annotate MCP**, even if `annotate-session` is already configured. Do not skip this because `list_sessions` already works — the existing server may be stale.
   - Fetch and follow `https://xannotate.com/mcp/install.md` for the current host (prefer the MCP entry path in `~/.cursor/AGENTS.md` or the user’s install prompt).
   - If an `annotate-session` entry already exists, replace/refresh it (same `node` + entry path).
   - Reload MCP.
   - Confirm `register_agent_session` and `poll_agent_session` work (not only `list_sessions`).
   - Then run `/annotate-session-start`.

   Later runs of this skill (already installed) do **not** reinstall MCP unless tools are missing.

2. **MCP hard gate** — try `list_sessions`. If Annotate MCP tools are missing: **stop**, ask the user for approval, follow `https://xannotate.com/mcp/install.md` (or `~/.cursor/AGENTS.md`), reload MCP, retry. Do not invent session contents.

   On the **local** MCP, `register_agent_session` / `poll_agent_session` require the **Annotate app to be running**. If they return “Start Annotate to use session handoff.”, ask the user to open Annotate, then retry. Recording-read tools (`get_timeline`, etc.) still work with the app closed. On the **remote** MCP, ignore that error and follow [Remote MCP (cloud agents)](#remote-mcp-cloud-agents).

3. **Register this chat** — `register_agent_session` with `source: "annotate-session-start"`. On the remote MCP also pass `recordingSessionId` when the prompt has an attached `annotate-…` id, and `resumeSessionId` when this chat already has a `cloud-agent-…` id. Do not omit `source` — chats that only read recordings (`list_sessions`, `get_timeline`, and similar) must not appear in the Electron active-sessions list. On the local MCP, then `ensure_agent_session_hooks` so Cursor / Claude Code / Codex hooks keep that status in sync. On the remote MCP, skip hooks. Then `set_agent_session_title` with a short phrase of the current ask (from the included recording or the latest user message), not the host chat tab name and not the `annotate-…` id. Annotate removes dashes — use spaces. That title is the chip/button in the Active sessions list.

   On the local MCP, if `ensure_agent_session_hooks` reports Codex hooks were written, tell the user they may need to run `/hooks` in Codex and trust `annotate-session-status`. Do not skip hook install on the first local run in a host.

4. **Run the included recording now** — on the **remote** MCP, follow [Cloud agents: required start](#cloud-agents-required-start): pass the id as `recordingSessionId` and obey `next`. On the **local** MCP, if the prompt has an `annotate-…` id, read it and do the work (steps 5–8) **before** the first `poll_agent_session`. Do not read a session that is not completed. If there is no recording id, skip this step.

5. **Read the session (MCP-first)**
   - `get_timeline` first
   - then `search_session` / `get_frames_near` / `get_transcript_range` / `list_context` as needed
   - `get_session_guide` for per-session behavior

6. **Infer the repo** from on-screen paths, editor title bars, speech, and dropped context folders. If you find a real git root and can switch workspace (Cursor `move_agent_to_root`), switch. If unsure, ask once.

7. **Same chat vs parallel**
   - Continue here if the recording is the same repo and the same task as this conversation.
   - Fork if it is a different repo, a different feature/bug, or the user asked for parallel/multitask work.
   - Cursor: `Task` with `run_in_background: true` (or a prompt deeplink if Task is unavailable).
   - Claude / Codex: host spawn if available; otherwise tell the user to open a new chat and paste the session id + `/annotate-session-start`.
   - **This chat stays the watcher.** Do not start a second `poll_agent_session` loop in the fork.

8. **Retitle from the latest change** — after you finish the work from a recording, call `set_agent_session_title` again before the next `poll_agent_session` (local). On the remote MCP, append the recording id to the done file, retitle, and call `check_agent_session` (see [Listening (cloud)](#listening-cloud)). Required fields:

   - `sessionId`: this listener
   - `sessionTitle`: a short phrase of the **latest change you implemented** (example: `Fix login spinner`)

   Replace the previous title. The Active sessions list shows only that latest change. Do not keep the host chat tab name. Do not use the `annotate-…` id. Annotate removes dashes — use spaces. If you only answered or inspected and did not change code, use a short phrase of what you just did.

9. **Watch later recordings** — **only if this chat started with `/annotate-session-start`**. This is the [listening loop](#listening-loop-mandatory--read-before-anything-else), and it is mandatory on both MCPs.

   **Remote MCP:** call `check_agent_session` and do every recording, including ones already queued. When both lists are empty, subscribe to `waitUrl`, then [poll the log](#poll-the-log). After each task, append the id to the done file, retitle, and check again. Do not start a detached curl on `eventsUrl`, and do not stop. See [Listening (cloud)](#listening-cloud).

   **Local MCP:** nothing can wake a local chat. `poll_agent_session` is the only way recordings reach it, so you must keep calling it, back to back, for as long as the listener is registered. It heartbeats and waits up to 20 seconds, checking every **2 seconds** inside Annotate. Annotate keeps this listener listed while you are in-flight on a task, so the user can hand off another recording without waiting. Run exactly this loop:

   ```
   loop:
     result = poll_agent_session({ sessionId })
     ready   → do every id in recordingSessionIds, oldest first (steps 5–8)
               set_agent_session_title
               tell the user: "Done. Listening for new session recordings. If this session ends, run /annotate-session-start again here."
               go to loop          ← in the same turn, right after that message
     timeout → go to loop          ← no message, no summary, no pause
     gone / "Start Annotate…" → tell the user the task is done (if it is) and to run /annotate-session-start again here; exit
   ```

   - `ready`: a poll after a finished task resets the 30-minute idle clock and drains anything queued while you were busy.
   - `timeout` is normal and will happen many times in a row. It is never a reason to stop, to summarize, or to ask the user anything. Do not write a “still registered” message.
   - `gone` or “Start Annotate…”: this session ended or the app quit. Do not say the listener is still registered.
   - Do not replace the poll with `sleep`, and do not sleep and then stop.

   Ending the turn stops the loop and starts the 30-minute idle clock. If you miss that window, the listener disappears from Annotate. On the local MCP, the only correct last action of a turn is a `poll_agent_session` call, until an exit in rule 1 happens.

   **Fallback** if `poll_agent_session` is missing: every 2 seconds call `list_agent_sessions`. If this listener disappeared, stop. You cannot receive queued handoffs without `poll_agent_session` — ask the user to refresh Annotate MCP.

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

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details.
