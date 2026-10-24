# Listening (cloud)

Detail for the remote MCP branch of [SKILL.md](SKILL.md). Rule numbers refer to the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else), and step numbers (5–8) to the [workflow](SKILL.md#workflow).

## Remote MCP setup

Use this branch when `ensure_agent_session_hooks` is not in the tool list, or the MCP server is the HTTP endpoint `/mcp`.

- Do not call `ensure_agent_session_hooks`. There are no editor hooks on a cloud agent.
- Do not ask the user to start the Annotate app. Listener tools do not need the desktop app, and “Start Annotate to use session handoff.” does not apply.
- Do not call `poll_agent_session`. It is not on the remote MCP.
- `register_agent_session` (with `source: "annotate-session-start"` and the attached `recordingSessionId`), `check_agent_session`, `set_agent_session_title`, `set_agent_session_status`, and `disconnect_agent_session` go through MCP. Register returns `eventsUrl`, `waitUrl`, `broadcastUrl`, `inFlightRecordingIds`, `queuedRecordingIds`, and `next`. Follow [Cloud agents: required start](#cloud-agents-required-start). `eventsUrl` is returned but this skill does not use it. `check_agent_session` is the queue read. It does not take recordings off the queue. Listening is that check, then a one-shot subscribe on `waitUrl`.
- Only cloud-synced recordings arrive. A local-only recording is never delivered. A recording that is still uploading stays queued until it is complete.
- Install this skill in the cloud agent environment (`npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y`) when it is missing. Skip the local MCP reinstall in SKILL.md step 1; the remote server is already configured.

## Cloud agents: required start

Do this first when this chat is a cloud agent (the MCP server is `/mcp`, or `ensure_agent_session_hooks` is missing) and the prompt ran `/annotate-session-start`.

1. Find the `annotate-…` id in the prompt (`Session id: annotate-…`, or right after `/annotate-session-start`). That id is the **initial recording**. Never skip it, and never treat “Then run /annotate-session-start.” as an empty start.
2. `register_agent_session` with `source: "annotate-session-start"`, `recordingSessionId` set to that id, and `resumeSessionId` set to the `cloud-agent-…` id this chat already has (omit `resumeSessionId` on a brand-new chat). If there is no recording id, omit `recordingSessionId`. If the result says `resumed: false`, this is a new session.
3. Follow the `next` array in the register result, in order. When the result includes `inFlightRecordingIds` or `queuedRecordingIds`, call `get_timeline` for each and do that work before listening. Skip an in-flight id this chat already finished.
4. When `next` says the recording is ready, call `get_timeline` with that id and do the work **before** listening. When `next` says it is still uploading, do not read it; listening delivers it as `handoff.ready`.
5. `set_agent_session_title` with a short phrase of what you did. Then send the feedback message ([rule 8](SKILL.md#listening-loop-mandatory--read-before-anything-else)) before any check or subscribe.
6. Touch the done file (`touch /tmp/annotate-$SESSION_ID.done`, where `SESSION_ID` is this chat's `cloud-agent-…` id). Then follow [Subscribe and done file](#subscribe-and-done-file). Do not start a detached curl on `eventsUrl`.
7. You are now in the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else). Do every recording `check_agent_session` or the poll returns, including anything already queued. After each one, retitle, then check again. Do not stop listening.

## Subscribe and done file

Recordings arrive through `check_agent_session` and one server-sent events subscribe on `waitUrl`. Do not look for new work with `list_sessions`. Do not start a detached curl on `eventsUrl`. The subscribe only writes a log. This chat polls that log itself, in the foreground, after the subscribe ends. A subagent must not poll for you.

An open subscribe means this chat is **connected**. Closing it early shows **Disconnected** in Annotate. While you are working a recording, Annotate shows **processing**. Retitling marks it **idle**. `check_agent_session` heartbeats the lease between subscribes.

`eventsUrl`, `waitUrl`, and `broadcastUrl` already carry a signed token for this listener. Do not add an `Authorization` header, and do not look for or ask the user for `ANNOTATE_MCP_KEY`. `eventsUrl` is returned but not used. Use `waitUrl` and `broadcastUrl` exactly as returned (quote them; they contain `&`). Do not print them in chat.

`SESSION_ID` is the `cloud-agent-…` id. Two files, both under `/tmp`:

- `LOG=/tmp/annotate-$SESSION_ID.log` — raw SSE lines, append-only. Truncate it only when you subscribe with a new `waitUrl` (the first start, `session.gone`, or `Unauthorized`).
- `DONE=/tmp/annotate-$SESSION_ID.done` — one finished `annotate-…` id per line. Append an id after you finish that recording. Never truncate it. This replaces any count of events you have seen, so a restart and a `redelivered` replay both skip work this chat already finished.

**You must keep listening.** Stopping after one task, or ending the turn without the check (or the subscribe and poll), leaves the queue stuck and is a bug.

The log is not the queue, because nothing is appending to it until you subscribe. After each task:

1. `set_agent_session_title` for the task you just finished. That clears it from the replay list. Then send the feedback message ([rule 8](SKILL.md#listening-loop-mandatory--read-before-anything-else)): what was done, the intent in the recording, and how you answer it. Do this before step 2.
2. `check_agent_session` with this chat's `sessionId`. It heartbeats the lease and does not take recordings off the queue.
3. `disconnectedAt` greater than 0 means this chat was disconnected. Stop, the same as `session.disconnected`. "Agent session is not registered." means the row is gone: treat it as `session.gone`.
4. Do every id in `inFlightRecordingIds`, then every id in `queuedRecordingIds`, that is not already in the done file. One at a time, oldest first (`get_timeline`, steps 5–8). Skip an in-flight id this chat already finished. After each one, `echo "<id>" >> "$DONE"`, `set_agent_session_title`, send the feedback message (rule 8), and call `check_agent_session` again. A recording queued while you worked shows up on that next check.
5. When both lists are empty, send the brief empty-queue line ([rule 8](SKILL.md#listening-loop-mandatory--read-before-anything-else)), then call `subscribe_agent_session` with this chat's `sessionId` and append the returned text to the log. If that result is `handoff.ready`, including `redelivered: true`, do that recording even when the queue count is already 0: it was handed off and this chat had not finished it. `event: timeout` is not a recording and is not an empty queue: call `check_agent_session` and do any unfinished ids before calling it again. If that tool is missing or the call errors, call `check_agent_session` first and do every unfinished in-flight id, then every queued id. Only when both lists are empty, send the brief empty-queue line, then subscribe once with the curl below. Then [poll the log](#poll-the-log) right away and do the `NEXT` id. Then repeat from step 1.

Touch `$DONE` before the first subscribe (`touch /tmp/annotate-$SESSION_ID.done`).

The subscribe writes every line to the log and ends only after a new recording (or `session.gone`, `session.disconnected`, or `Unauthorized`). `: open`, `: keepalive`, and `event: timeout` are not recordings, so they do not end it. Set `timeoutSec=540` on `waitUrl` (replace `1800`) so the server closes with `event: timeout` inside the 9-minute budget. Let curl run until the server closes. Do not exit on the data line and do not kill curl: `waitUrl` closes itself after one event, and hanging up early shows this chat as disconnected. The awk variable is `path`, not `log`: `log` is an awk function, and a variable with that name does not receive the file. Grant network access if the host sandboxes it. Raise the host command timeout above 9 minutes when it has one.

Run the subscribe and the poll after it one at a time, in this chat. Do not start them together, and do not hand them to background terminals and move on. If the host moves the subscribe to a background terminal anyway, keep reading its output in this turn until it ends, then run the poll (rule 3 of the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else)).

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

If the 9-minute budget ends with nothing written, call `check_agent_session` and do any unfinished ids, then run the same subscribe again only if both lists are empty. Do not write a message about the timeout itself, and do not end the turn. The brief empty-queue line (rule 8) still goes out before that subscribe. A subscribe that started and was then killed or errored means the same: check, then subscribe again only if both lists are empty. A subscribe that never started (the user declined the approval, or the command was blocked) is rule 7 of the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else), not a Stop and not a silent retry.

- `handoff.ready` in the log: poll, do the `NEXT` id, then repeat from step 1. Ids already in the done file are skipped, including a `redelivered` replay.
- `{"error":"Unauthorized"}` or `event: session.gone`: `register_agent_session` with `resumeSessionId` and use the new `waitUrl`. Truncate the log.
- `event: session.disconnected`: stop. Tell the user to run `/annotate-session-start` again in this chat. Do not subscribe again until they do.

## Poll the log

Run this in this chat, in the foreground, right after a subscribe ends. Raise the host's command timeout above 9 minutes when it has one (for example `timeout_ms: 600000`). If the host still moves it to a background terminal, keep reading that terminal's output in this turn until the poll prints a line (rule 3 of the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else)). It prints one next id, oldest first. Never end the turn with “Listening…” as plain text.

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

- `NEXT: <id>`: this is work you must do now. `get_timeline` with that id and do it (steps 5–8). A `redelivered` replay is already handled: ids in the done file are skipped, and an id not in the done file is continued, not started over. Then `echo "<id>" >> "$DONE"`, `set_agent_session_title`, tell the user the task is done with the feedback message (rule 8), and repeat from step 1 (`check_agent_session`). One poll prints one id, so the next check returns the next recording.
- `POLL TIMEOUT`: nothing new was written. Call `check_agent_session` again immediately, with no message about the timeout, and subscribe again when both lists are empty, after the brief empty-queue line (rule 8). A poll that was killed or errored means the same.
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

**Ending the turn without the check, or without the subscribe and poll, is a bug.** Queued recordings will never run. After every finished recording, `set_agent_session_title` to a short phrase of the latest change you implemented, tell the user the task is done with the feedback message (rule 8: what was done, the intent in the recording, how you answer it), append the id to the done file, and check again (rule 3 of the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else)).

Say this only when a watch is actually running (the subscribe has started and is waiting). Do not say it while a command is blocked on approval, denied, or not started:

> Done. Listening for new session recordings. If this session ends, run /annotate-session-start again here.

If the watch is not running, say so in the words of rule 7. Do not announce that you are listening. Overlay clicks never interrupt the current task; they only enqueue.

## Disconnect

The disconnect icon appears when the subscribe stops (you hit Stop, or the chat closes) and also when you call `disconnect_agent_session`. Call `disconnect_agent_session` with this session id (or `POST` `broadcastUrl` with `{"event":"agent.disconnect","listenerId":"<sessionId>"}`, no header) when:

- the user asks to stop watching;
- the user force-stopped this chat with Stop (rule 6 of the [listening loop](SKILL.md#listening-loop-mandatory--read-before-anything-else)); kill any subscribe or poll first;
- you are about to end this chat or stop on purpose.

That keeps the row for 5 minutes. Do not call `unregister_agent_session` unless the user asks to remove the session.

## Resume

If the user runs `/annotate-session-start` again in this disconnected chat, call `register_agent_session` with `source: "annotate-session-start"`, the same `resumeSessionId`, and any new `recordingSessionId`, then follow [Subscribe and done file](#subscribe-and-done-file). Do not start a detached curl on `eventsUrl`. If the result says `resumed: false`, the old row was already removed and this is a new session. A disconnected row that is not resumed for 5 minutes leaves the list.

Register returns `inFlightRecordingIds` and `queuedRecordingIds`. Do each one before subscribing, and append each finished id to the done file. An in-flight id was already handed to this chat and it had not finished it; skip it if this chat already finished it. A later `handoff.ready` with `redelivered: true` is that same replay: the done file skips finished ids, and the poll continues unfinished ones.
