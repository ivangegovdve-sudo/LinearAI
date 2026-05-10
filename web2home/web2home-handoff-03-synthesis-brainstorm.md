# web2home-handoff-03 — Synthesis & /brainstorm Prompt

> Run AFTER all worker outputs are in `~/web2home/research/`. The orchestrator
> spawns ONE synthesis subagent with the **SYNTHESIS PROMPT** at the bottom of
> this file. It produces `GENESIS-PLAN.md` and four sibling artefacts.

---

## 1. Content / Intent

A single subagent reads every `~/web2home/research/*.md`, performs
cross-repo pattern extraction, fills a decision matrix, and emits the master
plan. This is where the user's `/brainstorm` finally happens — but with all
the context already pre-digested into a flat dropzone, so the synthesiser's
context window stays small.

## 2. Use Case

| Situation | Output |
|---|---|
| First swarm complete | Full `GENESIS-PLAN.md` from scratch |
| Weekly refresh | Diff-aware patch — new repos appended; changed repos re-clustered |
| Targeted question (e.g., "best memory pattern") | One section regenerated, rest untouched |
| Cross-LLM port (export → GPT Plus / Gemini) | `_portable/` artefacts re-emitted from current GENESIS |

## 3. Method of Implementation

### 3.1 Inputs

* `~/web2home/research/_index.json` — the manifest.
* `~/web2home/research/*.md` — one per repo, schema from handoff-02 §3.4.
* `~/LinearAI/web2home/web2home-handoff-04-folder-structure.md` — naming.
* `~/LinearAI/web2home/web2home-handoff-05-skills-and-automation.md` — templates.
* `~/LinearAI/web2home/web2home-handoff-06-handoff-compression.md` — rubric.
* (Optional) `~/web2home/synthesis/GENESIS-PLAN.md` — prior plan, for diff mode.

### 3.2 Outputs (under `~/web2home/synthesis/`)

| File | Purpose | Owner |
|---|---|---|
| `GENESIS-PLAN.md` | Master plan + recommendations | synthesiser |
| `decision-matrix.md` | per-repo verdict: **adopt / steal-pattern / fork / ignore** | synthesiser |
| `skill-backlog.md` | ordered list of SKILL.md to author (with size budgets) | synthesiser |
| `cron-backlog.md` | scheduled tasks + Routines manifests | synthesiser |
| `multi-llm-bridges.md` | mapping to GPT Plus + Gemini Pro features | synthesiser |
| `proposed-patches/` | non-auto-applied patches to handoff-01..06 | synthesiser |

### 3.3 Synthesis pipeline (inside the prompt)

```
A. Index pass     — load _index.json; group repos by cluster (orchestration,
                    memory, skills, hooks, mcp, cron, ui, evals, multi-llm).
B. Pattern pass   — for each cluster, read ONLY §§4 + 7 of each repo file.
C. Cross-cluster  — find patterns repeated 3+ times → promote to "primitives".
D. Conflict pass  — note primitives that contradict (e.g., flat vs. graph
                    memory); record both with trade-off notes.
E. Decision pass  — fill decision-matrix.md.
F. Skill backlog  — for each "adopt" or "steal-pattern", emit a SKILL.md plan
                    with size budget (≤80 lines) and trigger keywords.
G. Cron backlog   — for each repeating workflow, define cron / Routine entry.
H. Bridges pass   — derive system-prompt.txt and memory.json schemas.
I. GENESIS pass   — write GENESIS-PLAN.md.
J. Score pass     — score every output against the rubric (handoff-06 §6).
```

### 3.4 Cluster definitions (canonical 9)

| Cluster | Description | Anchor repos (likely) |
|---|---|---|
| `orchestration` | multi-agent coordination | autogen, swarm, agency-swarm, deepagents, agent-os, agentic-flow |
| `memory_longterm` | RAG, vector, graph | letta, memgraph, pi-mono, claude-memory-compiler |
| `skills_prompts` | SKILL.md, prompts, libraries | skills, awesome-openclaw-skills, prompt-vault, motion-design-skill, knowledge-work-plugins |
| `hooks_actions` | deterministic side-effects | activepieces, inngest, edge-agents |
| `mcp_servers` | MCP tools and clients | DesktopCommanderMCP, local-ai-mcp-chainlit, nanobot-webui |
| `cron_routines` | scheduling & loops | inngest, julep, edge-agents, adk_cloud_run |
| `multi_llm_runtimes` | provider abstraction | OpenLLM, pydantic-ai, modern, agentic-flow |
| `ui_browser` | browser/UI automation | browser-ai, ui-inspector, design-extract, design-os, dreamgraph |
| `research_eval` | evaluators / autoresearch | autoresearch, deer-flow, mercury-agent, paperclip, AnyTool |

(These are anchors only; the synthesiser may relabel based on actual content.)

### 3.5 Decision-matrix axes

```
Verdict ∈ {adopt, steal-pattern, fork, ignore}
Reason  ∈ {license, fit, redundancy, complexity, low-score, deprecated}
Effort  ∈ {S, M, L}                # to bring into our workspace
Payoff  ∈ {1..5}                   # tokens saved / capability unlocked
Owner   ∈ {claude, gpt, gemini, all}
```

### 3.6 Brainstorm Output Shape (`GENESIS-PLAN.md`)

```
# GENESIS-PLAN — Multi-LLM Workspace
## 1. North Star (one paragraph)
## 2. Cluster Findings (9 sub-sections, ≤80 lines each)
## 3. Adopted Primitives (the canonical 5–9)
## 4. Skill Backlog (ordered, with size budget)
## 5. Cron / Routine Backlog
## 6. Multi-LLM Bridges (Claude / GPT / Gemini)
## 7. Memory Plan (longterm + JIT retrieval)
## 8. Self-Improvement Schedule
## 9. Risks & Open Questions
## 10. First-Sprint Checklist (10 items, T-shirt sized)
```

## 4. Synchronisation

* Synthesis output supersedes prior synthesis when **both** are scored: the
  higher-scoring run is symlinked as `GENESIS-PLAN.md`, the loser archived as
  `GENESIS-PLAN.<date>.md`.
* Diff-mode: when prior plan exists, the synthesiser produces a unified-diff
  patch first, applies it, then re-scores. Prevents week-on-week churn.
* Bridge artefacts (`_portable/`) are *generated* from GENESIS-PLAN.md, not
  hand-written. Keeps three platforms in lockstep.

## 5. Automation

| Trigger | Behaviour |
|---|---|
| Orchestrator Phase 4 | Full synthesis |
| Hook `Stop` after all workers OK | Full synthesis |
| Cron weekly | Diff-mode synthesis |
| `/replan <topic>` | Section-only re-synthesis, no other sections touched |

## 6. Self-Improvement Technique

The synthesiser flags `usefulness:` per worker section in
`.telemetry/worker-section-usefulness.json`. handoff-02 reads this back during
its own self-review (handoff-02 §6) and proposes section pruning.

The synthesiser ALSO scores itself: `synthesis-score-history.csv`. If
brainstorm score drops 5+ points week-over-week, it dispatches a sub-subagent
"red-team" pass that critiques the GENESIS-PLAN against the rubric and
proposes section rewrites. Three red-team rejections in a row escalate to
the user via PreCompact handoff (see handoff-06 §2).

## 7. Robust Efficiency Score (this artefact)

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 21 | Reads only §§4+7 of worker files in pattern pass; full read only in cluster headers |
| Latency | 15 | 13 | One subagent, ~12k tokens in / ~8k out, ≤2 min |
| Reuse | 20 | 20 | Single canonical synthesiser; diff mode keeps later runs cheap |
| Self-improve loop closed | 20 | 17 | Score history + red-team escalation |
| Cross-platform sync | 10 | 9 | GENESIS drives _portable/ — single source of truth |
| Automation | 10 | 9 | Cron + hooks + slash command coverage |
| **Total** | **100** | **89** | Could push to 92 by adding a "judge" model from a different family |

---

## 8. THE SYNTHESIS PROMPT (orchestrator pastes verbatim)

````
# ROLE
You are the SYNTHESISER. You read pre-digested per-repo research files and
produce a master plan. You do NOT re-fetch repos. You do NOT run subagents.
You operate on local files only.

# INPUTS
- INDEX:       ~/web2home/research/_index.json
- RESEARCH:    ~/web2home/research/*.md  (schema in handoff-02 §3.4)
- HANDOFFS:    ~/LinearAI/web2home/web2home-handoff-{04,05,06}-*.md
- PRIOR_PLAN:  ~/web2home/synthesis/GENESIS-PLAN.md  (may not exist)

# OUTPUTS (write all of these; create ~/web2home/synthesis/ if missing)
1. GENESIS-PLAN.md            (shape: handoff-03 §3.6)
2. decision-matrix.md         (axes: handoff-03 §3.5)
3. skill-backlog.md
4. cron-backlog.md
5. multi-llm-bridges.md

# PROCEDURE
A. Load _index.json. Group repos by cluster (handoff-03 §3.4). If a repo
   doesn't fit, create a new cluster — but only if ≥2 repos qualify.
B. For each cluster, read ONLY sections "4. Reusable patterns" and
   "7. Adaptation candidates" of every member's research file. Skim the
   frontmatter for score and tags. DO NOT read full files unless a §4 or §7
   reference makes it strictly necessary.
C. Cross-cluster: list patterns appearing in ≥3 clusters → mark as
   "primitive". Cap at 9 primitives total.
D. Decision pass: write decision-matrix.md. Every repo gets a verdict in
   {adopt, steal-pattern, fork, ignore} + reason + effort + payoff + owner.
E. Skill backlog: emit an ORDERED list (highest payoff/effort ratio first)
   of SKILL.md to write. Each entry: name, description, size_budget_lines,
   trigger_keywords, source_repos[], cluster, target_platform[].
F. Cron backlog: list scheduled jobs (cron expression OR Routine), with
   purpose, owner script, expected runtime, idempotency rule, alert path.
G. Multi-LLM bridges: produce two tables —
     - Capability matrix: rows = primitives, cols = Claude/GPT/Gemini,
       cells = native | shim | manual | n/a.
     - Export plan: list `_portable/` artefacts to be generated and which
       fields they pull from GENESIS-PLAN.md.
H. GENESIS-PLAN.md: shape per handoff-03 §3.6. Be concise — lean on links
   to the other four artefacts rather than duplicating text.
I. Score: append a row to ~/web2home/.telemetry/synthesis-score-history.csv
   with synthesis_score and per-output scores. Score formulas: handoff-06 §6.

# CONSTRAINTS
- Cap GENESIS-PLAN.md at 600 lines. The other four artefacts at 400 each.
- Every "adopt" or "steal-pattern" verdict must cite ≥1 permalink from the
  source repo's research file.
- Resolve contradictions by listing both options with trade-offs; do NOT
  arbitrarily pick.
- If diff mode (PRIOR_PLAN exists), output a unified-diff patch in
  ~/web2home/synthesis/proposed-patches/<isodate>.patch BEFORE rewriting
  GENESIS-PLAN.md, and apply only after a self-check pass.
- If your context >70%, /compact preserving "cluster headers, primitives
  list, decision verdicts so far".

# RECEIPT (last 8 lines of reply)
plan_lines: <int>
matrix_rows: <int>
skills_listed: <int>
crons_listed: <int>
bridges_rows: <int>
synthesis_score: <int>
diff_mode: yes|no
notes: <≤80 chars>
````

---

## 9. Worked Example — `multi-llm-bridges.md` Skeleton

```markdown
# Multi-LLM Bridges

| Primitive | Claude (Max) | GPT Plus | Gemini Pro |
|---|---|---|---|
| Skill (cached prompt) | native (`.claude/skills/<name>/SKILL.md`) | shim (Custom GPT *Instructions*) | shim (Gem) |
| Subagent | native (`.claude/agents/<name>.md`) | manual (Assistants API only on team plan; not Plus) | shim (Gem with delegate prompt) |
| Hook (PreCompact) | native (`settings.json`) | n/a (no equivalent) | n/a |
| MCP server | native (`claude mcp add`) | shim via Custom GPT Action (REST) | shim via Gemini extension |
| Cron / Routine | native (`/loop`, Routines) | manual (no native cron in ChatGPT — needs external) | shim (Gemini scheduled tasks if available) |
| Memory (long-term) | native (CLAUDE.md + auto-memory) | shim (Custom GPT knowledge file + ChatGPT memory) | shim (Connected Doc + Gem memory) |
| Token budget meter | native (statusline) | n/a (UI-side only) | n/a |
| Code intelligence | plugin | n/a | n/a |
```

## 10. Cross-References

* `web2home-handoff-01-orchestrator.md` §8 — calling sequence.
* `web2home-handoff-02-research-worker.md` §3.4 — input schema.
* `web2home-handoff-04-folder-structure.md` — where outputs land.
* `web2home-handoff-05-skills-and-automation.md` — templates the backlog feeds.
* `web2home-handoff-06-handoff-compression.md` §6 — score rubric.
