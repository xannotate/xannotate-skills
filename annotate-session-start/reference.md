# Annotate session start — reference

## Install

Global (required):

```bash
npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
```

Project-local install is wrong for this skill — recordings are not tied to one repo.

## MCP contract

`list_sessions` returns only **completed** sessions:

- `dataDir`, `sessionsRoot`, `framesRoot`, `filesRoot`
- `handoff`: `{ version, autoHandOff, updatedAt, lastProcessedSessionId?, pendingSessionId?, workspaceHint? }`
- `sessions[]` newest first; each includes `completed: true` and may include `autoHandOff`

In-progress recordings are omitted. Session read tools (`get_timeline`, `get_session_guide`, `search_session`, `get_frames_near`, `get_transcript_range`, `get_manifest_summary`, `list_context`) refuse an incomplete session id.

Tools:

| Tool | Use |
| --- | --- |
| `wait_for_handoff` | **Watch loop.** Blocks up to `timeoutMs` (default 20000), polling every 1s. Returns `ready` + `sessionIds`, `disabled`, or `timeout`. |
| `get_handoff_state` | Snapshot: flag, `pendingSessionId`, `lastProcessedSessionId`, `unprocessedSessionIds` |
| `set_handoff_state` | `{ autoHandOff?, lastProcessedSessionId?, workspaceHint?, stampSessionId? }` |
| `get_timeline` | First read of a completed session |
| `get_manifest_summary` | Includes `completed` and `autoHandOff` when stamped |
| `search_session` / `get_frames_near` / `get_transcript_range` / `list_context` | Evidence |

`stampSessionId` writes that session’s `.manifest.json` `autoHandOff: true`. Use it on skill start for the latest completed session (it was saved while the global flag was still false).

Later sessions: Annotate writes `completed: true` on the manifest at finalize, and if `autoHandOff` is already true it also writes `pendingSessionId` on `handoff.json` and stamps `autoHandOff` on the manifest.

`wait_for_handoff` / `get_handoff_state.unprocessedSessionIds` drain **completed** ids after `lastProcessedSessionId`, not only a `pendingSessionId` change. Two finishes in one second are not dropped.

## First run vs later runs

1. User copies **Auto hand off prompt** from the session-saved popup and pastes it here.
2. Install skill if needed. On that first install, reinstall Annotate MCP even if it already exists, then this workflow.
3. Latest **completed** session runs immediately (same-chat vs parallel).
4. Mark `lastProcessedSessionId`, then call `wait_for_handoff` and **do not end the turn**.
5. **Disable hand off** in the app writes `autoHandOff: false`. The next `wait_for_handoff` returns `disabled` and the loop stops.

## Same chat vs parallel

Continue here when:

- The recording’s inferred repo matches this workspace
- The ask is a continuation of the current task

Fork when:

- Different repo
- Different feature/bug
- User asked for parallel / multitask work

### Cursor

Prefer `Task` (`run_in_background: true`) with the session id and “run `/annotate-session-start` for this session only (do not start a second watcher).”

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

Host-agnostic. The 1-second poll lives in MCP:

```
wait_for_handoff
→ disabled: stop
→ ready: process sessionIds (oldest first), set lastProcessed, wait_for_handoff
→ timeout: wait_for_handoff
```

Do not read `handoff.json` on disk. Do not treat a user-facing “watching for the next recording” sentence as the end of work — call `wait_for_handoff` after it.

Fallback if `wait_for_handoff` is missing:

1. `get_handoff_state` and `list_sessions` every 1 second
2. If `autoHandOff` is false → stop
3. Process every completed session after `lastProcessedSessionId` (oldest first), including `pendingSessionId` and `unprocessedSessionIds` when present
4. `set_handoff_state({ lastProcessedSessionId })` to the newest claimed id
5. Poll again immediately

## First-time skill install

When this conversation just ran `npx skills add … annotate-session-start`, reinstall Annotate MCP **even if it already exists**:

1. Fetch `https://xannotate.com/mcp/install.md`.
2. Refresh the host’s `annotate-session` entry (Cursor `~/.cursor/mcp.json`, Claude Code `claude mcp add --scope user`, Codex `~/.codex/config.toml`, etc.). Prefer the absolute entry path from `~/.cursor/AGENTS.md` or the pasted install prompt.
3. Reload MCP.
4. Confirm `get_handoff_state` and `wait_for_handoff` are available.

Do not skip this because `list_sessions` already works. An older Annotate MCP bundle will not have handoff tools.

Later `/annotate-session-start` runs skip MCP reinstall unless tools are missing.

## MCP missing

Stop. Ask for approval. Fetch `https://xannotate.com/mcp/install.md`. Do not invent session contents.
