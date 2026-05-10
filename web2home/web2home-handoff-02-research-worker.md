# web2home-handoff-02 — Per-Repo Research Worker Prompt

> The orchestrator (handoff-01) instantiates one subagent per target repo and
> hands it the **WORKER PROMPT** below with `{{REPO_URL}}` substituted.
> Every worker uses the **same** template so outputs collide cleanly during
> synthesis. Do not rewrite the prompt per-repo — that breaks determinism.

---

## 1. Content / Intent

A parameterised research prompt that produces a 600–800 line markdown module
for a single repository. The module is schema-fixed: the synthesis stage
(handoff-03) parses it deterministically.

## 2. Use Case

* Initial swarm: 50–60 repos in 10–12 batches.
* Incremental refresh: only repos whose default-branch SHA changed since the
  last `_index.json` entry.
* Spot research: invoked manually as `/research <repo-url>` slash command (the
  command file embeds this prompt verbatim).

## 3. Method of Implementation

### 3.1 Worker lifecycle

```
spawn (Agent: general-purpose)
  └─ ingest WORKER PROMPT with REPO_URL substituted
     └─ Phase A: identity + heads (gh api or WebFetch /repos/{owner}/{repo})
     └─ Phase B: tree probe (top-level dirs, README, package manifest, LICENSE)
     └─ Phase C: targeted reads (3–8 files, never whole files >200 lines)
     └─ Phase D: pattern extraction (skill/agent/hook/cron/mcp/prompt artefacts)
     └─ Phase E: write ~/web2home/research/<slug>.md
     └─ Phase F: 6-line receipt back to orchestrator
return
```

### 3.2 Slug rules

* Lowercase the repo name.
* Replace `_` and spaces with `-`.
* If two repos collide post-slugging, suffix with `-{owner}` (e.g.,
  `swarm-openai`, `swarm-ruvnet`).

### 3.3 Hard limits inside a worker

| Limit | Value | Reason |
|---|---|---|
| Max files read | 8 | Caps token blast radius |
| Max lines per file | 200 | Forces representative reads |
| Max output lines | 800 | Synthesis budget |
| Max wallclock | 6 min | Trips orchestrator backoff |
| Max WebFetch calls | 12 | Rate-limit safety |

### 3.4 Output schema (frontmatter + sections, MANDATORY)

```yaml
---
slug: <kebab-slug>
repo_url: <upstream or fork URL the worker actually inspected>
fork_url: <ivangegovdve-sudo/<name> if different>
upstream: <owner/repo>
default_branch: <name>
head_sha: <40-char>
license: <SPDX id>
primary_language: <lang>
stars: <int|null>
last_commit_iso: <ISO-8601>
fetched_at: <ISO-8601>
worker_run_id: <uuid>
status: ok|partial|failed
efficiency_score: <0..100>
score_rubric_version: v1
focus_themes: [orchestration, memory, skills, hooks, mcp, cron, multi-llm]
tags: [...]
---
```

Body sections (each ≤120 lines):

```
## 1. Elevator pitch
## 2. Architecture map           (modules → files → role)
## 3. Artefacts present          (skills/agents/hooks/cron/mcp/prompts/tools)
## 4. Reusable patterns          (concrete extracts, with permalinks)
## 5. Token-efficiency analysis  (bloat hotspots; what to prune)
## 6. Multi-LLM portability      (Claude / GPT Plus / Gemini Pro fit)
## 7. Adaptation candidates      (→ SKILL.md skeletons, ≤80 lines each)
## 8. Risks & licence flags
## 9. Efficiency score breakdown (per dimension, see §6 of handoff-06)
## 10. Sources                   (file_path:line_range — permalink)
```

### 3.5 Permalink format

`https://github.com/<owner>/<repo>/blob/<head_sha>/<path>#L<start>-L<end>`

If the worker uses `gh api` instead of WebFetch, it constructs the same URL
using the SHA returned by `/repos/{owner}/{repo}`.

## 4. Synchronisation

* Workers do not talk to each other. The dropzone is the only shared state.
* The orchestrator updates `_index.json` after each batch — the worker never
  writes there directly.
* If a worker re-runs and the file already exists, it overwrites only when its
  `head_sha` differs; otherwise it exits with `status: skipped`.

## 5. Automation

* Triggered by handoff-01 §8 prompt (Phase 2) for mass runs.
* Triggered by `/research <url>` slash command for one-offs.
* Triggered by `cron-research-refresh` (handoff-05 §3.2) weekly.

## 6. Self-Improvement Technique

Every worker run logs to `~/web2home/.telemetry/workers/<run_id>.json`:

```json
{
  "slug": "...",
  "tokens_in": 0, "tokens_out": 0,
  "files_read": 0, "webfetch_calls": 0,
  "wallclock_s": 0,
  "score": 0,
  "rubric_version": "v1",
  "anomalies": []
}
```

`cron-self-review` (handoff-05 §3.4) reads the last 100 logs weekly, computes
percentile distributions, and proposes patches to **this file's WORKER PROMPT
block**. Allowed auto-patches:

* Tighten line/file limits if mean output > 800.
* Drop sections that score <0.3 on "useful in synthesis" (a flag set by the
  synthesiser when sections are unused).
* Add a section if synthesis routinely had to re-spawn workers to fetch it.

## 7. Robust Efficiency Score (this artefact)

| Dimension | Weight | Score | Notes |
|---|---:|---:|---|
| Token efficiency | 25 | 23 | Hard caps + skipped-on-SHA-match keep cost flat after week 1 |
| Latency | 15 | 12 | 8 reads + 12 fetches per worker; fits under 6 min |
| Reuse | 20 | 20 | Single template, parameterised — perfect reuse |
| Self-improve loop closed | 20 | 16 | Auto-patch is gated; expand to add/remove sections needs care |
| Cross-platform sync | 10 | 9 | Output is plain markdown + frontmatter — universally portable |
| Automation | 10 | 9 | Runs from cron, slash, or orchestrator |
| **Total** | **100** | **89** | Score ceiling rises if we add a `code_intelligence` plugin probe |

---

## 8. THE WORKER PROMPT (orchestrator pastes this verbatim, with substitutions)

````
# ROLE
You are a research worker. Your sole task is to produce ONE markdown file
describing a single GitHub repository, conforming exactly to the schema in
{{HANDOFF_DIR}}/web2home-handoff-02-research-worker.md §3.4. You are not
asked to evaluate or implement anything. Be concrete, be source-cited, be
brief.

# INPUTS
- REPO_URL:    {{REPO_URL}}
- FORK_OWNER:  {{OWNER}}                    # default: ivangegovdve-sudo
- HANDOFF_DIR: {{HANDOFF_DIR}}              # absolute path; orchestrator resolves
- DROPZONE:    {{DROPZONE}}/research        # absolute path; orchestrator resolves
- HEAD_LIMIT:  8 files, 200 lines each, 12 WebFetch calls, 6 min wallclock
- FOCUS_THEMES: orchestration, memory, skills, hooks, mcp, cron, multi-llm,
                token-efficiency, self-improvement, scheduling, retrieval

# PROCEDURE
A. Resolve the repo:
   1. Try {{REPO_URL}} (fork). If 404 or empty, resolve to upstream by reading
      the fork's parent field (gh api repos/<owner>/<repo>) and use upstream.
   2. Capture: default_branch, head_sha, license, primary_language, stars,
      last_commit ISO timestamp.
B. Tree probe:
   - Fetch top-level dir listing.
   - Identify: README*, LICENSE*, package.json/pyproject.toml/Cargo.toml,
     .claude/, .cursor/, skills/, agents/, hooks/, prompts/, mcp/, cron*,
     workflows/, examples/, src/.
C. Targeted reads (≤8):
   - README (first 200 lines)
   - 1 manifest (package.json or pyproject.toml)
   - up to 4 of: a representative skill/agent/prompt/hook file
   - up to 2 of: a cron/scheduler file or routine manifest
   Never read whole files; use line ranges. Prefer files that are themselves
   prompts, skill definitions, or orchestration logic.
D. Extract artefacts table — present-or-absent for each of:
   skills, subagents, hooks, MCP servers, cron jobs, slash commands,
   memory/RAG, evaluators, scoring, self-improve loops, multi-LLM
   abstractions, token-budget controls.
E. Pattern extraction:
   - For each artefact present, copy at most 20 lines as an extract, with a
     permalink in `https://github.com/<owner>/<repo>/blob/<sha>/<path>#L<a>-L<b>`
     form.
   - Note bloat hotspots: configs >200 lines, deeply-nested folder trees,
     hardcoded provider keys, etc.
F. Score (0–100) using the rubric below:

   token_efficiency  /25  cheap to keep loaded; lazy-loaded?
   latency           /15  startup time; cold-start prompt size
   reuse             /20  drop-in vs. fork-only? portable across LLMs?
   self_improve      /20  has a feedback loop / scoring / regen?
   cross_platform    /10  works on Claude+GPT+Gemini? (or 1 of 3?)
   automation        /10  cron / hooks / triggers present?
   ---
   Show the breakdown. Show the total.

G. Adaptation candidates (≤3 per repo). For each, write a SKILL.md skeleton
   ≤80 lines. Frontmatter MUST include: name, description, args, version,
   trigger_keywords, lazy_imports, last_updated, source_repo.

H. Write the output to {{DROPZONE}}/research/<slug>.md. Slug rules in §3.2.

# OUTPUT FORMAT
Strictly the schema in §3.4. No prose preamble. No conclusion. No emoji.
Every claim above section "10. Sources" MUST be backed by a permalink in §10.

# RECEIPT (last 6 lines of your reply, exactly this shape)
slug: <slug>
score: <int>
bytes: <int>
sources: <int>
status: ok|partial|failed
notes: <≤80 chars>

# CONSTRAINTS
- Do NOT clone the repo locally.
- Do NOT execute repo code.
- Do NOT speculate. If a section is empty, write "n/a — none found".
- Do NOT repeat the input prompt back.
- If WebFetch is rate-limited, retry with `gh api` once, then mark partial.
- Cap the output file at 800 lines.
````

---

## 9. Substitution Examples

```
{{REPO_URL}} = https://github.com/ivangegovdve-sudo/agency-swarm
{{REPO_URL}} = https://github.com/ivangegovdve-sudo/DesktopCommanderMCP
{{REPO_URL}} = https://github.com/ivangegovdve-sudo/letta
```

## 10. Cross-References

* `web2home-handoff-01-orchestrator.md` — dispatch + concurrency.
* `web2home-handoff-03-synthesis-brainstorm.md` — consumer of these outputs.
* `web2home-handoff-06-handoff-compression.md` — efficiency-score rubric.
