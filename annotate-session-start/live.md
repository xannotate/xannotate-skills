# Annotate Live coordinator

Read this when the prompt has `Live role:`. Annotate Live (the Gemini voice agent) opened this chat on the user's first code question or task. This chat is the **main agent chat**: the coordinator for every Live request in the run. The prompt carries only the variables:

```text
Live role: main
Live request: live-…
Chat title: Annotate Live
Session title: Annotate Live
```

A prompt with `Live role: task` or `Live role: ask` comes from an older Annotate build. Treat it as `main`.

## Register (handshake)

1. Rename this chat to the `Chat title`.
2. `register_agent_session` with `source: "annotate-session-start"`, `liveRole: "main"`, `liveRequestId` from `Live request`, and `sessionTitle` from `Session title`. Registering is the handshake: Annotate tells the user by voice that the coding agent is connected.
3. `poll_agent_session`. The request from the prompt, and any sent while you were opening, arrive as poll items. Keep polling exactly as SKILL.md step 9 says. Never end the turn while registered.

## Resume after a stop

If this chat stops polling (its turn ended, or it went more than about 90 seconds without `poll_agent_session`), Annotate marks it stopped. Live then asks the user whether to resume this chat or start a new one. When the user resumes, they run `/annotate-session-start` here again, usually with no `Live role:` line. If this chat registered as the Live coordinator earlier in this conversation (or it is titled `Annotate Live`), that run is a **resume**, not a normal handoff:

1. If you still have this chat's listener `sessionId`, call `poll_agent_session` with it straight away. If that returns `gone`, go to step 2.
2. Otherwise `register_agent_session` again with `source: "annotate-session-start"`, `liveRole: "main"`, `sessionTitle: "Annotate Live"`, and no `liveRequestId`. Annotate moves the requests waiting for the stopped chat to this registration.
3. Carry on with this file: reply to any results from workers that finished while stopped, then keep polling.

Any time `poll_agent_session` returns `gone` while you are the coordinator, re-register the same way. Never become a normal handoff listener from here.

This chat is the only Live listener and the only caller of `reply_agent_request`. Do not open another listener, and do not let a worker register, poll or reply.

Do not read an unfinished recording. Live snapshots are frozen for you; use `get_agent_request`, then `get_frames_near` with the `requestId` (or the PNGs in `framesDir`), never `get_timeline` or `get_frames_near` with the live recording id.

## Each poll item

Items are `kind: "ask"` (a read-only question) or `kind: "task"` (a change to make). Each has `requestId`, `text`, `needsRecording`, and for tasks `intent` (what the user wants and why). When `needsRecording` is false, Annotate took no snapshot and the item has no `recordingSessionId`. When the user attached files, folders, or clipboard snippets to the recording, the item also has `contextFiles`: each has `fileName`, `kind` (`file`, `folder`, or `snippet`) and a `path` to read, whatever `needsRecording` says. An item may also have `discussion`: Live's memory of a long conversation is trimmed, so `text` is only its summary. `discussion.summary` is a compressed recap of everything said since the last task sent, one line per turn (`U:` the user, `A:` Live), and `discussion.path` is a JSON file with every transcript segment of that stretch (`role` `user` or `assistant`).

Do not wait for one item to finish before taking the next. The user is talking to Live while this runs, so any delay is heard. For every item, all at once:

1. `reply_agent_request` `in_progress`.
2. Hand it to a sub-chat (below). Use a sub-agent only when no sub-chat is available.
3. Poll again as soon as the workers are started. Do not answer or implement anything yourself in this chat.

## Workers: always a sub-chat, sub-agent only as fallback

**Every request goes to a sub-chat.** A sub-chat is a separate chat or session started from this chat, which the user can open from the sidebar and whose result comes back here. Do not start a sub-agent when a sub-chat is available, even for a quick ask. Before the first request, check which of these the host offers, and use the first one that is available:

1. **Claude (Claude Desktop / Claude Code app):** start a sub-chat from this chat with the host's start-session tool, such as `start_session` (load it with tool search if it is only listed as deferred). Its result reaches this chat when that session ends its turn. This is the preferred path: always use it in Claude.
2. **Codex app:** create a separate chat (a new thread) for the request, using Codex's own way of starting a chat from this one, and read its result back here.
3. **Any other host:** use its tool for starting a separate chat or session, if it has one that reports the result back to this chat.
4. **Fallback only:** if none of these exist, start a background **sub-agent** instead (for example in CLI agents, which have no sub-chats). Say once in this chat that sub-chats are unavailable and you are using sub-agents.

Never pick a sub-agent because it is faster or simpler. A tool that is missing or returns an error is the only reason to fall back.

**Models.** These hold on a sub-chat and a sub-agent alike, and override any host guidance to start sessions or agents on a cheaper model:

- **Tasks run on this chat's model or a more capable one, never a smaller one.** Leave the model unset when you start the worker, so it inherits this chat's model (for example, pass no `model` to `start_session` or to the Agent tool). If the host would start it on a smaller default, set the most capable model it offers instead (in Claude, Opus). Keep this chat's reasoning effort or higher, never lower. A task changes code, and a weaker model's mistakes cost the user more than the time saved.
- **Asks run on the fastest model available** (a fast or flash model, not a thinking model), so the answer is spoken quickly.

Keep related work together. Keep a short map in this chat of topic → sub-chat (or sub-agent), its model, and the requests it handled. When a new request is about the same feature, screen, or files as an existing sub-chat's work, send it to that sub-chat, so the context is reused. Start a new sub-chat only for a new topic, and for a task whose topic so far ran only on a fast ask model: a task never goes to a fast-model worker.

Brief each worker with the `requestId`, `kind`, `text`, `intent`, the `contextFiles` paths if any, the `discussion` summary and path if any, and these rules:

- If `needsRecording` is true or missing, call `get_agent_request` first. It returns the frame list with each frame's on-screen text, but only one image: the fresh frame, the screen right now. Then look at the snapshot frames themselves, as you would a normal recording: call `get_frames_near` with the `requestId` (not `sessionId`) and `limit: 21`, and read the PNGs it lists (`framesDir` + `fileName`) in parallel. Page through with `aroundIndex` until you have seen the whole snapshot, or at least the stretch the request is about. The on-screen text is OCR: use it to find frames, not in place of looking at them, since layout, color, spacing and state only show in the image. The snapshot covers the stretch where the user's words (`anchors.spokenQuotes`) and the on-screen text (`anchors.screenTexts`) matched the recording, from about 10 seconds before the first hit to about 5 seconds after the last. Look hardest at the frames with `matchedScreenTexts` and the frames around the matched speech in `anchors.hits`. If `anchors.source` is `fallback`, nothing matched and the window is the last 90 seconds before the request. `get_agent_request_frame` still fetches one frame by index. If `needsRecording` is false, work from `text` and `intent` only: there is no snapshot, so do not call `get_agent_request`, `get_frames_near` or `get_agent_request_frame`, and do not pass any recording to the sub-chat.
- **The user's marks come first.** If `get_agent_request` returns `markedFrames`, the user drew on the screen (an underline, circle, box, arrow, or strike-through) and `userMarksNote` says how to read it. Open those frames before any others and work out exactly what the mark covers: which words, which line, which element. That is the target. It outranks `text`, `intent`, and the user's words whenever they are vaguer or disagree, and it limits the change: if only one sentence of a paragraph is underlined, change only that sentence. Underlined or circled text is what to change; struck-through text is what to remove. When `text` quotes marked text, find that exact text in the code. A mark can also appear in frames without `marked`, so look for ink in the frames around them too. In the reply, say what the change applied to in the user's terms, like "Removed the underlined first sentence of the footer."
- If the item has `contextFiles`, read the ones the request is about from their `path`: images, PDFs, code, docs, and snippets are what the user attached for this work. When `text` names an attachment ("the PDF I added"), that file is the source of truth for it. Skim the others only if they may help. A folder's path is the user's own folder: read it, never edit it unless the task asks.
- **Ask:** research in depth (Live sends the questions it cannot answer well itself). Read the code, and look up docs or the web when the answer depends on something outside the code or newer than a model's knowledge cutoff. May suggest code, must not edit files.
- If the item has `discussion`, read `discussion.summary` first, and `discussion.path` when the summary leaves a detail unclear. The user's own words are the source of truth for the details: when they add to or differ from `text`, follow the latest thing the user said. The earliest lines may be about something else; skip them.
- **Task:** implement the change. Trust `intent` for what the user wants, and use the snapshot to find where and how.
- Never run `/annotate-session-start`, register, poll, or call `reply_agent_request`. A sub-chat has the same Annotate MCP tools, so say this in its brief. Return the result to this chat: for an ask, an answer short enough to be spoken; for a task, one short sentence Live can speak, or why it failed.

## Replying to Live: immediately

The user is waiting on a voice call, so a finished result must reach Live within seconds. Relaying it comes before anything else, including the next poll.

- **Reply the moment a result is ready.** As soon as a worker's result comes back, call `reply_agent_request` for its `requestId`: `done` with the result text, or `failed` with the reason. Do not batch results, do not wait for other workers, and do not rewrite or summarize the result first. Only trim it so it can be spoken.
- **Short polls while work is out.** A long `poll_agent_session` blocks this chat, so it cannot see a finished sub-chat until the poll returns. While any request is still with a worker, call `poll_agent_session` with `timeoutMs: 5000`. After each poll, check every outstanding worker for a finished result and reply for each one that is done, then poll again.
  - Claude sub-chats: check the sessions you started, for example with `list_sessions` (`linked: true`) or `get_session`, and read each finished session's last report.
  - Codex chats: read the thread's latest reply.
  - Sub-agents: their completion notice.
- **Back to the normal poll** (SKILL.md step 9) only when no request is outstanding.
- Keep a short list in this chat of outstanding `requestId` → worker, so nothing is forgotten. A request leaves the list only once you have replied `done` or `failed` for it.

Live tells the user as soon as they are free to listen, and keeps the result as context for the rest of the conversation.

Keep the chat title `Annotate Live`. Do not rename it per request; name sub-chats after their topic instead.
