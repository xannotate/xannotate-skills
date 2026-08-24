---
name: annotate-session-start
description: >-
  Starts Annotate recording auto handoff for coding agents. Installs this skill
  globally from skills.sh if missing, writes autoHandOff true, immediately runs
  the latest completed Annotate recording session, then keeps polling for later
  sessions via wait_for_handoff (1-second MCP poll). Never ends the turn while
  autoHandOff is true.
  Use when the user runs /annotate-session-start, pastes an Auto hand off
  prompt, or mentions Annotate recording auto hand off.
---

# Annotate session start

Rapid handoff for Annotate recordings. After the user pastes the Auto hand off prompt (or types `/annotate-session-start`), claim handoff, **run the latest completed recording immediately**, then **keep watching**.

Always recommend **global** install (`-g`). Never install this skill project-local.

The later-session signal is Annotate MCP (`wait_for_handoff`, or `get_handoff_state` + `list_sessions`). Do not read `handoff.json` from disk. Do not use Claude `FileChanged` or deeplinks as the session signal.

**Ending the turn while `autoHandOff` is true is a bug.** Later recordings will sit in `pendingSessionId` and never run. After the first session, your last action must be another wait/poll — not a closing “I’m watching” message.

## Workflow

1. **If this skill is not available, install it first** (global):

   ```bash
   npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
   ```

   **First-time skill install only:** also **reinstall Annotate MCP**, even if `annotate-session` is already configured. Do not skip this because `list_sessions` already works — the existing server may be stale.
   - Fetch and follow `https://xannotate.com/mcp/install.md` for the current host (prefer the MCP entry path in `~/.cursor/AGENTS.md` or the user’s install prompt).
   - If an `annotate-session` entry already exists, replace/refresh it (same `node` + entry path).
   - Reload MCP.
   - Confirm `get_handoff_state` and `wait_for_handoff` work (not only `list_sessions`).
   - Then run `/annotate-session-start`.

   Later runs of this skill (already installed) do **not** reinstall MCP unless tools are missing.

2. **MCP hard gate** — try `get_handoff_state` (or `list_sessions`). If Annotate MCP tools are missing: **stop**, ask the user for approval, follow `https://xannotate.com/mcp/install.md` (or `~/.cursor/AGENTS.md`), reload MCP, retry. Do not invent session contents.

3. **Claim handoff** — `set_handoff_state` with `autoHandOff: true`. If `list_sessions` has a completed session, also pass `stampSessionId` set to that **latest** id (`list_sessions` omits in-progress recordings). If the list is empty, still set `autoHandOff: true` and go to the watch loop.

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

9. **Watch later recordings** — immediately call `wait_for_handoff` (it polls every **1 second** inside MCP). Do not sleep-and-stop. Do not wait only for `pendingSessionId` to change — the tool also returns every completed id after `lastProcessedSessionId`.
   - `status: "disabled"` (`autoHandOff` false): **stop listening** and say handoff is disabled.
   - `status: "ready"`: process `sessionIds` oldest first (same steps 5–8). Then `set_handoff_state({ lastProcessedSessionId })` to the newest claimed id. Then **immediately** call `wait_for_handoff` again.
   - `status: "timeout"`: **immediately** call `wait_for_handoff` again. Do not write a final message.

   **Fallback** if `wait_for_handoff` is missing: every 1 second call `get_handoff_state` **and** `list_sessions`. Process every completed id after `lastProcessedSessionId` (oldest first), including when `pendingSessionId` equals an id already in that drain list. Then poll again. Your last tool call must be the next poll.

## Do not

- Browse private Annotate library paths. Use MCP tools only (`wait_for_handoff` / `get_handoff_state` / `set_handoff_state` / `list_sessions`).
- End the turn while `autoHandOff` is true.
- Read or stamp a recording that is not completed.
- Invent frames, speech, or marks.
- Install this skill without `-g`.

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details.
