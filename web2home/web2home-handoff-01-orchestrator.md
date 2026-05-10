# web2home-handoff-01 — Orchestrator (Swarm-Spawning Master Prompt)

> Entry point for the suite. Paste the **PROMPT** block at the bottom of this file
> into a fresh Claude Code session. It will fan out a research swarm across 50+
> repos, synthesise findings, and emit deterministic markdown deliverables under
> `~/web2home/research/`.

---

## 1. Content / Intent

A self-contained orchestration prompt that turns one Claude Code session into a
controller for a parallel research swarm. The controller:

1. Reads the Claude Code documentation index (`https://code.claude.com/docs/llms.txt`)
   plus the **Best Practices** and **Skills / Subagents / Hooks / MCP / Scheduled
   Tasks / Routines / Channels / Memory** pages.
2. Spawns one **research worker** per target repository using the template in
   `web2home-handoff-02-research-worker.md` (parameterised, identical per worker).
3. Streams worker outputs into the dropzone `~/web2home/research/<repo-slug>.md`.
4. Triggers the **Synthesis & Brainstorm** phase from
   `web2home-handoff-03-synthesis-brainstorm.md`.
5. Emits the final plan (`GENESIS-PLAN.md`) and updates ~/.claude memory.
6. Closes a self-improvement loop: scores the run, patches its own prompt, commits.

## 2. Use Case

| Scenario | Why this prompt |
|---|---|
| Bootstrapping a Claude-Max-centred multi-LLM workspace | Single-shot, deterministic, idempotent |
| Continuous repo-watch (cron-driven) | Re-running picks up only changed repos via SHA cache |
| Onboarding a new machine | Re-emits `~/.claude/CLAUDE.md`, skills, hooks, cron |
| Cross-platform sync (Claude / GPT Plus / Gemini Pro) | Produces portable artefacts in `_portable/` |

## 3. Method of Implementation

### 3.1 Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│ ORCHESTRATOR (this prompt, foreground main session)                │
│                                                                    │
│  Phase 0 — Bootstrap (read llms.txt + best-practices)              │
│  Phase 1 — Plan (TodoWrite, /context, budget)                      │
│  Phase 2 — Spawn N workers (Agent tool, batches of 5)              │
│  Phase 3 — Aggregate worker outputs into ~/web2home/research/      │
│  Phase 4 — Synthesis subagent → GENESIS-PLAN.md                    │
│  Phase 5 — Skill/Hook/Cron emission → ~/.claude/{skills,hooks,...} │
│  Phase 6 — Self-improvement loop (score, critique, patch)          │
└────────────────────────────────────────────────────────────────────┘
        │           │           │           │           │
   ┌────▼───┐  ┌───▼───┐   ┌───▼───┐   ┌───▼───┐   ┌───▼───┐
   │ Worker │  │Worker │   │Worker │ … │Worker │   │Synth. │
   │ repo01 │  │repo02 │   │repo03 │   │repoNN │   │ agent │
   └────────┘  └───────┘   └───────┘   └───────┘   └───────┘
```

### 3.2 Concurrency policy

* **Batch size:** 5 workers per Agent dispatch. Run all 5 in a single message
  (parallel tool calls). After the batch returns, fire the next batch.
* **Why 5:** balances main-context bloat (each worker returns ~600–1200 tokens of
  summary) and wall-clock latency. Tune via `BATCH_SIZE` env in the prompt.
* **Backoff:** if any worker errors, retry once with a stricter focus block; on
  second failure, mark `status: failed` in the dropzone and continue.
* **Isolation:** never inline worker output back into the main thread beyond a
  3-line confirmation; the dropzone file is the source of truth.

### 3.3 Dropzone shape

```
~/web2home/
├── research/
│   ├── _index.json              # repo → file → status → score → SHA
│   ├── _failures.log
│   ├── agency-agents.md
│   ├── ruflo.md
│   └── … (one per repo)
├── synthesis/
│   ├── GENESIS-PLAN.md
│   ├── decision-matrix.md
│   ├── skill-backlog.md
│   ├── cron-backlog.md
│   └── multi-llm-bridges.md
├── _portable/                   # GPT-Plus / Gemini-Pro friendly exports
│   ├── system-prompt.txt
│   ├── memory.json
│   └── skills.zip
└── .telemetry/
    ├── run-<isodate>.json
    └── score-history.csv
```

### 3.4 Quality gates

A worker output is **accepted** only if it satisfies:

- [ ] Frontmatter present, schema-valid (see handoff-02 §4).
- [ ] At least one `source:` link with file path + commit SHA.
- [ ] `efficiency_score` is an integer 0–100 with rubric reference.
- [ ] No bare claims: every "this repo does X" is followed by a `source:` line.
- [ ] No more than 800 markdown lines.

Failed outputs are quarantined to `_failures.log`; the worker is re-spawned with
a slimmer focus block (architecture only).

## 4. Synchronisation

* **State of truth:** `~/web2home/research/_index.json` (atomic writes).
* **Sync target 1 — Claude:** `~/.claude/CLAUDE.md` imports from
  `~/web2home/synthesis/GENESIS-PLAN.md` via `@~/web2home/synthesis/GENESIS-PLAN.md`.
* **Sync target 2 — GPT Plus:** `~/web2home/_portable/system-prompt.txt` is
  pasted into a Custom GPT *Instructions* field; `memory.json` is uploaded as a
  knowledge file; Actions are generated from MCP manifests.
* **Sync target 3 — Gemini Pro:** Same `system-prompt.txt` becomes a Gem;
  `memory.json` is attached as a connected Doc; extensions map to Gemini
  extensions where supported.
* **Cron:** `cron-research-refresh` (handoff-05 §3) re-runs the orchestrator
  weekly and diffs against `_index.json`.

## 5. Automation

| Trigger | Action |
|---|---|
| Manual (`/web2home-refresh`) | Full orchestrator run |
| Cron weekly (`Mon 03:17 UTC`) | Incremental refresh of changed-SHA repos only |
| Hook `Stop` | If `_failures.log` non-empty, re-spawn failed workers once |
| Hook `PreCompact` | Flush in-flight worker results to dropzone first |
| Routine (cloud) | Same orchestrator, but `--remote`, results PR'd to repo |

## 6. Self-Improvement Technique

After every run the orchestrator appends to `.telemetry/score-history.csv`:

```
run_id,timestamp,workers_spawned,failures,total_tokens,wallclock_s,
mean_worker_score,brainstorm_score,rubric_version
```

A `cron-self-review` skill (handoff-05 §3.4) opens the last 5 rows, asks Claude
to write a critique against the rubric, and proposes an Edit to *this file*
(handoff-01) plus handoff-02. Critiques that lower mean tokens by ≥5% or raise
mean score by ≥3 points without quality regression are auto-applied; otherwise
they're staged in `~/web2home/synthesis/proposed-patches/` for human review.

## 7. Robust Efficiency Score (this artefact)

Computed against the rubric in handoff-06 §6.

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 22 | Worker outputs capped at 800 lines, batches of 5, dropzone offload keeps main context lean |
| Latency | 15 | 12 | 50 repos / batch 5 ≈ 10 batches; bottleneck is per-worker WebFetch |
| Reuse | 20 | 19 | All workers share one template; one orchestrator survives weekly re-runs |
| Self-improve loop closed | 20 | 17 | Telemetry → critique → auto-patch is wired; gated thresholds prevent regressions |
| Cross-platform sync | 10 | 9 | `_portable/` exports map cleanly onto GPT Custom GPT + Gemini Gems |
| Automation | 10 | 9 | Cron + hooks + Routines all defined in handoff-05 |
| **Total** | **100** | **88** | Replace `WebFetch` with `gh api` for the auth-rate-limit win to push above 92 |

---

## 8. THE PROMPT (paste this block into a fresh Claude Code session)

> Replace nothing. The prompt is self-contained. The repo list is embedded.
> If you change the repo list, also update `~/web2home/.telemetry/repos.txt`.

````
# ROLE
You are the orchestrator of a Claude Code research swarm. Your job is not to
research repositories yourself; your job is to plan, dispatch workers, gather
their outputs, run a synthesis pass, and emit deterministic deliverables.
Stay light on context. Push reading into subagents. Push writing into files.

# OPERATING RULES
1. Read these docs first, in order, with WebFetch — DO NOT READ WHOLE PAGES,
   ask for the §s named:
     - https://code.claude.com/docs/llms.txt  (whole)
     - https://code.claude.com/docs/en/best-practices.md  (Verify, Plan, Context)
     - https://code.claude.com/docs/en/skills.md  (SKILL.md, frontmatter, args)
     - https://code.claude.com/docs/en/sub-agents.md  (definition + invocation)
     - https://code.claude.com/docs/en/hooks.md  (PreCompact, Stop, PreToolUse)
     - https://code.claude.com/docs/en/scheduled-tasks.md  (cron + /loop)
     - https://code.claude.com/docs/en/routines.md  (cloud)
     - https://code.claude.com/docs/en/channels.md  (push events)
     - https://code.claude.com/docs/en/memory.md  (CLAUDE.md, auto-memory)
     - https://code.claude.com/docs/en/checkpointing.md  (rewind)
     - https://code.claude.com/docs/en/context-window.md  (budget)
2. Create the dropzone tree (handoff-01 §3.3) with mkdir -p.
3. Use TodoWrite for every phase. Mark in_progress / completed in real time.
4. Spawn workers via the Agent tool, subagent_type=general-purpose, in batches
   of BATCH_SIZE=5 (parallel calls in ONE message per batch). Wait for each
   batch before firing the next.
5. Each worker receives the EXACT WORKER PROMPT from
   {{HANDOFF_DIR}}/web2home-handoff-02-research-worker.md, with
   {{REPO_URL}} substituted. It MUST be told to write its output to
   {{DROPZONE}}/research/<repo-slug>.md and reply with at most 6 lines:
   `slug:`, `score:`, `bytes:`, `sources:`, `status:`, `notes:`.
6. After each batch, atomically append worker results to
   {{DROPZONE}}/research/_index.json. Use Bash:
     `python -c '...build _index.json...' > _index.json.tmp \
        && mv _index.json.tmp _index.json`
   (the `mv` is the atomic step on POSIX; never write `_index.json` in place).
7. After ALL workers finish, spawn ONE synthesis subagent using the prompt
   from {{HANDOFF_DIR}}/web2home-handoff-03-synthesis-brainstorm.md. It writes
   {{DROPZONE}}/synthesis/GENESIS-PLAN.md and siblings.
8. After synthesis, emit:
     - ~/.claude/CLAUDE.md (or merge), importing GENESIS-PLAN.md
     - ~/.claude/skills/<name>/SKILL.md  (one per skill in skill-backlog.md)
     - ~/.claude/settings.json hooks block (PreCompact, Stop, SessionStart)
     - ~/.claude/commands/<slash>.md  (one per /command in the backlog)
     - {{DROPZONE}}/_portable/{system-prompt.txt, memory.json, skills.zip}
     - cron entries (or Routines manifest if --routine flag set)
9. Run the self-improvement loop: append a row to
   {{DROPZONE}}/.telemetry/score-history.csv with the schema in handoff-01 §6.
10. Stop. Report a 10-line summary: phases done, workers OK/failed, files
    written, total tokens (estimate), score, next-action recommendation.

# CONSTRAINTS
- NEVER read more than 200 lines of any single repo; that is the worker's job.
- NEVER paste worker output back into your main context beyond the 6-line
  receipt. The dropzone file is the source of truth.
- If main context > 60% during dispatch, /compact with the instruction
  "preserve repo list, dropzone state, current batch index".
- If WebFetch is rate-limited, switch to `gh api repos/{owner}/{repo}` (the
  GitHub MCP, when available) or queue with backoff.
- Idempotent: if {{DROPZONE}}/research/<slug>.md exists AND repo SHA matches the
  recorded SHA in _index.json, SKIP the worker. Always update SHA after a run.

# TARGET REPOSITORIES (full list, do not edit; 60 entries)
agency-agents,ruflo,activepieces,inngest,agency-swarm,PageIndex,
claude-cookbooks,context-mode,open-agents,memgraph,mercury-agent,
pydantic-ai,cookbook-cursor,knowledge-work-plugins,SWE-agent,OpenLLM,
awesome-openclaw-skills,hyperframes,remotion,design-extract,swarm,
autogen,AI-Context-OS,pi-mono,letta,claude-memory-compiler,deer-flow,
nanobot-webui,openfang,browser-ai,impeccable,DesktopCommanderMCP,
dreamgraph,autoresearch,paperclip,deepagents,HolyClaude,awesome-opencode,
NemoClaw,skills,PicoClaw,LoongFlow,claude-agent-sdk-demos,
motion-design-skill,julep,modern,AnyTool,agentic-flow,design-os,agent-os,
prompt-vault,edge-agents,adk_cloud_run,ui-inspector,awesome-crawler,
Retrieval-based-Voice-Conversion-WebUI,local-ai-mcp-chainlit
# (also: mempalace — used as the parity benchmark; see handoff-06 §3)

# ENVIRONMENT (substitute these BEFORE pasting into your session)
- BATCH_SIZE=5
- DROPZONE=$HOME/web2home                 # default; change if you want a different dropzone
- HANDOFF_DIR=<path-to-this-clone>/web2home
    # set to the absolute path of the directory containing this orchestrator file.
    # Examples:
    #   HANDOFF_DIR=$HOME/code/LinearAI/web2home
    #   HANDOFF_DIR=$(git rev-parse --show-toplevel)/web2home   # if pasting from inside the repo
    # The orchestrator MUST resolve {{HANDOFF_DIR}} and {{DROPZONE}} to absolute
    # paths at Phase 0 and use those in every subsequent file path.
- OWNER=ivangegovdve-sudo                 # for fork resolution; workers also resolve to upstream
- TIMEZONE=UTC
- DATE=2026-05-10

# GO
Begin Phase 0. Print "Phase 0: bootstrap" and start.
````

---

## 9. Operating Notes

* **DesktopCommanderMCP**: yes, it runs as a stand-alone MCP server (Node
  process). It does **not** require the Claude Desktop app. Any MCP-aware
  client (Claude Code CLI, Cursor, Continue, etc.) can attach to it via
  `claude mcp add desktop-commander npx -y @wonderwhy-er/desktop-commander`.
  The "Desktop" in its name refers to the *commands it executes on your
  desktop machine*, not the Claude Desktop application.
* **Repo URLs**: workers are told to first try the user's fork (`ivangegovdve-sudo/<name>`)
  and fall back to upstream (e.g., `OpenAI/swarm`) if the fork is bare. Where a
  fork has substantive divergence, the worker reports both.
* **Compression-safe**: this file is the *only* file the orchestrator must keep
  in context. Everything else is read just-in-time and offloaded to the dropzone.

## 10. Cross-References

* `web2home-handoff-02-research-worker.md` — the per-repo template each worker receives.
* `web2home-handoff-03-synthesis-brainstorm.md` — the synthesis subagent prompt.
* `web2home-handoff-04-folder-structure.md` — exact filesystem layout & naming.
* `web2home-handoff-05-skills-and-automation.md` — SKILL.md / hook / cron blueprints.
* `web2home-handoff-06-handoff-compression.md` — pre-compaction, memory, scoring.
