# web2home-handoff-05 — SKILL.md "On Steroids", Plugin→Skill Transforms, Hooks, Cron, Routines

> Concrete, copy-paste-able templates. Each section is a working artefact:
> drop the file at the path noted in handoff-04, run, done. Every template
> embeds a self-improvement loop.

---

## 1. Content / Intent

Three things, jointly:

1. **Cached prompt skills** — `SKILL.md` used as a *cached prompt on steroids*:
   compact, lazy-loading, trigger-keyword-aware, self-testable, version-pinned.
2. **Plugin → Skill conversion recipes** — turn heavy plugins into 60–80 line
   skills that load only what's needed.
3. **Automation primitives** — Hooks for determinism, Cron / `/loop` /
   Routines for scheduling, all wired to a self-improving loop.

## 2. Use Case

| Goal | Use the template from |
|---|---|
| New domain workflow | §3.1 (SKILL.md skeleton) |
| Migrating an existing plugin | §3.2 (Plugin → Skill) |
| Run something every PreCompact | §3.3 (Hooks) |
| Daily / weekly job inside Claude | §3.4 (Cron + /loop) |
| Cloud-managed schedule (no terminal open) | §3.5 (Routines) |
| Push events from Slack/CI/GitHub | §3.6 (Channels) |
| Self-improvement loop wiring | §4 |
| Multi-LLM portability | §5 |

## 3. Templates

### 3.1 SKILL.md "on steroids"

A skill is a **lazy-loaded, trigger-keyword-bound, cache-friendly prompt** with
a verifiable self-test. Keep `SKILL.md` ≤80 lines; offload reference docs to
`./ref-*.md` and import them only when needed.

```markdown
---
name: skl-mempalace-recall
kind: skill
description: |
  Retrieve relevant longterm memory for the current task using the mempalace-style
  semantic+graph hybrid. Use for any "what did we decide about X?" or
  "remind me what's in memory about Y?" question. Loads <2k tokens at trigger.
version: 0.3.0
last_updated: 2026-05-10
trigger_keywords: [recall, remember, memory, mempalace, what did we decide, history]
args: "<query>  e.g. /skl-mempalace-recall 'auth strategy'"
lazy_imports:
  - "@./ref-retrieval-policy.md"
  - "@./ref-graph-schema.md"
disable_model_invocation: false
target_platform: [claude, gpt, gemini]
efficiency_score: 87
score_rubric_version: v1
source_repos:
  - https://github.com/letta-ai/letta
  - https://github.com/memgraph/memgraph
  - https://github.com/<owner>/mempalace
---

# Recall — semantic + graph hybrid

## When to use
Trigger on memory questions. NOT for fresh research (use /research instead).

## Pre-flight (always)
- If `~/.claude/memory/_index.json` last_updated > 24h, run `/skl-memory-refresh`
  before this skill.

## Procedure
1. Parse `<query>`. Extract entities (people, projects, files, decisions).
2. Vector search top-8 from `~/.claude/memory/vector/` with cosine ≥ 0.78.
3. Graph expand 1-hop from any entity-node hits in `~/.claude/memory/graph/`.
4. Re-rank by recency × cluster-overlap. Keep top-5.
5. Emit a 200-word brief with permalinks back to source markdown shards.
6. If <2 hits, escalate: lazy-load `ref-retrieval-policy.md` and retry with
   relaxed thresholds (0.65), with caveat printed.

## Self-test
$ /skl-mempalace-recall "auth strategy"
Expect: at least 1 hit referencing `mem-*-auth-*.md`. If not, exit code 2.

## Failure mode
Print a single-line note suggesting `/skl-memory-ingest` if the index is empty.

## Anti-bloat
Never paste raw memory shards back; always summarise + link.
```

**Why "on steroids":**
- Frontmatter is **rich** so the model + tools can match without loading body.
- `lazy_imports:` keeps body lean; loaded only on path-specific instructions.
- `trigger_keywords:` shortcuts the model-match step.
- A concrete `Self-test` block makes the skill *verifiable*.
- `efficiency_score` ties into the rubric → eligible for the auto-improvement loop.

### 3.2 Plugin → Skill conversion recipe

Plugins are great but ship a lot of payload. Convert in 6 steps:

1. **Read the plugin manifest.** Identify which sub-skills it bundles.
2. **Pick 1 sub-skill** that the user actually uses ≥1×/week (otherwise drop).
3. **Copy ONLY the prompt body** into a new `SKILL.md` (≤80 lines).
4. **Move reference docs** to `ref-*.md` files in the same skill folder.
5. **Add `trigger_keywords`** distilled from the plugin's "When to use".
6. **Drop tools** that aren't strictly needed (a skill rarely needs both Read
   and Write; pick one if possible).

Worked example (hypothetical plugin → skill):

```
plugin/code-intel-pro/    (1.2 MB, ships LSP + 3 sub-skills)
└── plugin.json
└── skills/
    ├── go-to-symbol/
    ├── find-references/
    └── auto-fix-on-edit/

→ converts to →

~/.claude/skills/skl-go-to-symbol/
└── SKILL.md                     (60 lines)
└── ref-symbol-formats.md        (lazy-imported, 120 lines)
```

Decision rule: **convert** if the plugin's full SKILL.md set is >300 lines
*or* if you only use one sub-skill. **Keep as plugin** if the plugin includes
hooks/MCP that you also use — re-implementing those is its own project.

### 3.3 Hook templates (drop into `~/.claude/hooks/`, register in `settings.json`)

`hk-pre-compact-handoff.sh` — runs *before* compaction, writes a handoff snapshot:

```bash
#!/usr/bin/env bash
set -euo pipefail
TS="$(date -u +%Y-%m-%dT%H-%M-%SZ)"
DEST="$HOME/web2home/handoff/ho-${TS}.md"
{
  echo "---"
  echo "kind: handoff"
  echo "name: ho-${TS}"
  echo "trigger: PreCompact"
  echo "timestamp: ${TS}"
  echo "version: 1"
  echo "---"
  echo
  echo "## Active todo"
  cat "$HOME/.claude/state/current-todos.json" 2>/dev/null || echo "(none)"
  echo
  echo "## Open files"
  cat "$HOME/.claude/state/open-files.txt" 2>/dev/null || echo "(none)"
  echo
  echo "## Recent decisions"
  tail -n 50 "$HOME/.claude/memory/flat/decisions.md" 2>/dev/null || echo "(none)"
  echo
  echo "## Context budget"
  echo "estimated_tokens=${CLAUDE_CONTEXT_TOKENS:-?}"
} > "$DEST"
ln -sf "$DEST" "$HOME/web2home/handoff/ho-current.md"
```

Register in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreCompact": [
      { "matcher": "*", "hooks": [{ "type": "command", "command": "~/.claude/hooks/hk-pre-compact-handoff.sh" }] }
    ],
    "Stop": [
      { "matcher": "*", "hooks": [{ "type": "command", "command": "~/.claude/hooks/hk-stop-summary.sh" }] }
    ],
    "SessionStart": [
      { "matcher": "*", "hooks": [{ "type": "command", "command": "~/.claude/hooks/hk-session-start-load.sh" }] }
    ],
    "PreToolUse": [
      { "matcher": "Write|Edit", "hooks": [{ "type": "command", "command": "~/.claude/hooks/hk-deny-secrets.sh" }] }
      // hk-deny-secrets.sh uses the targeted prefix list in handoff-04 §3.5.1
      // (NOT a broad [A-Z0-9_]{16,}= regex). Prefers `gitleaks` when on PATH.
    ]
  }
}
```

`hk-session-start-load.sh` — quick orientation (≤5 lines printed):

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "🟢 Workspace status:"
test -f "$HOME/web2home/handoff/ho-current.md" && \
  echo "   last handoff: $(readlink "$HOME/web2home/handoff/ho-current.md" | xargs -I{} basename {} .md)"
test -f "$HOME/web2home/synthesis/GENESIS-PLAN.md" && \
  echo "   plan version: $(grep '^version:' "$HOME/web2home/synthesis/GENESIS-PLAN.md" | head -1)"
echo "   active skills: $(ls "$HOME/.claude/skills/" 2>/dev/null | wc -l)"
echo "   memory shards: $(find "$HOME/.claude/memory/flat/" -name '*.md' 2>/dev/null | wc -l)"
echo "   tip: type /skl-mempalace-recall <query> to pull relevant context."
```

### 3.4 Cron / scheduled-tasks (inside Claude Code)

Each `~/.claude/cron/cron-<name>.md` is a small spec file. The
`scheduled-tasks` feature reads them.

`cron-research-refresh.md`:

```markdown
---
name: cron-research-refresh
kind: cron
schedule: "17 3 * * 1"     # Mondays 03:17 UTC
purpose: Re-run the orchestrator weekly; only changed-SHA repos research.
owner: handoff-01
idempotent: true
budget_tokens: 200000
budget_minutes: 30
on_failure: alert via channel `cron-alerts`
last_updated: 2026-05-10
version: 0.2.0
efficiency_score: 90
---

# Body
1. /clear
2. Read ${HANDOFF_DIR}/web2home-handoff-01-orchestrator.md §8.
   (HANDOFF_DIR is set in ~/.claude/settings.json env block; e.g.
    "$HOME/code/LinearAI/web2home" — adjust on first install.)
3. Execute the embedded prompt with INCREMENTAL=true.
4. After Phase 6 (self-improvement), write a row to
   ${DROPZONE}/.telemetry/score-history.csv.
5. If mean worker score dropped ≥5 points vs. last run, post a 3-line note to
   the user's `cron-alerts` channel.
```

`cron-self-review.md`:

```markdown
---
name: cron-self-review
kind: cron
schedule: "0 4 * * 1"      # Mondays 04:00 UTC (after research-refresh)
purpose: Critique handoff-01..06 + skills/cron specs; propose patches.
owner: meta
idempotent: true
budget_tokens: 80000
on_failure: silent (try again next week)
version: 0.1.0
efficiency_score: 84
---

# Body
1. Spawn a subagent with prompt:
   "You are an auditor. Read ~/web2home/.telemetry/score-history.csv,
    synthesis-score-history.csv, and per-skill scores. Identify the lowest
    3 efficiency_score artefacts. Propose minimal Edits to reach +3 points
    each, without quality regression. Write proposals to
    ~/web2home/synthesis/proposed-patches/<isodate>-self-review.md."
2. After: if no auto-applicable patch, append the file's path to
   ~/web2home/handoff/ho-current.md so the user sees it on next session.
```

`cron-memory-dedupe.md`:

```markdown
---
name: cron-memory-dedupe
kind: cron
schedule: "0 5 * * 0"      # Sundays 05:00 UTC
purpose: Compact long-term memory; merge near-duplicates; drop stale shards.
owner: handoff-06
idempotent: true
version: 0.2.0
efficiency_score: 86
---

# Body
1. List all ~/.claude/memory/flat/*.md older than 180 days with no recent retrieval.
2. Cluster by cosine sim ≥0.92; merge into a single shard with provenance.
3. Drop merged originals; update vector + graph indexes.
4. Append run summary to ~/web2home/.telemetry/memory-dedupe.csv.
```

### 3.5 Routines (cloud-managed, no terminal needed)

`~/.claude/routines/rt-weekly-genesis.yaml`:

```yaml
name: rt-weekly-genesis
kind: routine
schedule: "17 3 * * 1"
repository: ivangegovdve-sudo/LinearAI
branch: claude/web2home-genesis
prompt_file: web2home/web2home-handoff-01-orchestrator.md
prompt_section: "## 8. THE PROMPT"
network_access: [github.com, code.claude.com]
timeout_minutes: 45
on_complete:
  open_pr: true
  pr_template: |
    Weekly GENESIS refresh.
    See ~/web2home/synthesis/decision-matrix.md for changes.
on_failure:
  notify: github_issue
efficiency_score: 92
version: 0.1.0
last_updated: 2026-05-10
```

Routines run on Anthropic-managed infra. No laptop required. Output lands as
a PR you can merge to update `~/.claude/` from anywhere.

### 3.6 Channels (push events into a running session)

When a CI run, GitHub review, or external scheduler should *wake* a session,
register a channel. Minimal example for a "memory ingest" channel that takes
URLs and runs an ingest skill:

```yaml
# ~/.claude/mcp/mcp-memory-channel.json
{
  "name": "memory-channel",
  "command": "node",
  "args": ["~/.claude/mcp/memory-channel/server.js"],
  "channel": {
    "events": [{ "name": "ingest", "schema": { "url": "string" } }],
    "permission_relay": "PreToolUse(WebFetch)"
  }
}
```

A POST to the channel sends `{ "event": "ingest", "url": "..." }` and Claude
wakes to run `/skl-memory-ingest <url>`. Combine with GitHub webhooks → every
new starred repo flows into research automatically.

## 4. Self-Improvement Loop (canonical)

The loop is the same shape for skills, hooks, cron jobs, and routines:

```
┌─ run ───────────────────────┐
│  emit telemetry row         │
└──────────────┬──────────────┘
               ▼
┌─ score (rubric handoff-06 §6) ┐
│  store score                  │
└──────────────┬────────────────┘
               ▼
┌─ critique (subagent) ─────────┐
│  diff vs last 5 runs          │
│  propose smallest patch       │
└──────────────┬────────────────┘
               ▼
┌─ gate ─────────────────────────┐
│  if patch lowers tokens ≥5%    │
│     OR raises score ≥3 pts     │
│     AND no quality regression  │
│  → auto-apply (Edit)           │
│  else                          │
│  → stage in proposed-patches/  │
└──────────────┬─────────────────┘
               ▼
        next run uses new spec
```

Three rejected proposals in a row trigger a one-line escalation to the user
in `~/web2home/handoff/ho-current.md`.

## 5. Multi-LLM Portability

| Primitive | Claude Max ($90/mo) | GPT Plus ($20/mo) | Gemini Pro |
|---|---|---|---|
| `SKILL.md` | native — auto-loaded by description match | paste body into Custom GPT *Instructions*; `lazy_imports` become uploaded knowledge files | paste body into a Gem; `lazy_imports` become Connected Docs |
| Subagent | native (`.claude/agents/`) | not available on Plus (Assistants API is Team+) | shim: a Gem can be invoked from another Gem only manually |
| Hook | native | n/a (no per-event scripting) | n/a |
| MCP | native | shim via Custom GPT Action (REST OpenAPI 3.1) | shim via Gemini extension (where available) |
| Cron | native (`/loop`, scheduled-tasks) | external scheduler triggers ChatGPT API or webhook → in-thread reply | shim via Google Tasks where available |
| Channel | native | external webhook → email → user-paste; lossy | shim via Gmail trigger |
| Memory (long-term) | native (`CLAUDE.md` + auto-memory + `~/.claude/memory/`) | ChatGPT *Memory* + uploaded knowledge files | Gemini *Memory* + Connected Docs |
| Plan/Cloud | native (`/ultraplan`, Routines) | n/a | n/a |
| Token meter | native (statusline) | n/a | n/a |

**Pragma:** treat **Claude Max** as the *control plane* and **GPT/Gemini** as
*workers* you delegate isolated tasks to (e.g., a one-shot summary, a long
single-turn rewrite). The Claude session orchestrates; the others execute.

## 6. Synchronisation

* All artefacts in §3.1–§3.5 live in `~/.claude/` (handoff-04 §3.1).
* `_portable/` (handoff-04 §3.7) is regenerated whenever GENESIS changes.
* For GPT Plus / Gemini Pro: copy `system-prompt.txt` into the Custom GPT /
  Gem, attach `memory.json`. A skill `skl-platform-export` does this in
  one shot, prompting the user once for the upload.

## 7. Robust Efficiency Score (this artefact)

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 23 | All templates ≤80 lines; lazy imports keep body small |
| Latency | 15 | 13 | Trigger-keyword shortcuts skip model-match latency |
| Reuse | 20 | 19 | Same template shape across skill/hook/cron/routine |
| Self-improve loop closed | 20 | 18 | Single canonical loop diagram (§4); telemetry → critique → gated patch |
| Cross-platform sync | 10 | 9 | Explicit GPT / Gemini mapping in §5 |
| Automation | 10 | 9 | Hook + cron + routine + channel coverage |
| **Total** | **100** | **91** | |

## 8. Cross-References

* `web2home-handoff-04-folder-structure.md` — paths used here.
* `web2home-handoff-06-handoff-compression.md` — score rubric, retention.
* Claude Code docs: skills, hooks, scheduled-tasks, routines, channels, MCP.
