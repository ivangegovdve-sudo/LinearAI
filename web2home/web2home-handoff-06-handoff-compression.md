# web2home-handoff-06 — Pre-Compaction Handoff, Long-Term Memory, Multi-LLM Sync, Efficiency Rubric

> The "mempalace-grade" backbone. How context is preserved across compaction
> events, how memory ingests/retrieves, how Claude/GPT/Gemini stay in sync,
> and the efficiency score every artefact in this suite is graded against.

---

## 1. Content / Intent

Four pillars in one document because they are mutually load-bearing:

1. **Handoff** — when context is about to compact, snapshot the state into a
   token-cheap, reload-friendly file so a fresh session can pick up cleanly.
2. **Memory** — long-term store with cheap, fast, semantic retrieval; ingestion
   is automatic; retention is policy-driven; dedupe runs on cron.
3. **Multi-LLM sync** — exports keep Claude / GPT Plus / Gemini Pro on the same
   spec without the user manually copying anything more than once.
4. **Efficiency rubric** — the v1 100-point scoring framework used everywhere
   in handoffs 01–05. Concrete formulas, no hand-waving.

## 2. Use Case

| Trigger | Handoff path |
|---|---|
| Auto-compact about to fire (≥85% context) | `hk-pre-compact-handoff.sh` writes `ho-<ts>.md` |
| User runs `/clear` | Quick handoff: only "active todo + open files" |
| Session crash / reboot | Most recent `ho-current.md` rehydrates next session |
| Cross-LLM hand-off (Claude → GPT one-shot) | Pack `_portable/` + last 5 memory shards |
| End-of-day | Cron archives `ho-current.md` to `~/.claude/memory/flat/` |

## 3. Method of Implementation

### 3.1 Handoff file shape

```yaml
---
kind: handoff
name: ho-<ISO-8601-Z>
trigger: PreCompact|Stop|/clear|crash|cross-llm
session_id: <uuid>
context_tokens_before: <int>
context_tokens_after: <int>
git: { branch: <name>, head: <sha>, dirty: <bool> }
todos_open: <int>
files_open: <int>
last_user_intent: <≤80 chars>
next_actionable_step: <≤120 chars>
version: 1
score_rubric_version: v1
---
```

Body sections (each capped):

```
## Active todo (≤30 lines)        # mirror of TodoWrite state
## Open files (≤30 lines)         # paths + last edit timestamps
## Recent decisions (≤40 lines)   # tail of memory/flat/decisions.md
## Hot variables (≤20 lines)      # values mentioned 3+ times this session
## Bookmarks (≤10 lines)          # permalinks the user explicitly pinned
## Resume sentence (1 line)       # "Resume by: <verb> <object> from <file>:<line>"
```

A handoff file is **always under 200 lines**. The `Resume sentence` is the
single most important line: a fresh session reads only that on `SessionStart`.

### 3.2 Compression policy

* **Threshold:** at 70% context → orchestrator advised to `/compact` with
  preserve hint. At 85% → `PreCompact` hook fires unconditionally.
* **Preserve hint:** every `/compact` call includes
  `"preserve: active todos, open files, hot variables, last decision"`.
* **Discard policy:** worker outputs are never preserved in main context;
  they live in the dropzone.
* **Rehydration:** on `SessionStart`, `hk-session-start-load.sh` prints the
  Resume sentence and the count of open todos. The user (or model) decides
  whether to load the full handoff body via `@~/web2home/handoff/ho-current.md`.

### 3.3 Long-term memory

#### 3.3.1 Storage layers

| Layer | Path | Purpose |
|---|---|---|
| Flat shards | `~/.claude/memory/flat/mem-*.md` | human-readable canonical text |
| Vector | `~/.claude/memory/vector/<store>` | embeddings for semantic search |
| Graph | `~/.claude/memory/graph/` | entity-edge structure (Memgraph or sqlite) |
| Index | `~/.claude/memory/_index.json` | shard → vectors → graph nodes manifest |

Graph nodes carry `(:Entity {name, kind, last_seen})` and edges
`(:Decision)-[:ABOUT]->(:Entity)`. Shards reference graph node ids in
frontmatter so retrieval can hop both ways.

#### 3.3.2 Ingestion pipeline

```
sources                             pipeline                          stores
──────                              ──────                            ──────
- chat handoffs (ho-*.md)    ─┐
- synthesis outputs          ─┤
- inbox/ pastes              ─┤── chunk (700 tokens, 80 overlap)
- web2home/research/*.md     ─┤── tag  (rule + LLM)
- selected emails / docs     ─┘── embed (text-embedding-3-large or local)
                                  └── upsert vector
                                  └── extract entities → graph
                                  └── append shard markdown
```

Implemented by `skl-memory-ingest`; runs on:
- `Stop` hook (any new content in `inbox/`)
- `cron-memory-ingest` (handoff-05 §3.4)
- explicit `/skl-memory-ingest <path-or-url>`

#### 3.3.3 Retrieval policy (just-in-time)

`skl-mempalace-recall` (handoff-05 §3.1) runs the hybrid:
1. Vector top-8 (cosine ≥ 0.78).
2. Graph 1-hop expansion from any matched entity nodes.
3. Re-rank by `0.6 * similarity + 0.3 * recency_decay + 0.1 * cluster_overlap`.
4. Return top-5 with permalinks.

The retriever NEVER pastes raw shards back; it returns a summary + links.
Caller decides whether to load full text.

#### 3.3.4 Retention

| Age | Policy |
|---|---|
| 0–30 d | hot; never dedupe |
| 31–180 d | candidate for merge if cosine ≥ 0.92 with another shard |
| 181 d + | drop unless flagged `pin: true` in frontmatter |

`cron-memory-dedupe` (handoff-05 §3.4) executes the policy weekly.

### 3.4 Multi-LLM sync

#### 3.4.1 Source of truth

```
                ┌─────────────────────────────────┐
                │   ~/web2home/synthesis/         │
                │   GENESIS-PLAN.md  (canonical)  │
                └────────┬───────────┬────────────┘
                         │           │
                         ▼           ▼
   ~/.claude/CLAUDE.md           ~/web2home/_portable/
   (Claude Max)                   ├── system-prompt.txt
                                  ├── memory.json
                                  ├── actions/openapi.yaml
                                  └── skills.zip
                         │           │
                         ▼           ▼
                  Custom GPT      Gemini Gem
                  (GPT Plus)      (Gemini Pro)
```

#### 3.4.2 Export contract

`skl-platform-export` (a thin skill) generates `_portable/` from
`GENESIS-PLAN.md`:

| File | Source | Notes |
|---|---|---|
| `system-prompt.txt` | GENESIS §§ 1, 3, 6, 7, 8 | ≤8000 chars to fit Custom GPT field |
| `memory.json` | top-50 shards by retrieval frequency last 30 d | ≤512 KB |
| `actions/openapi.yaml` | MCP server manifests | one server → one Action |
| `skills.zip` | each `~/.claude/skills/*/SKILL.md` + reachable `lazy_imports` | flat layout |
| `bridge-map.json` | handoff-03 §3.6 multi-llm-bridges.md | machine-readable |

#### 3.4.3 Triggers

* On every GENESIS write → emit `_portable/` (synthesiser does this).
* Manual: `/skl-platform-export`.
* Cron weekly: validates that `_portable/` matches GENESIS hash; regenerates
  if not. Flags drift in `~/web2home/handoff/ho-current.md`.

#### 3.4.4 What does NOT sync (and why)

| Feature | Why not synced |
|---|---|
| Hooks | Only Claude Code has them; behaviour is local |
| Subagent infra | GPT Plus has no agent fan-out; Gemini's Gem-to-Gem is manual |
| Statusline | UI-only |
| `/loop` and Routines | Scheduling primitive is platform-specific |

For these we provide *equivalent advice* in `system-prompt.txt` instead of
trying to replicate behaviour: e.g., "If you need a sub-task done in
isolation, ask the user to delegate to a Custom GPT named X."

### 3.5 mempalace parity gap analysis

Target reference: `mempalace`-level efficiency. Concretely we aim for:

| Property | mempalace ref | this suite | gap |
|---|---|---|---|
| Compact, lossy-but-faithful long-term memory | yes | yes (flat+vector+graph) | parity |
| O(1) retrieval for hot queries | yes | yes (vector top-k) | parity |
| Provenance / source links | yes | yes (permalinks) | parity |
| Cron-driven dedupe | yes | yes (handoff-05 §3.4) | parity |
| Cross-LLM portable export | yes | yes (`_portable/`) | parity |
| Self-improving retrieval ranks | partial | partial (recency decay + cluster overlap) | tune |
| Native graph reasoning | yes (graph layer) | yes (memgraph option) | parity if memgraph configured |
| Token-efficient handoff on compaction | implicit | explicit (handoff-06 §3.1) | better |

Net: parity-or-better on 7/8 axes; tune retrieval ranking with telemetry-driven
weights from `cron-self-review`.

## 4. Synchronisation summary

* `~/.claude/` is git-tracked (private remote). Push = sync.
* `~/web2home/` is local-only and treated as ephemeral / regenerable.
* `_portable/` is the only external surface; recreate any time from GENESIS.
* Handoff files are pruned by retention; canonical decisions migrate to
  `~/.claude/memory/flat/decisions.md` so they survive forever.

## 5. Automation

| Job | Schedule | Spec |
|---|---|---|
| PreCompact handoff | event | `hk-pre-compact-handoff.sh` |
| Stop summary | event | `hk-stop-summary.sh` |
| SessionStart load | event | `hk-session-start-load.sh` |
| Memory ingest | event + 5 min poll on `inbox/` | `skl-memory-ingest` |
| Memory dedupe | weekly Sun 05:00 UTC | `cron-memory-dedupe` |
| Self-review | weekly Mon 04:00 UTC | `cron-self-review` |
| Research refresh | weekly Mon 03:17 UTC | `cron-research-refresh` |
| Genesis emit (cloud) | weekly Mon 03:17 UTC | `rt-weekly-genesis` (Routine) |
| Platform export | on GENESIS change | `skl-platform-export` |

## 6. Robust Efficiency Score — Rubric v1 (canonical)

Every artefact in handoffs 01–06 carries an `efficiency_score` against this
rubric. The rubric is intentionally short so it stays loadable.

```
total_score = token_efficiency      (25)
            + latency               (15)
            + reuse                 (20)
            + self_improve_closed   (20)
            + cross_platform_sync   (10)
            + automation            (10)
            = 100 max
```

Per-dimension formulas:

```
token_efficiency = 25 - clamp(0..25, ceil((tokens_p50 - target_p50) / 200))
                   target_p50 by kind: skill 800, agent 1500, hook 200,
                                       cron-spec 300, handoff 6000
latency          = 15 - clamp(0..15, floor((wallclock_p50_s - target_s) / 5))
                   target_s by kind: skill 6, agent 60, hook 2, cron-spec 0, handoff 0
reuse            = 20 if used by ≥3 callers in last 30 d, 12 if 1–2, 4 if 0
self_improve     = 20 if telemetry → critique → gated-patch loop closes,
                   12 if telemetry only,
                   4 if no telemetry
cross_platform   = 10 if Claude+GPT+Gemini all supported (native or shim),
                   6 if 2 of 3, 2 if 1 of 3
automation       = 10 if hook OR cron OR routine OR channel triggers it,
                   5 if /loop only, 0 if manual only
```

Score thresholds:

| Score | Verdict |
|---|---|
| ≥85 | ship |
| 70–84 | ship + queue improvement |
| 50–69 | rewrite before adopting |
| <50 | reject; do not adopt |

The rubric itself is versioned (`score_rubric_version: v1`). Bumping to v2
is only allowed via a synthesis-stage proposal that re-scores ≥10 existing
artefacts under both rubrics and shows the new rubric correlates with the
old at r ≥ 0.85 — preventing score inflation/deflation.

## 7. Self-Improvement (this artefact)

* `cron-self-review` reads the handoff hit-rate (how often a fresh session
  loads `ho-current.md` body vs. just the Resume sentence). If body-load rate
  >40%, the handoff is too lossy → propose a section to add. If <5%, the
  handoff is over-detailed → propose a section to drop.
* Memory dedupe stats feed into a "compression ratio" metric. If ratio falls
  <1.2 (i.e., dedupe isn't paying for itself) the policy thresholds tighten.
* Multi-LLM drift: a checksum of `system-prompt.txt` is compared to the GPT
  Custom GPT's last-known instructions hash (recorded by the user once);
  drift triggers an update reminder.

## 8. Robust Efficiency Score (this artefact)

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 22 | Handoffs ≤200 lines; rubric is the canonical short-form |
| Latency | 15 | 13 | All operations indexed; vector store O(log n) |
| Reuse | 20 | 20 | Rubric is the single reference used by every other handoff |
| Self-improve loop closed | 20 | 18 | Hit-rate metric + dedupe ratio + drift checksum |
| Cross-platform sync | 10 | 9 | `_portable/` contract + drift detection |
| Automation | 10 | 9 | Hooks + cron + routine cover all triggers |
| **Total** | **100** | **91** | |

## 9. Cross-References

* `web2home-handoff-01-orchestrator.md` §6 — uses telemetry CSV defined here.
* `web2home-handoff-02-research-worker.md` §6 — uses worker telemetry shape.
* `web2home-handoff-03-synthesis-brainstorm.md` §6 — uses synthesis telemetry.
* `web2home-handoff-04-folder-structure.md` §3 — paths used here.
* `web2home-handoff-05-skills-and-automation.md` §3, §4 — runs the loop.

## 10. FAQ-style Notes (collected from prompt context)

* **DesktopCommanderMCP without Claude Desktop?** Yes. It's a stand-alone MCP
  server (Node). Attach via `claude mcp add desktop-commander npx -y
  @wonderwhy-er/desktop-commander`. Works in CLI, VS Code, JetBrains, Cursor —
  any MCP-aware client. The "Desktop" refers to the host it commands, not the
  Claude Desktop app.
* **Why list 50+ repos before brainstorming?** Because pattern extraction is
  better when fed many cheap module summaries than one expensive deep dive.
  The orchestrator + workers exist precisely to make broad-and-shallow cheap.
* **Why a 100-point rubric instead of pass/fail?** It enables gated
  auto-improvement (handoff-05 §4): the gate condition is a numeric delta.
* **Why Claude Max as the control plane?** Hooks + skills + subagents +
  Routines + scheduled tasks + sandboxing + auto-mode are all *native* on
  Claude Code; the other two platforms are best treated as workers.
