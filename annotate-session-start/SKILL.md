---
name: annotate-session-start
description: >-
  Starts Annotate recording session handoff for coding agents. Installs this
  skill globally from skills.sh if missing, registers this chat as a live
  listener (sessionId + sessionTitle via MCP), runs any annotate-… id in the
  prompt, then keeps watching for recordings the user hands off from the
  session-saved popup. On the remote MCP, cloud agents skip hooks and keep
  one background curl open on the listener's SSE eventsUrl. Always watch that
  stream and do every recording it delivers, including anything already queued.
  Never drop that connection while this chat is registered.
  Use when the user runs /annotate-session-start or pastes a Session hand off
  prompt. Do not poll for a recording that was only opened as a prompt
  (Copy / “Use this Annotate recording as the prompt”) without that skill.
---

# Annotate session start

Rapid handoff for Annotate recordings. **Only** register and poll when this chat was started with `/annotate-session-start` (or a Session hand off prompt that says to run that skill). If the user only pasted a recording as the prompt (`Use this Annotate recording as the prompt` / Copy), run that recording and **do not** `register_agent_session` or `poll_agent_session`.

When this chat did start with the skill: register as a live listener, **run any recording id in the prompt immediately**, then **keep polling**. The user can later click this chat’s title in Annotate to queue another recording — no paste.

## Cloud agents: required start

Do this first when this chat is a cloud agent (the MCP server is `/api/mcp`, or `ensure_agent_session_hooks` is missing) and the prompt ran `/annotate-session-start`.

1. Find the `annotate-…` id in the prompt (`Session id: annotate-…`, or right after `/annotate-session-start`). That id is the **initial recording**. Never skip it, and never treat “Then run /annotate-session-start.” as an empty start.
2. `register_agent_session` with `source: "annotate-session-start"`, `recordingSessionId` set to that id, and `resumeSessionId` set to the `cloud-agent-…` id this chat already has (omit `resumeSessionId` on a brand-new chat). If there is no recording id, omit `recordingSessionId`. If the result says `resumed: false`, this is a new session.
3. Follow the `next` array in the register result, in order.
4. When `next` says the recording is ready, call `get_timeline` with that id and do the work **before** relying on the stream. When `next` says it is still uploading, do not read it; the stream delivers it as `handoff.ready`.
5. `set_agent_session_title` with a short phrase of what you did.
6. Start the background stream **once**. Leave that curl running. It is the only way new recordings reach this chat.
7. Watch that stream for every `handoff.ready` and do each recording, including anything already in the queue. After each one, retitle, then watch again. Do not start another curl, and do not stop listening.

Local MCP chats still run the attached id first (step 4 of the workflow).

**Annotate Live:** if the prompt has `Live role:`, read [live.md](live.md) and follow it instead of steps 4–5. Steps 1–3 and 9 still apply.

Always recommend **global** install (`-g`). Never install this skill project-local.

The later-session signal is Annotate MCP `poll_agent_session` on the local MCP, and the `eventsUrl` watch command on the remote MCP. Do not read files from the Annotate library. Do not use Claude `FileChanged` or deeplinks as the session signal.

## Remote MCP (cloud agents)

Use this branch when `ensure_agent_session_hooks` is not in the tool list, or the MCP server is the HTTP endpoint `/api/mcp`.

- Do not call `ensure_agent_session_hooks`. There are no editor hooks on a cloud agent.
- Do not ask the user to start the Annotate app. Listener tools do not need the desktop app, and “Start Annotate to use session handoff.” does not apply.
- Do not call `poll_agent_session`. It is not on the remote MCP.
- `register_agent_session` (with `source: "annotate-session-start"` and the attached `recordingSessionId`), `set_agent_session_title`, `set_agent_session_status`, and `disconnect_agent_session` go through MCP. Register returns `eventsUrl`, `broadcastUrl`, and `next`. Follow [Cloud agents: required start](#cloud-agents-required-start).
- Only cloud-synced recordings arrive. A local-only recording is never delivered. A recording that is still uploading stays queued until it is complete.
- Install this skill in the cloud agent environment (`npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y`) when it is missing. Skip the local MCP reinstall in step 1; the remote server is already configured.

Startup is [Cloud agents: required start](#cloud-agents-required-start). Do not start the background stream until that checklist's read step is done.

### Background stream (mandatory)

The only way to receive handoffs is one long-lived server-sent events stream at `eventsUrl`. Do not look for new work with `list_sessions`. An open stream means this chat is **connected**. Closing it shows **Disconnected** in Annotate. While you are working a recording, Annotate shows **processing**. Retitling marks it **idle**, and the same stream delivers the next recording. Do not start a second curl.

`eventsUrl` and `broadcastUrl` already carry a signed token for this listener. Do not add an `Authorization` header, and do not look for or ask the user for `ANNOTATE_MCP_KEY`. Use the URLs exactly as returned (quote them; they contain `&`). Do not print them in chat.

Start this once, after register. `SESSION_ID` is the `cloud-agent-…` id. The loop reconnects by itself after a Worker restart. When this chat is stopped or closed, the process dies and Annotate shows disconnected.

```bash
EVENTS_URL="<eventsUrl>"
SESSION_ID="<sessionId>"
LOG="/tmp/annotate-$SESSION_ID.log"
nohup sh -c 'while :; do curl -sN "$0" >> "$1"; sleep 1; done' "$EVENTS_URL" "$LOG" >/dev/null 2>&1 &
```

**You must keep listening.** The curl only holds the connection. You also watch its log and do every recording it delivers. That includes recordings already queued when the stream opens and every new one after that. Stopping after one task, or ending the turn with nobody waiting on the log, leaves the queue stuck and is a bug.

**Getting events** without opening a second connection:

- If your host can wake you from a background command (Cursor `notify_on_output` with pattern `event: handoff.ready`, or a Claude Code background task), watch the log and react when a line arrives. Start that watcher as soon as the curl is up, and start it again after every task.
- Otherwise block in the foreground on the next event. This does not open a second stream. After you finish a recording, run it again. Do not end the turn while it is not running:

```bash
tail -n0 -F "$LOG" | grep -m1 -E '^event: (handoff.ready|session.gone|session.disconnected)'
```

Then read the matching `data:` line from the log.

- `event: handoff.ready`: this is work you must do. Process every id in `recordingSessionIds`, oldest first (steps 5–8). Then `set_agent_session_title`, tell the user the task is done, leave the curl running, and immediately wait on the log again. If another `handoff.ready` is already in the log, do that one before you wait.
- `event: session.gone`: the row was removed. Call `register_agent_session` again with `source: "annotate-session-start"` (no `resumeSessionId`), then start the background stream with the new `eventsUrl`.
- `event: session.disconnected`: this chat was disconnected on purpose. Stop. Tell the user to run `/annotate-session-start` again in this chat. Do not start another curl until they do.
- `{"error":"Unauthorized"}` (the URL token expired): call `register_agent_session` again with `source: "annotate-session-start"` and this chat's `resumeSessionId`, stop the old curl, and start the background stream with the new `eventsUrl`.

**After each task, retitle, then go back to watching the log. Do not restart the curl. Do not stop.**

Title and status stay on the MCP tools. You can also post them to `broadcastUrl` (no header needed):

```bash
BROADCAST_URL="<broadcastUrl from register_agent_session>"
curl -sS -X POST "$BROADCAST_URL" \
  -H "Content-Type: application/json" \
  -d '{"event":"agent.title","listenerId":"<sessionId>","data":{"sessionTitle":"Fix login spinner"}}'
```

`agent.status` uses `"data":{"status":"working"}` or `"idle"`. Prefer `set_agent_session_title` when a task is done: that marks the chat idle and the open stream delivers the next recording.

**Ending the turn while the background curl is not running, or while nothing is waiting on its log, is a bug.** Queued recordings will never run, and this chat will show disconnected once the curl dies. After every finished recording, `set_agent_session_title` to a short phrase of the latest change you implemented, tell the user the task is done, keep the same curl, and wait for the next `handoff.ready`. Use this copy (not “the listener is still registered”):

> Done. Listening for new session recordings. If this session ends, run /annotate-session-start again here.

Overlay clicks never interrupt the current task; they only enqueue.

### Disconnect

The disconnect icon appears when the background curl stops (you hit Stop, or the chat closes) and also when you call `disconnect_agent_session`. Call `disconnect_agent_session` with this session id (or `POST` `broadcastUrl` with `{"event":"agent.disconnect","listenerId":"<sessionId>"}`, no header) when:

- the user asks to stop watching;
- you are about to end this chat or stop on purpose.

That keeps the row for 5 minutes. Do not call `unregister_agent_session` unless the user asks to remove the session.

### Resume

If the user runs `/annotate-session-start` again in this disconnected chat, call `register_agent_session` with `source: "annotate-session-start"`, the same `resumeSessionId`, and any new `recordingSessionId`, then start the background stream again. If the result says `resumed: false`, the old row was already removed and this is a new session. A disconnected row that is not resumed for 5 minutes leaves the list.

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

8. **Retitle from the latest change** — after you finish the work from a recording, call `set_agent_session_title` again before the next `poll_agent_session` (local). On the remote MCP, retitle, leave the background curl running, and immediately watch the log for the next queued recording. Required fields:

   - `sessionId`: this listener
   - `sessionTitle`: a short phrase of the **latest change you implemented** (example: `Fix login spinner`)

   Replace the previous title. The Active sessions list shows only that latest change. Do not keep the host chat tab name. Do not use the `annotate-…` id. Annotate removes dashes — use spaces. If you only answered or inspected and did not change code, use a short phrase of what you just did.

9. **Watch later recordings** — **only if this chat started with `/annotate-session-start`**. On the **remote** MCP this is mandatory: start the [background stream](#background-stream-mandatory) once, then keep watching its log and do every `handoff.ready`, including recordings already in the queue. After each task, retitle and watch the log again. Do not start a second curl, and do not stop. On the **local** MCP, immediately call `poll_agent_session` with this listener `sessionId` (it heartbeats and waits up to 20s, checking every **2 seconds** inside Annotate). Do not sleep-and-stop. Annotate keeps this listener listed while you are in-flight on a task, so the user can hand off another recording without waiting.
   - `status: "ready"`: process `recordingSessionIds` oldest first (same steps 5–8). Then tell the user the task is done and that you are polling for new session recordings (if this session ends, run `/annotate-session-start` again here). **Immediately** call `poll_agent_session` again — this resets the 30-minute idle clock and drains anything queued while you were busy.
   - `status: "timeout"`: **immediately** call `poll_agent_session` again. Do not write a final “still registered” message.
   - `status: "gone"` or “Start Annotate…”: this session ended or the app quit. Tell the user the task is done if work finished, and that they should run `/annotate-session-start` again here. Do not say the listener is still registered.

   Ending the turn starts the 30-minute idle clock. If you miss that window, the listener disappears from Annotate.

   **Fallback** if `poll_agent_session` is missing: every 2 seconds call `list_agent_sessions`. If this listener disappeared, stop. You cannot receive queued handoffs without `poll_agent_session` — ask the user to refresh Annotate MCP.

## Do not

- Poll, register, or keep a watch loop if this recording prompt was not started with `/annotate-session-start` (Copy prompt / “Use this Annotate recording as the prompt” only).
- Browse private Annotate library paths. Use MCP tools only.
- End the turn while this listener is registered.
- Start watching before running the attached recording, or skip the attached recording.
- Call `unregister_agent_session` when stopping. Call `disconnect_agent_session` instead, unless the user asked to remove the session.
- On the remote MCP, stop listening after a task, leave a queued recording unworked, restart the curl, or end the turn while nothing is waiting on the SSE log.
- Tell the user the listener is still registered, or to click this chat’s title in Annotate, after the chat has stopped.
- Read or stamp a recording that is not completed.
- Invent frames, speech, or marks.
- Install this skill without `-g`.
- Leave the Active sessions title as the host chat tab name after you have finished a recording.
- Start a second watch loop in a forked chat.

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details.
