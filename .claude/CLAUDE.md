# LinearAI — project CLAUDE.md

> Lightweight project persona. Imports the global persona below; do not
> duplicate global rules here.

@~/.claude/CLAUDE.md

## Project intent
Linear-oriented tooling and experiments. Free-tier-friendly workflows.

## Local conventions
- Prefer Node 20+ with TypeScript or Python 3.11+; no large framework until
  the first concrete workflow is defined.
- Expected secret: `LINEAR_API_KEY` — never write it to a tracked file.
- Branch protocol: all work goes on `claude/<topic>-<short-rand>`;
  PRs open as draft.

## Hand-off
- The `web2home/` directory contains the canonical handoff suite
  (handoff-01..06). When in doubt, read
  `web2home/web2home-handoff-01-orchestrator.md` §8.
- Swarm dropzone: `$HOME/web2home` (research, synthesis, handoff, _portable,
  inbox, .telemetry). Configurable via `DROPZONE` in `~/.claude/settings.json`.

## Activations
- Skills in `.claude/skills/` apply only inside this tree.
- Hooks declared globally in `~/.claude/settings.json`; project-only hooks
  may be added later in `.claude/settings.json` if needed.
