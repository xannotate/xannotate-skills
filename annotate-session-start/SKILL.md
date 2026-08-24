---
name: annotate-session-start
description: >-
  Starts Annotate recording auto handoff for coding agents. Installs this skill
  globally from skills.sh if missing, writes autoHandOff true, immediately runs
  the latest completed Annotate recording session, then polls handoff.json
  every 1 second for later sessions.
  Use when the user runs /annotate-session-start, pastes an Auto hand off
  prompt, or mentions Annotate recording auto hand off.
---

# Annotate session start

Rapid handoff for Annotate recordings. After the user pastes the Auto hand off prompt (or types `/annotate-session-start`), claim handoff, **run the latest completed recording immediately**, then poll for more.

Always recommend **global** install (`-g`). Never install this skill project-local.

This skill is host-agnostic. Use MCP tools and a 1-second sleep. Do not use Cursor `/loop`, Claude `FileChanged`, deeplinks, or other host-specific watchers for the later-session loop.

## Workflow

1. **If this skill is not available, install it first** (global):

   ```bash
   npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
   ```

   **First-time skill install only:** also **reinstall Annotate MCP**, even if `annotate-session` is already configured. Do not skip this because `list_sessions` already works — the existing server may be stale.
   - Fetch and follow `https://xannotate.com/mcp/install.md` for the current host (prefer the MCP entry path in `~/.cursor/AGENTS.md` or the user’s install prompt).
   - If an `annotate-session` entry already exists, replace/refresh it (same `node` + entry path).
   - Reload MCP.
   - Confirm `get_handoff_state` works (not only `list_sessions`).
   - Then run `/annotate-session-start`.

   Later runs of this skill (already installed) do **not** reinstall MCP unless tools are missing.

2. **MCP hard gate** — try `get_handoff_state` (or `list_sessions`). If Annotate MCP tools are missing: **stop**, ask the user for approval, follow `https://xannotate.com/mcp/install.md` (or `~/.cursor/AGENTS.md`), reload MCP, retry. Do not invent session contents.

3. **Claim handoff** — `set_handoff_state` with `autoHandOff: true`. If `list_sessions` has a completed session, also pass `stampSessionId` set to that **latest** id (`list_sessions` omits in-progress recordings). If the list is empty, still set `autoHandOff: true` and go to the 1s watch loop.

4. **Run the latest completed recording now** — that session is the first auto-handoff. Do not wait for the next stop. Do not read a session that is not completed. If none is completed yet, skip this step.

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

8. **Mark processed** — `set_handoff_state({ lastProcessedSessionId })` after claiming, before long work.

9. **Watch later recordings** — every **1 second** call `get_handoff_state` (sleep 1s between checks; any host: `sleep 1` or a 1-second timer). Do not read `handoff.json` from disk.
   - `autoHandOff` is **false** (Disable hand off): **stop listening** and say handoff is disabled.
   - `pendingSessionId` is set and different from `lastProcessedSessionId`: process that session. Also drain `list_sessions` for every completed id after `lastProcessedSessionId` (oldest first) so two finishes in one second are not dropped. Then `set_handoff_state({ lastProcessedSessionId })` to the newest one you claimed.
   - Otherwise sleep 1 second and check again.

## Do not

- Browse private Annotate library paths. Use MCP tools only (`get_handoff_state` / `set_handoff_state` / `list_sessions.handoff`).
- Read or stamp a recording that is not completed.
- Invent frames, speech, or marks.
- Install this skill without `-g`.

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details.
