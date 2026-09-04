---
name: annotate-session-start
description: >-
  Starts Annotate recording session handoff for coding agents. Installs this
  skill globally from skills.sh if missing, registers this chat as a live
  listener (sessionId + sessionTitle via MCP), runs any annotate-… id in the
  prompt, then polls every 2 seconds for recordings the user hands off from the
  session-saved popup. Never ends the turn while registered.
  Use when the user runs /annotate-session-start or pastes a Session hand off
  prompt. Do not poll for a recording that was only opened as a prompt
  (Copy / “Use this Annotate recording as the prompt”) without that skill.
---

# Annotate session start

Rapid handoff for Annotate recordings. **Only** register and poll when this chat was started with `/annotate-session-start` (or a Session hand off prompt that says to run that skill). If the user only pasted a recording as the prompt (`Use this Annotate recording as the prompt` / Copy), run that recording and **do not** `register_agent_session` or `poll_agent_session`.

When this chat did start with the skill: register as a live listener, **run any recording id in the prompt immediately**, then **keep polling**. The user can later click this chat’s title in Annotate to queue another recording — no paste.

Always recommend **global** install (`-g`). Never install this skill project-local.

The later-session signal is Annotate MCP (`poll_agent_session`). Do not read files from the Annotate library. Do not use Claude `FileChanged` or deeplinks as the session signal.

**Ending the turn while this listener is registered is a bug.** Later handoffs will sit in the queue and never run. After every finished recording, tell the user the task is done, then keep polling. Use this copy (not “the listener is still registered”):

> Done. Polling for new session recordings. If this session ends, run /annotate-session-start again here.

Then your last action must be another `poll_agent_session`. That next poll resets a **30-minute idle window** and picks up anything the user queued in Annotate while you were working. Overlay clicks never interrupt the current task; they only enqueue.

Never say the listener is still registered in Annotate, or to click this chat’s title there, especially after this chat has stopped — a stopped chat is unregistered.

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

   `register_agent_session` / `poll_agent_session` require the **Annotate app to be running**. If they return “Start Annotate to use session handoff.”, ask the user to open Annotate, then retry. Recording-read tools (`get_timeline`, etc.) still work with the app closed.

3. **Register this chat** — `register_agent_session` with `source: "annotate-session-start"` (starts `status: "working"` so Annotate keeps this listener listed while you work). Do not omit `source` — chats that only read recordings (`list_sessions`, `get_timeline`, and similar) must not appear in the Electron active-sessions list. Then `ensure_agent_session_hooks` so Cursor / Claude Code / Codex hooks keep that status in sync. Then `set_agent_session_title` to **this chat’s visible title** — the sidebar/tab name in Cursor, Claude Code, or Codex. Keep the same words, case, and spaces; Annotate removes dashes. Do not invent a task summary like `Login fix`. If the host has not named the chat yet, use the shortest phrase from the latest user message, then call `set_agent_session_title` again when the chat title appears. That title is the chip/button the user sees in Annotate.

   If `ensure_agent_session_hooks` reports Codex hooks were written, tell the user they may need to run `/hooks` in Codex and trust `annotate-session-status`. Do not skip hook install on the first run in a host.

4. **Run the included recording now** — if the prompt has `Session id: annotate-…`, that recording is the first handoff. Do not wait for the next stop. Do not read a session that is not completed. If there is no recording id, skip this step.

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

8. **Watch later recordings** — **only if this chat started with `/annotate-session-start`**. Then immediately call `poll_agent_session` with this listener `sessionId` (it heartbeats and waits up to 20s, checking every **2 seconds** inside Annotate). Do not sleep-and-stop. Annotate keeps this listener listed while you are in-flight on a task, so the user can hand off another recording without waiting.
   - `status: "ready"`: process `recordingSessionIds` oldest first (same steps 5–7). Then tell the user the task is done and that you are polling for new session recordings (if this session ends, run `/annotate-session-start` again here). **Immediately** call `poll_agent_session` again — this resets the 30-minute idle clock and drains anything queued while you were busy.
   - `status: "timeout"`: **immediately** call `poll_agent_session` again. Do not write a final “still registered” message.
   - `status: "gone"` or “Start Annotate…”: this session ended or the app quit. Tell the user the task is done if work finished, and that they should run `/annotate-session-start` again here. Do not say the listener is still registered.

   Ending the turn starts the 30-minute idle clock. If you miss that window, the listener disappears from Annotate.

   **Fallback** if `poll_agent_session` is missing: every 2 seconds call `list_agent_sessions`. If this listener disappeared, stop. You cannot receive queued handoffs without `poll_agent_session` — ask the user to refresh Annotate MCP.

## Do not

- Poll, register, or keep a watch loop if this recording prompt was not started with `/annotate-session-start` (Copy prompt / “Use this Annotate recording as the prompt” only).
- Browse private Annotate library paths. Use MCP tools only.
- End the turn while this listener is registered.
- Tell the user the listener is still registered, or to click this chat’s title in Annotate, after the chat has stopped.
- Read or stamp a recording that is not completed.
- Invent frames, speech, or marks.
- Install this skill without `-g`.
- Start a second watch loop in a forked chat.

See [reference.md](reference.md) for MCP fields, host spawn notes, and poll details.
