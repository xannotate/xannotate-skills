# Handoff coordinator (Parallel tasks)

Read this when a recording comes with **mode coordinator**:

- the Session hand off prompt has `Handoff mode: coordinator`, or
- a `poll_agent_session` item, a `handoff.ready` item, or a `recordingModes` entry from `check_agent_session` / `register_agent_session` says `mode: "coordinator"`.

The user turned on **Parallel tasks** in Annotate. This chat is still the listener from [SKILL.md](SKILL.md), but it does not do the recordings itself. It is the coordinator: it hands each recording to a sub-chat and goes straight back to listening, so the user can hand off the next recording while earlier ones still run. Everything in SKILL.md still holds (register, title, the listening loop and its exits, rule 8 feedback, Stop), except steps 5–7, which the workers do.

The mode belongs to each recording, not to the chat. A recording with `mode: "direct"`, or no mode, is done in this chat as SKILL.md steps 5–8 say, even while sub-chats run. The user turned Parallel tasks off for it.

## Register

As SKILL.md steps 1–3. On the remote MCP also pass `handoffMode: "coordinator"` to `register_agent_session` with the prompt's `recordingSessionId`, so the first recording is handed over too. Then take the prompt's recording like any other coordinator recording (below), instead of reading it here.

## Each coordinator recording

For every recording with mode coordinator, all at once, without waiting for one to finish before taking the next:

1. **Ack it:** `set_recording_status` with `status: "in_progress"`. On the local MCP an unacked recording is handed over again after about a minute; on the remote MCP it keeps coming back in `inFlightRecordingIds` and the subscribe replay. Once acked it shows as in progress on this chat's chip in Annotate.
2. **Triage only:** `get_manifest_summary` for the recording, to learn the repo (on-screen paths, editor titles, dropped folders, speech) and the topic. Do not read frames, the timeline, or code here, and do not plan the change. Never read a recording that is not completed.
3. **Pick the worker** as [workers.md](workers.md) says: the sub-chat that already worked on the same repo and feature, or a new sub-chat started in that repo and named after the task ([workers.md](workers.md#name-the-sub-chat-after-its-task)), never `Annotate recording <time> handoff`. Code-changing work never goes to a smaller model.
4. **Brief it** with:
   - The recording id and which MCP to use: "Use the local Annotate MCP (annotate-session)" or "Use the Annotate remote MCP (annotate-session-remote)", the same as the prompt that reached this chat.
   - "Do the work in this recording: read it MCP-first (`get_timeline` first, then `search_session`, `get_frames_near`, `get_transcript_range`, `list_context` as needed, and `get_session_guide`), work in the repo it is about, and make the change it asks for."
   - The rules every brief carries ([workers.md](workers.md#every-brief-says)), including: "As soon as you have read the recording and know what it asks for, rename this chat to a short title of that intent."
   - "End with: what you did, what the recording asked for, how your work answers it, anything left for the user, and a short title phrase of the change (spaces, no dashes)."
5. **Listen again** right away. Do not answer or implement anything in this chat.

Keep a short list in this chat: recording id → worker, and the topic map from workers.md. On the remote MCP the done file still lists finished ids; an id goes in it when its worker finishes, not when it is handed over.

## While workers run

- **Local MCP:** call `poll_agent_session` with `timeoutMs: 5000` while any recording is with a worker. After each poll, take any new recordings, then check every outstanding worker as [workers.md](workers.md#seeing-a-finished-worker) says. Back to the normal long poll (SKILL.md step 9) only when no worker is out.
- **Remote MCP:** `runningRecordingIds` from `check_agent_session` are with workers and do not block the subscribe. Hand over any coordinator id in `inFlightRecordingIds` or `queuedRecordingIds` that this chat has not handed over yet, and do any direct ones. Then subscribe with `timeoutSec: 15` while any worker is out (the normal wait when none is), then check every outstanding worker. A `handoff.ready` with `redelivered: true` repeats ids this chat may already have handed over: skip those that are on the list.

## When a worker finishes

Relay it the moment you see it, before the next poll or subscribe:

1. `set_recording_status` with `status: "done"`, or `"failed"` with the reason if the worker could not do it. A recording you stopped on purpose (the user asked, or a newer recording replaced it) is `failed` too.
2. `set_agent_session_title` with the worker's title phrase. The chip shows the latest finished change. If that phrase reads clearly better than the sub-chat's current name, rename the sub-chat too ([workers.md](workers.md#name-the-sub-chat-after-its-task), step 3).
3. The rule 8 feedback in this chat: what was done, what the recording asked for, how it answers it. Say which sub-chat did it, so the user can open it.
4. On the remote MCP, append the id to the done file.
5. Back to the loop: poll (short while other workers are out) or check and subscribe.

If a worker is waiting on the user (a question, an approval), say so in the feedback and name the sub-chat, keep its recording `in_progress`, and keep listening. Report it done once the worker finishes.

## Resume after a stop

If this chat stopped and the user runs `/annotate-session-start` again here, register again as SKILL.md says (remote: with `resumeSessionId`). Then read your list. For a recording you had handed over, check its worker before handing it over again: a finished one is relayed as above, a running one stays with it, and only a recording with no worker goes to a new one. On the remote MCP, `runningRecordingIds` lists the recordings Annotate still counts as in progress.
