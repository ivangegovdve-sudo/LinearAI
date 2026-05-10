# web2home-handoff-04 — Folder Structure, Naming, Rules, Activations

> Deterministic filesystem layout for the multi-LLM workspace. Every artefact
> in handoffs 01–03/05/06 lands in a predictable place. Naming is convention
> over configuration so the synthesiser can glob safely.

---

## 1. Content / Intent

A single source of truth for **where things live** and **how they are named**.
Without this, parallel subagents step on each other and weekly cron runs
silently shadow earlier outputs. The layout below is the output of cumulative
patterns observed in the canonical Claude Code docs (`.claude/`, `~/.claude/`,
plugin & skill directories) plus mempalace-style memory partitioning.

## 2. Use Case

| Audience | Reads / writes |
|---|---|
| Orchestrator (handoff-01) | mkdir -p tree, writes to dropzone |
| Workers (handoff-02) | write `~/web2home/research/<slug>.md` only |
| Synthesiser (handoff-03) | reads dropzone, writes `~/web2home/synthesis/` |
| Skill/cron emitter (handoff-05) | writes `~/.claude/{skills,agents,hooks,commands}/` |
| User | navigates by glob; never has to remember a path |
| Cross-platform exporter | reads GENESIS, writes `~/web2home/_portable/` |

## 3. Method of Implementation

### 3.1 Top-level layout

```
~/                                            (HOME)
├── .claude/                                  (USER-SCOPED CLAUDE)
│   ├── CLAUDE.md                             (global persona; imports GENESIS)
│   ├── settings.json                         (hooks, perms, env)
│   ├── settings.local.json                   (machine-specific overrides; gitignored)
│   ├── skills/                               (one dir per SKILL.md)
│   │   ├── skl-cached-prompt-runner/
│   │   ├── skl-mempalace-recall/
│   │   └── …
│   ├── agents/                               (subagent definitions)
│   │   ├── agt-research-worker.md
│   │   ├── agt-synthesiser.md
│   │   └── …
│   ├── commands/                             (slash commands)
│   │   ├── web2home-refresh.md
│   │   ├── research.md
│   │   └── replan.md
│   ├── hooks/                                (referenced from settings.json)
│   │   ├── hk-pre-compact-handoff.sh
│   │   ├── hk-stop-summary.sh
│   │   └── hk-session-start-load.sh
│   ├── mcp/                                  (mcp server configs)
│   │   ├── mcp-desktop-commander.json
│   │   ├── mcp-memgraph.json
│   │   └── mcp-fs-readonly.json
│   ├── cron/                                 (scheduled-tasks specs; see handoff-05)
│   │   ├── cron-research-refresh.md
│   │   ├── cron-self-review.md
│   │   └── cron-memory-dedupe.md
│   ├── routines/                             (cloud Routines manifests)
│   │   └── rt-weekly-genesis.yaml
│   ├── memory/                               (longterm; see handoff-06)
│   │   ├── flat/                             (markdown snippets)
│   │   ├── vector/                           (embeddings store handle)
│   │   ├── graph/                            (memgraph or sqlite)
│   │   ├── _index.json
│   │   └── retention-policy.md
│   └── plugins/                              (installed marketplace plugins)
│
├── web2home/                                 (USER-CONVENIENT DROPZONE)
│   ├── research/                             (handoff-02 outputs, one per repo)
│   ├── synthesis/                            (handoff-03 outputs)
│   ├── _portable/                            (GPT / Gemini exports)
│   ├── handoff/                              (PreCompact handoffs; see handoff-06)
│   │   ├── ho-2026-05-10T13-37-22Z.md
│   │   └── ho-current.md (symlink to most recent)
│   ├── inbox/                                (one-shot pastes / ad hoc)
│   ├── outbox/                               (artefacts ready for human review)
│   └── .telemetry/                           (CSV / JSON metrics)
│
└── (repo-specific projects ...)
    └── <repo>/
        ├── .claude/                          (PROJECT-SCOPED CLAUDE; checked in)
        │   ├── CLAUDE.md                     (project persona; imports root)
        │   ├── settings.json                 (project hooks, allowed tools)
        │   ├── skills/                       (project-only skills)
        │   ├── agents/                       (project-only subagents)
        │   ├── commands/
        │   ├── rules/                        (project rules, see §3.5)
        │   └── _portable/                    (exportable artefacts)
        ├── CLAUDE.local.md                   (personal; gitignored)
        └── AGENTS.md                         (legacy; kept for compatibility)
```

### 3.2 Naming conventions

| Domain | Prefix | Example |
|---|---|---|
| Skills | `skl-` | `skl-mempalace-recall` |
| Agents (subagents) | `agt-` | `agt-research-worker.md` |
| Hooks | `hk-` | `hk-pre-compact-handoff.sh` |
| Cron specs | `cron-` | `cron-research-refresh.md` |
| Routines (cloud) | `rt-` | `rt-weekly-genesis.yaml` |
| MCP configs | `mcp-` | `mcp-memgraph.json` |
| Slash commands | (no prefix; user-typed) | `web2home-refresh.md` |
| Handoff snapshots | `ho-` | `ho-2026-05-10T13-37-22Z.md` |
| Memory shards | `mem-` | `mem-2026-q2-prompts.md` |
| Telemetry runs | `run-` | `run-2026-05-10.json` |

Filenames are **kebab-case**, ASCII only, **no spaces**, **no emoji**. ISO-8601
in UTC for any time-bearing name. No version suffixes — use git for history.

### 3.3 Frontmatter schema (every artefact)

```yaml
---
name: <kebab-name>
kind: skill|agent|hook|cron|routine|mcp|command|memory|handoff
description: <≤200 chars; first paragraph used as model-discovery hint>
version: <semver>
last_updated: <ISO-8601>
trigger_keywords: [..]                # for skills only
args: <free-form usage hint>          # for skills/commands
lazy_imports: [@./ref-card.md]        # never auto-loaded; pulled on demand
disable_model_invocation: false       # true for explicit-only workflows
source_repo: <if derived from research>
target_platform: [claude, gpt, gemini]
efficiency_score: <0..100>
score_rubric_version: v1
---
```

Required fields: `name`, `kind`, `description`, `version`, `last_updated`,
`efficiency_score`, `score_rubric_version`. Missing required fields cause the
synthesiser to reject the artefact and queue a regen.

### 3.4 Activation patterns

* **Skills**: triggered by `description` model-match OR explicit `/skl-*` OR
  `trigger_keywords` matched in the user prompt (case-insensitive, word-bound).
* **Project skills**: only loaded when CWD is inside the project tree.
* **User skills**: always candidate-loaded; only the matched ones consume tokens.
* **Hooks**: declared in `~/.claude/settings.json` per Claude Code hook events
  (`PreCompact`, `Stop`, `SessionStart`, `PreToolUse`, `PostToolUse`).
* **Cron**: handled inside Claude Code with `/loop` + scheduled-tasks; OS-level
  cron only as a last resort and only to invoke `claude -p`.
* **Routines**: cloud-managed; manifest in `~/.claude/routines/*.yaml`.

### 3.5 Rules / restrictions matrix

| Rule | Where enforced | Mechanism |
|---|---|---|
| No write outside `~/web2home/` and `~/.claude/` during research | hook `PreToolUse` (Write/Edit) | regex deny |
| No external network from workers except github.com & code.claude.com | hook `PreToolUse` (WebFetch/Bash) | URL allowlist |
| No secrets in any file | hook `PreToolUse` (Write) | regex scan for `[A-Z0-9_]{16,}=` |
| Max output 800 lines for workers | inside worker prompt | self-policed + hook `PostToolUse` |
| PreCompact handoff mandatory | hook `PreCompact` | runs `hk-pre-compact-handoff.sh` |
| Skill list capped per session at 30 | hook `SessionStart` | warns + suggests pruning |
| Stale memory shard (>180 days) | cron `cron-memory-dedupe` | weekly compaction |
| Skill version drift (>30 days) | cron `cron-self-review` | proposes Edit |

### 3.6 Project vs. user split

| Where | What lives there |
|---|---|
| `~/.claude/` | Anything that should follow you across machines / projects (research workspace, longterm memory, primary persona, generic skills, cron). |
| `<project>/.claude/` | Project-specific personas, scripts, project-only skills, project hooks, project rules. Checked into git. |
| `<project>/CLAUDE.local.md` | Personal/project notes never committed (e.g., test creds expectations). |

### 3.7 Cross-platform mirror (`_portable/`)

`~/web2home/_portable/` is the **export pivot**. Anything inside is generated,
never authored:

| File | Consumer |
|---|---|
| `system-prompt.txt` | Custom GPT *Instructions*; Gemini Gem instructions |
| `memory.json` | uploaded to Custom GPT knowledge; attached to Gem as Connected Doc |
| `actions/openapi.yaml` | Custom GPT Actions (one per MCP-as-REST shim) |
| `skills.zip` | bundle of SKILL.md, README, plus a manifest |
| `bridge-map.json` | which feature on which platform; written by handoff-03 |

## 4. Synchronisation

* `~/.claude/` is a *git-tracked* repo with `.gitignore` excluding `*.local.*`,
  `memory/*/raw/`, `.telemetry/`. Push to a private remote → identical setup
  on any machine in <2 min.
* `~/web2home/` is **not** version-controlled (it churns). Backed up via a
  `cron-backup-web2home` skill that snapshots to `~/.claude/snapshots/` weekly.
* The pivot `_portable/` is regenerated each synthesis; the previous version
  is moved to `_portable/archive/<isodate>/`.

## 5. Automation

| Trigger | Effect |
|---|---|
| `claude /web2home-init` | Creates the entire tree with `mkdir -p` and seed files |
| `claude /web2home-refresh` | Runs the orchestrator |
| `cron-self-review` weekly | Audits naming and frontmatter, opens PR with fixes |
| Hook `SessionStart` | Loads `~/.claude/CLAUDE.md`, prints a 5-line workspace status |
| Hook `PreToolUse(Write)` | Rejects writes that violate naming convention |

## 6. Self-Improvement Technique

The naming convention itself is auditable: `cron-self-review` runs a glob
against every artefact, validates frontmatter, and opens a `proposed-patches/`
PR if drift is detected. Three rejected proposals in a row escalate to the
user with a single line in `~/web2home/handoff/ho-current.md`.

The synthesiser tracks "naming churn" (rate of rename events) — high churn
indicates an unstable convention; if rate >5/week, the synthesiser proposes
a convention amendment in `decision-matrix.md`.

## 7. Robust Efficiency Score (this artefact)

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 22 | Convention is short to load; references rather than duplicates |
| Latency | 15 | 13 | Glob lookups are O(1) once tree is materialised |
| Reuse | 20 | 19 | Convention is stable across projects/machines |
| Self-improve loop closed | 20 | 16 | Audit + proposed-patches is wired; convention amendment path exists |
| Cross-platform sync | 10 | 9 | `_portable/` is the explicit pivot |
| Automation | 10 | 9 | Init, refresh, audit, hook coverage |
| **Total** | **100** | **88** | Add a json-schema validator in CI to push to 92 |

## 8. Cross-References

* Claude Code docs: `.claude/`, settings, skills, hooks, scheduled-tasks, routines.
* `web2home-handoff-01-orchestrator.md` §3.3 — dropzone shape mirrors §3.1.
* `web2home-handoff-05-skills-and-automation.md` — templates that target this layout.
* `web2home-handoff-06-handoff-compression.md` §3 — `handoff/` directory contract.
