# xannotate-skills

Public [skills.sh](https://skills.sh) package for Annotate recording handoff.

## Skill

- `annotate-session-start` — `/annotate-session-start` runs the latest completed session immediately, then keeps polling via `wait_for_handoff` (1-second MCP poll) and does not end the turn while auto hand off is on.

## Install

Install **globally** so every project can pick up recordings:

```bash
npx skills add xannotate/xannotate-skills --skill annotate-session-start -g
```

Non-interactive:

```bash
npx skills add xannotate/xannotate-skills --skill annotate-session-start -g -y
```

Then run `/annotate-session-start` (or paste the Auto hand off prompt from the Annotate session-saved popup).

## What it includes

- [`annotate-session-start/SKILL.md`](annotate-session-start/SKILL.md)
- [`annotate-session-start/reference.md`](annotate-session-start/reference.md)

Requires [Annotate MCP](https://xannotate.com/mcp/install.md) (`annotate-session`).

## Publish

1. Push to `https://github.com/xannotate/xannotate-skills.git`.
2. Install once with the command above.
3. skills.sh lists the skill from install telemetry.

Docs: https://skills.sh/docs
