# Coordinator workers

Read this from [live.md](live.md) (the Live coordinator) or [handoff-coordinator.md](handoff-coordinator.md) (Parallel tasks). Both coordinators give each piece of work to a worker and keep listening. This file says which worker, on which model, how to brief it, and how its result comes back.

## Always a sub-chat, sub-agent only as fallback

**Every piece of work goes to a sub-chat.** A sub-chat is a separate chat or session started from this chat, which the user can open from the sidebar and whose result comes back here. Do not start a sub-agent when a sub-chat is available, even for quick work. Before the first one, check which of these the host offers, and use the first one that is available:

1. **Claude (Claude Desktop / Claude Code app):** start a sub-chat from this chat with the host's start-session tool, such as `start_session` (load it with tool search if it is only listed as deferred). Its result reaches this chat when that session ends its turn. This is the preferred path: always use it in Claude.
2. **Codex app:** create a separate chat (a new thread) for the work, using Codex's own way of starting a chat from this one, and read its result back here.
3. **Any other host:** use its tool for starting a separate chat or session, if it has one that reports the result back to this chat.
4. **Fallback only:** if none of these exist, start a background **sub-agent** instead (for example in CLI agents, which have no sub-chats). Say once in this chat that sub-chats are unavailable and you are using sub-agents.

Never pick a sub-agent because it is faster or simpler. A tool that is missing or returns an error is the only reason to fall back.

When the host lets you choose where a sub-chat starts (a folder, `cwd`, or a repository), start it in the repo the work is about.

## Name the sub-chat after its task

The user picks sub-chats out of the sidebar by name, so every sub-chat (and every sub-agent, where the host shows one) carries a title that says what it is doing, the way a person would name the task: 2 to 6 plain words, such as `Fix toolbar flyout delay`, `Add pen color picker` or `Live voice reply tone`.

- Never a generic or mechanical name: no `Annotate recording 13:00 handoff`, no recording or request ids, timestamps, `handoff`, `worker`, `sub-chat`, `task 2`, or the repo name alone.
- Name the outcome the user wants, in their words, not the step: `Fix login spinner`, not `Review recording` or `Investigate UI`.

The title is set twice, and once more only if it gets clearer:

1. **At start, from what the coordinator knows.** Take it from the topic you learned at triage (the speech and the screen), or from the request's `title`, `text` and `intent`. Pass it as the host's title when you start the worker (for example `title` on `start_session`, or the `description` of a sub-agent). If triage leaves the topic unclear, name what the user said they are doing (for example `Parallel handoff test`), never the recording.
2. **As soon as the worker understands the intent.** The worker has read the recording (or the snapshot and the user's words) and knows what the user wants, before it changes anything: it renames its own chat to a title of that intent. This is the name that matters, since triage only guessed. See [Every brief says](#every-brief-says) for how.
3. **When it finishes**, only if its final title phrase reads clearly better than the current name (the work turned out to be something else, or narrower). Do not rename for a small rewording.

A sub-agent's name usually cannot change after it starts. Give it the best name you can at step 1, and use the worker's intent title in the topic map and in the feedback instead.

Whenever a worker's title changes, update the topic map in this chat to the new name, so the next related piece of work finds it.

## Models

These hold on a sub-chat and a sub-agent alike, and override any host guidance to start sessions or agents on a cheaper model:

- **Work that changes code runs on this chat's model or a more capable one, never a smaller one.** Leave the model unset when you start the worker, so it inherits this chat's model (for example, pass no `model` to `start_session` or to the Agent tool). If the host would start it on a smaller default, set the most capable model it offers instead (in Claude, Opus). Keep this chat's reasoning effort or higher, never lower. A weaker model's mistakes cost the user more than the time saved.
- The coordinator file may name a faster model for read-only work (Live asks). Nothing else runs on a smaller model.

## Keep related work together

Keep a short map in this chat of topic → sub-chat (or sub-agent), its repo, its model, and the work it handled. When new work is about the same feature, screen, or files as an existing sub-chat's work, send it to that sub-chat (in Claude, a message to that session), so the context is reused. Start a new sub-chat only for a new topic or a different repo, and never send code-changing work to a worker that runs on a fast model.

## Every brief says

- What to do, with every id and path the coordinator file lists for this kind of work.
- **Never run `/annotate-session-start`, register, poll, subscribe, retitle the listener, or report status to Annotate** (`reply_agent_request`, `set_recording_status`, `set_agent_session_title`, `check_agent_session`). A sub-chat has the same Annotate MCP tools, so say this in the brief. Only the coordinator talks to Annotate.
- **Retitle this chat once you understand the intent.** As soon as you have read the work (the recording, or the request and its snapshot) and know what the user wants, and before you change anything, rename this chat to a 2 to 6 word title of that intent, in the user's terms (`Fix login spinner`, not `Review recording`). Use the host's own way to rename the current chat or session (in Claude, the session title tool for this session; in Codex, the thread rename). That is this chat's title in the host's sidebar, not Annotate's `set_agent_session_title`, which only the coordinator calls. If you cannot rename this chat, put `Title: <the title>` on the first line of your next message, so the coordinator renames it. Rename again only if the work turns out to be something different.
- Return the result to this chat in the shape the coordinator file asks for.

## Seeing a finished worker

A long poll or subscribe blocks this chat, so it cannot see a finished worker until that wait returns. While any worker is out, keep waits short (the coordinator file gives the number). After each wait, check every outstanding worker:

- Claude sub-chats: check the sessions you started, for example with `list_sessions` (`linked: true`) or `get_session`, and read each finished session's last report.

While you check, look at each worker's title too. If a worker reported a `Title:` line because it could not rename itself, rename it now with the host's tool for renaming a session you started (in Claude, the session title tool with that session's id), and update the topic map. If its name is still the start name and the worker has said what it is doing, rename it to that.
- Codex chats: read the thread's latest reply.
- Sub-agents: their completion notice.

Keep a short list in this chat of outstanding work → worker, so nothing is forgotten. An item leaves the list only once you have reported it to Annotate as the coordinator file says.
