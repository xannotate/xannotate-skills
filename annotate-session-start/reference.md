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
| Agent listener `sessionId` | `agent-…` | This chat. Returned by `register_agent_session`. Shown in Annotate as `sessionTitle`. |
| Recording `sessionId` | `annotate-…` | A finished screen recording. In the Session hand off prompt and in `poll_agent_session.recordingSessionIds`. |

## MCP contract

Recording-read tools are unchanged and work with Annotate quit:

- `list_sessions` — completed recordings only; also `dataDir`, `sessionsRoot`, `framesRoot`, `filesRoot`
- `get_timeline` — first read
- `get_session_guide` / `search_session` / `get_frames_near` / `get_transcript_range` / `get_manifest_summary` / `list_context`

Listener tools require the Annotate app (localhost API):

| Tool | Use |
| --- | --- |
| `register_agent_session` | Create this chat’s listener. Optional `sessionTitle`. Returns `{ sessionId, sessionTitle }`. |
| `set_agent_session_title` | `{ sessionId, sessionTitle }` — button label in the session-saved popup |
| `poll_agent_session` | **Watch loop.** Heartbeat + block up to `timeoutMs` (default 20000), checking every 2s. Returns `ready` + `recordingSessionIds`, `timeout`, or `gone`. |
| `list_agent_sessions` | Live `{ sessionId, sessionTitle }[]` |
| `unregister_agent_session` | Stop appearing in Annotate |

If Annotate is quit, listener tools return `Start Annotate to use session handoff.`

While `poll_agent_session` has returned work and the agent has not polled again, the listener stays listed (in-flight). Overlay clicks enqueue only — they do not interrupt the current task. The next `poll_agent_session` after a finished task resets a **30-minute idle** window and drains anything queued while busy. Idle listeners with no poll for 30 minutes are removed.

## First run vs later runs

1. User copies **Session hand off** from the session-saved popup and pastes it here (includes `Session id: annotate-…`).
2. Install skill if needed. On that first install, reinstall Annotate MCP even if it already exists, then this workflow.
3. Register, set a title, run the included recording (same-chat vs parallel).
4. Call `poll_agent_session` and **do not end the turn**.
5. The user can click this title after the next save to queue another recording, including while this chat is still working. After the current task, poll again so the queued recording runs.

`/annotate-session-start` with no recording id: register, title, poll. Do not steal another chat’s recordings.

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

Host-agnostic. The 2-second poll lives in Annotate:

```
poll_agent_session
→ gone / app quit: stop
→ ready: process recordingSessionIds (oldest first), poll_agent_session
  (resets 30-minute idle; drains anything queued while busy)
→ timeout: poll_agent_session
```

Do not treat a user-facing “watching for the next recording” sentence as the end of work — call `poll_agent_session` after it. Ending the turn starts the 30-minute idle clock.

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
