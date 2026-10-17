# Annotate session start — reference

## Install

Global (required):

```bash
npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
```

Project-local install is wrong for this skill — recordings are not tied to one repo.

## Two ids

| Id | Format | Meaning |
| --- | --- | --- |
| Agent listener `sessionId` | `agent-…` locally, `cloud-agent-…` on the remote MCP | This chat. Returned by `register_agent_session`. Shown in Annotate as `sessionTitle`. |
| Recording `sessionId` | `annotate-…` | A finished screen recording. In the Session hand off prompt and in `poll_agent_session.recordingSessionIds`. |

## MCP contract

Recording-read tools are unchanged and work with Annotate quit:

- `list_sessions` — completed recordings only; also `dataDir`, `sessionsRoot`, `framesRoot`, `filesRoot`
- `get_timeline` — first read
- `get_session_guide` / `search_session` / `get_frames_near` / `get_transcript_range` / `get_manifest_summary` / `list_context`

Listener tools require the Annotate app (localhost API):

| Tool | Use |
| --- | --- |
| `register_agent_session` | Create this chat’s listener. Optional `sessionTitle`. Starts `status: "working"`. Returns `{ sessionId, sessionTitle, status, queuedCount }`. On the remote MCP, also returns the stream URLs and the queue. |
| `check_agent_session` | Remote MCP only. Read this listener's in-flight and queued recording ids without taking them off the queue, and heartbeat the lease. Use it when no background curl is writing the log. |
| `set_agent_session_title` | `{ sessionId, sessionTitle }` — short phrase of the latest implemented change; replace after every finished recording |
| `set_agent_session_status` | `{ sessionId, status: "idle" \| "working" }` |
| `ensure_agent_session_hooks` | Install Cursor / Claude / Codex hooks that heartbeat this listener |
| `poll_agent_session` | **Watch loop.** Heartbeat + block up to `timeoutMs` (default 20000), checking every 2s. Returns `ready` + `recordingSessionIds`, `timeout`, or `gone`. |
| `list_agent_sessions` | Live `{ sessionId, sessionTitle, status, queuedCount }[]` |
| `unregister_agent_session` | Stop appearing in Annotate |

If Annotate is quit, **local** listener tools return `Start Annotate to use session handoff.`

## Remote MCP (cloud agents)

When `ensure_agent_session_hooks` is missing, or the server is HTTP `/api/mcp`, this chat is a cloud agent:

- Skip `ensure_agent_session_hooks` and the “Start Annotate” error. The desktop app does not need to be open for register or title.
- Do not call `poll_agent_session`. It is not on the remote MCP.
- The listener id is `cloud-agent-…`. It shows in the Annotate bar next to local listeners, with a cloud marker.
- `register_agent_session` returns `eventsUrl`, `waitUrl`, `broadcastUrl`, `inFlightRecordingIds`, `queuedRecordingIds`, `resumed`, and `next`. Follow `next` and the checklist in SKILL.md ([Cloud agents: required start](SKILL.md#cloud-agents-required-start)). Do every in-flight and queued id before starting the curl; skip an in-flight id this chat already finished. `eventsUrl` is `?stream=1` with no timeout. One detached curl appends it to `/tmp/annotate-$SESSION_ID.log` for as long as this chat is connected ([background curl](SKILL.md#background-curl)). `waitUrl` is the [foreground subscribe](SKILL.md#foreground-no-background-curl) (one recording, then the server closes it) for hosts that cannot keep that curl. `check_agent_session` returns the same URLs plus the current in-flight and queued ids, and it heartbeats the lease. It does not drain the queue. Both URLs carry a signed token scoped to this listener (valid 7 days, cannot enqueue or read other listeners), so no `Authorization` header or `ANNOTATE_MCP_KEY` is needed. A `401` means the token expired: register again with `resumeSessionId`.
- The open curl is the presence signal in background mode. Closing it sets `disconnectedAt` and Annotate shows the disconnect icon. The row stays for 5 minutes (`removeInMs`), then it leaves the list. Reconnecting or resuming clears the disconnect. A chat that never opens a stream and never calls `check_agent_session` drops off after the short backstop lease. `check_agent_session` refreshes that lease without clearing an explicit disconnect.
- Pass the prompt's attached `annotate-…` id as `recordingSessionId`. Pass this chat's existing `cloud-agent-…` id as `resumeSessionId`. If `resumed` is false, the old row was gone and this is a new session. Read a ready recording now; an upload still in progress arrives later as `handoff.ready`. Start the background curl only after that read.
- Listening is mandatory, and it happens in this chat. Start the detached curl once (`nohup`, writing only to `/tmp/annotate-$SESSION_ID.log`, pid in `/tmp/annotate-$SESSION_ID.pid`). In Cursor that `Shell` call uses `block_until_ms: 0` and `required_permissions: ["full_network"]`. Without network permission the sandbox blocks the realtime host and `curl -s` fails silently. Do not count it as connected until the pid is alive and the log shows `: open`. If either check fails, delete the pid file and use foreground mode. After every task, poll the log in the foreground for the next `annotate-…` id that is not in `/tmp/annotate-$SESSION_ID.done`, do that one recording, append the id to the done file, retitle, and poll again. Do not poll from a subagent. Do not use `notify_on_output`: the curl's output is the file, and this chat reads the file. Do not stop, and do not start a second curl.
- Foreground mode does not trust the log as the queue, because nothing is appending to it. After each task, `check_agent_session` and do unfinished in-flight ids, then queued ids, one at a time, checking again after each. When both lists are empty, subscribe to `waitUrl` with `timeoutSec=540`. That curl appends every line to the log and ends when the server closes after `handoff.ready` (or `session.gone`, `session.disconnected`, or `Unauthorized`). `: open`, `: keepalive`, and `event: timeout` do not end it. Do not kill curl on the data line; hanging up marks the chat disconnected. Then poll the log and do the new id. A 9-minute budget with no recording means subscribe again.
- `handoff.ready` carries `recordingSessionIds` only after that cloud copy is `complete`. Uploading ids stay queued. `redelivered: true` means this chat was already handed those ids and the previous stream died: the done file skips any id already finished here, and the poll continues any id not in that file. Reconnecting replays that list until `set_agent_session_title` finishes the task. `session.gone` means register again without `resumeSessionId`.
- Reading a queued recording with the MCP read tools (`get_timeline`, `get_session_guide`, and similar) also removes it from this listener's queue, so the Annotate queue count drops even if it was not taken from the stream.
- Only recordings the user saved with Cloud on can be handed off. Local-only recordings stay disabled in the Active sessions list.
- If the user asks to stop, call `disconnect_agent_session` (or `POST` `{ "event": "agent.disconnect", "listenerId" }` to `broadcastUrl`). Stopping the chat also closes the curl, which shows the disconnect icon. The row stays for 5 minutes. `unregister_agent_session` removes the row; use it only when the user asks to remove the session.
- To resume, run `/annotate-session-start` in the same chat and pass `resumeSessionId`. If that row is already gone, register returns `resumed: false` and a new id.
- Title and status stay on the MCP tools. `POST $broadcastUrl` with `{ "event": "agent.title" | "agent.status", "listenerId", "data" }` is the alternative.

`register_agent_session` starts the listener as **working**. Editor hooks (`preToolUse` / `Stop` / `SessionEnd`) bind this chat and keep heartbeats going so Annotate does not drop the chip while the agent is mid-task. While `poll_agent_session` has returned work and the agent has not polled again, the listener stays listed (in-flight). Overlay clicks enqueue only — they do not interrupt the current task. The next `poll_agent_session` after a finished task resets a **30-minute idle** window and drains anything queued while busy. Idle listeners with no poll for 30 minutes are removed. Codex may require `/hooks` trust after install.

## First run vs later runs

1. User copies **Session hand off** from the session-saved popup and pastes it here (includes `Session id: annotate-…`).
2. Install skill if needed. On that first install, reinstall Annotate MCP even if it already exists, then this workflow.
3. Register (working), `ensure_agent_session_hooks`, set the title from the current ask, run the included recording (same-chat vs parallel), then retitle from the latest implemented change.
4. Call `poll_agent_session` and **do not end the turn**.
5. The user can click this title after the next save to queue another recording, including while this chat is still working. After the current task, poll again so the queued recording runs.

`/annotate-session-start` with no recording id: register, title, poll. Do not steal another chat’s recordings.

**Do not poll** if this chat only received a Copy prompt (`Use this Annotate recording as the prompt`). Read that recording, then stop. No `register_agent_session`, no `poll_agent_session`.

## Same chat vs parallel

Continue here when:

- The recording’s inferred repo matches this workspace
- The ask is a continuation of the current task

Fork when:

- Different repo
- Different feature/bug
- User asked for parallel / multitask work

### Cursor

Prefer `Task` (`run_in_background: true`) with the recording session id and “do not start a second watcher.”

If Task is unavailable, open a Cursor prompt deeplink with the session-id prompt plus `/annotate-session-start`.

If the inferred repo differs and `move_agent_to_root` is available, switch before doing the work (same chat) or pass that root to the forked task.

### Claude / Codex / others

Use the host’s agent/task spawn if it exists. Otherwise tell the user to open a new chat and paste:

```
Run skill /annotate-session-start
Session id: annotate-…
```

Do not start a second watch loop in the forked chat.

## Watch loop

Host-agnostic. The 2-second poll lives in Annotate. Nothing can wake a local chat, so the loop only runs while you keep calling `poll_agent_session`:

```
poll_agent_session
→ gone / app quit: stop
→ ready: process recordingSessionIds (oldest first), retitle, tell the user, poll_agent_session
  (same turn; resets 30-minute idle; drains anything queued while busy)
→ timeout: poll_agent_session (no message; timeouts in a row are normal)
```

The last action of every turn is `poll_agent_session` until the session is gone or the user says stop. A text-only reply ends the loop.

After a finished recording, `set_agent_session_title` to a short phrase of the latest implemented change, tell the user the task is done, and that you are polling for new session recordings. If this session ends, they should run `/annotate-session-start` again here. Do not say the listener is still registered, or to click this chat’s title in Annotate — a stopped chat is unregistered.

Do not treat that status sentence as the end of work — call `poll_agent_session` after it. Ending the turn starts the 30-minute idle clock.

## First-time skill install

When this conversation just ran `npx skills add … annotate-session-start`, reinstall Annotate MCP **even if it already exists**:

1. Fetch `https://xannotate.com/mcp/install.md`.
2. Refresh the host’s `annotate-session` entry (Cursor `~/.cursor/mcp.json`, Claude Code `claude mcp add --scope user`, Codex `~/.codex/config.toml`, etc.). Prefer the absolute entry path from `~/.cursor/AGENTS.md` or the pasted install prompt.
3. Reload MCP.
4. Confirm `register_agent_session` and `poll_agent_session` are available.

Do not skip this because `list_sessions` already works. An older Annotate MCP bundle will not have listener tools.

Later `/annotate-session-start` runs skip MCP reinstall unless tools are missing.

## MCP missing

Stop. Ask for approval. Fetch `https://xannotate.com/mcp/install.md`. Do not invent session contents.
