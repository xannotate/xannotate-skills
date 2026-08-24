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
| `get_handoff_state` | Read the shared flag and `pendingSessionId` |
| `set_handoff_state` | `{ autoHandOff?, lastProcessedSessionId?, workspaceHint?, stampSessionId? }` |
| `get_timeline` | First read of a completed session |
| `get_manifest_summary` | Includes `completed` and `autoHandOff` when stamped |
| `search_session` / `get_frames_near` / `get_transcript_range` / `list_context` | Evidence |

`stampSessionId` writes that session’s `.manifest.json` `autoHandOff: true`. Use it on skill start for the latest completed session (it was saved while the global flag was still false).

Later sessions: Annotate writes `completed: true` on the manifest at finalize, and if `autoHandOff` is already true it also writes `pendingSessionId` on `handoff.json` and stamps `autoHandOff` on the manifest.

## First run vs later runs

1. User copies **Auto hand off prompt** from the session-saved popup and pastes it here.
2. Install skill if needed, then this workflow.
3. Latest **completed** session runs immediately (same-chat vs parallel).
4. Poll `get_handoff_state` every 1 second. When `pendingSessionId` differs from `lastProcessedSessionId`, run that session (and drain any other completed ids after `lastProcessedSessionId`).
5. **Disable hand off** in the app writes `autoHandOff: false`. On the next poll, stop the loop.

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

Host-agnostic. After the first session (or when idle):

```
sleep 1
get_handoff_state
```

Use whatever 1-second sleep the host has (`sleep 1`, a 1s timer). Do not use Cursor `/loop`, file-change hooks, or deeplinks for this watch. Do not open `handoff.json` on disk.

Each tick:

1. `get_handoff_state`
2. If `autoHandOff` is false → stop listening
3. If `pendingSessionId` is set and ≠ `lastProcessedSessionId`:
   - `list_sessions`
   - process every completed session after `lastProcessedSessionId` (oldest first), including `pendingSessionId`
   - `set_handoff_state({ lastProcessedSessionId })` to the newest claimed id
4. Sleep 1 second and repeat

## MCP missing

Stop. Ask for approval. Fetch `https://xannotate.com/mcp/install.md`. Do not invent session contents.
