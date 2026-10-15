# Annotate Live listener

Read this when the prompt has `Live role:`. Annotate Live opened this chat for a Gemini Live task or ask. The prompt carries only the variables:

```text
Live role: task | ask
Live request: live-…
Chat title: Task · … | Ask · …
Session title: …
```

## Register

1. Rename this chat to the `Chat title` so the sidebar matches the Live pill.
2. `register_agent_session` with `source: "annotate-session-start"`, `liveRole` from `Live role`, `liveRequestId` from `Live request`, and `sessionTitle` from `Session title`. Then `ensure_agent_session_hooks` (SKILL.md step 3).
3. `poll_agent_session`. The request from the prompt arrives as a poll item. Handle items as below, `reply_agent_request` for each, and keep polling exactly as SKILL.md step 9 says. Never end the turn while registered.

Do not read an unfinished recording. Live snapshots are frozen for you; use `get_agent_request` and `get_agent_request_frame`, never `get_timeline` on the live recording id.

## Task role

Annotate Live can run several task chats at once. This chat owns only the tasks routed to it, so keep working on what poll returns here. Tasks arrive one at a time.

Each task item has `text` (an implementation-ready prompt written by Gemini Live), `intent` (what the user wants and why), and `needsRecording`.

- **`needsRecording: false`** — implement directly from `text` and `intent`. Do not call `get_agent_request`. The full snapshot is a fallback only: call it if you cannot find the target in the code or the request is ambiguous.
- **`needsRecording: true`, or missing** — call `get_agent_request` first (transcript window, marks, frame list, fresh frame), then `get_agent_request_frame` for the frames you need. Trust `intent` for what the user wants; use the snapshot to find where and how.

Then `reply_agent_request` with `done` (one short sentence Gemini can speak) or `failed` with the reason.

After each task, set the session title to a short phrase of that change and rename this chat to `Task ·` plus that phrase, so Live can tell the user which session a later task is going to.

## Ask role

This chat is the one ask session for Annotate Live. Do not open another ask listener. `delegate_task` work does not belong here.

Do not wait for the current question to finish. Recording handoffs stay one at a time; asks do not. When poll returns asks:

1. `reply_agent_request` `in_progress` for each ask immediately.
2. Start a background sub-agent per ask, all at once, on the fastest model available (a fast or flash model, not a thinking model).
3. Poll again as soon as those sub-agents are started, so the next question starts in parallel. Do not wait for them. Do not answer questions yourself.

Each sub-agent:

- Calls `get_agent_request` for its request id and inspects the code.
- Researches in depth — Gemini Live sends the questions it cannot answer well on its own. Read the code, and look up docs or the web when the answer depends on something outside the code or newer than a model's knowledge cutoff.
- May suggest code in the answer but must not edit files.
- Calls `reply_agent_request` `done` with an answer short enough to be spoken.
- Must not register or poll.

When a new question arrives, rename this chat to `Ask ·` plus that question and set the session title to the question, so the pill and the sidebar stay the same.
