# Daily AI Brief — Design

**Date:** 2026-04-24
**Owner:** Ennio (techno1731@gmail.com)
**Project:** `daily-ai-brief`
**Status:** Design approved, pending spec review

## Context

Ennio is building Sapiens Lab, a responsible AI tooling company in Spain. Stack: LiteLLM Proxy, Langfuse (self-hosted), DeepEval, MLflow, Kubernetes on AWS. Target market: Spanish/EU public sector, so EU AI Act and sovereignty are first-class concerns.

This project builds a recurring daily AI intelligence briefing delivered by email. The briefing is a senior-engineer-to-senior-engineer digest biased toward signal over volume (target: 8 great items > 30 mediocre), covering 11 topic areas relevant to Sapiens Lab's stack and customer base.

## Goals

- Daily email digest of the last 24h (72h on Mondays) of AI news relevant to Sapiens Lab.
- Covers 11 prioritized categories (see Source Map).
- Grounded in concrete primary sources (GitHub releases, official blogs, arXiv), not hype threads.
- Calls out EU AI Act / Spanish public sector relevance explicitly.
- Delivered as a well-designed HTML email to `techno1731@gmail.com` by 07:30 Europe/Madrid on weekdays.
- Trivial to iterate on: editing sources or prompts is a git push, not a routine redeploy.

## Non-goals (v1)

- Persisted "since last run" state (rolling 24h/72h window is sufficient).
- Heartbeat monitoring / alerting if a run fails.
- Retry queues or automated failure recovery.
- Deduplication against previous days' emails.
- Archive / search UI.
- Langfuse tracing (deferred until a K8s-hosted version).
- Twitter/X as a source (poor signal/noise, bad WebFetch).
- Per-run cost tracking beyond what the Anthropic console already provides.

## Architecture

### Execution environment

Claude Code `/schedule` routine. Cron: `30 7 * * 1-5`, timezone `Europe/Madrid`. Weekends are skipped.

**Window computation rule:** Monday → 72h lookback (covers the weekend). Tue–Fri → 24h lookback. **The window does not widen after holidays or missed runs.** If a Monday run fails or Tuesday is a Spanish public holiday, the next successful run still uses its normal window; coverage gaps are accepted rather than retroactively filled. Persisted-state-based recovery is an explicit v2 feature.

The routine prompt (`routine-prompt.md`) is a thin shim. At run time, the orchestrator WebFetches `https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/config.yaml` to load the full config: source list, per-category sub-agent prompts, email template skeleton. This means iteration happens in git — edit YAML, commit, push, next run uses the new config — without touching `/schedule`.

### Orchestrator + sub-agents

```
/schedule cron (daily 07:30 Europe/Madrid, Mon–Fri)
          │
          ▼
┌───────────────────────────────────────────┐
│        Orchestrator Agent (Opus)          │
│                                           │
│  1. WebFetch config.yaml                  │
│  2. Compute window:                       │
│       Mon → 72h, Tue–Fri → 24h            │
│  3. Dispatch 11 sub-agents in parallel    │
│  4. Collect JSON results                  │
│  5. Write TL;DR + emerging patterns       │
│  6. Render HTML template                  │
│  7. Send via Gmail MCP                    │
└───────────────────────────────────────────┘
          │                 ▲
          │ Agent tool      │ JSON
          ▼                 │
  ┌─────────┐ ┌─────────┐ ┌─────────┐  ...11 total
  │ Cat 1   │ │ Cat 2   │ │ Cat 3   │
  │ Models  │ │ Agents  │ │  RAG    │
  │ (Sonnet)│ │ (Sonnet)│ │ (Sonnet)│
  └─────────┘ └─────────┘ └─────────┘
       │           │           │
   WebFetch     WebFetch    WebFetch
   WebSearch    WebSearch   WebSearch
```

**Orchestrator (Opus):** scheduling/window logic, sub-agent dispatch, final synthesis (TL;DR, "Worth deeper research", "Emerging patterns" cross-category analysis), HTML rendering, Gmail MCP delivery. Does NOT fetch sources directly.

**Sub-agents (Sonnet, 11 instances in parallel):** each gets a narrowly scoped prompt with one category's primary source URLs and WebSearch hints baked in. Returns a strict JSON shape:

```json
{
  "category": "<id>",
  "nothing_notable": false,
  "items": [
    {
      "headline": "...",
      "summary": "2-3 sentence takeaway",
      "sapiens_angle": "one line or null",
      "url": "https://..."
    }
  ],
  "errors": []
}
```

**Sub-agent output contract:**
- `items` is capped at **5 per category**. If a sub-agent finds more than 5 notable items, it ranks and returns the top 5. This bounds the digest size and forces per-category prioritization.
- `nothing_notable: true` **must** coincide with `items: []`. Treat this as the canonical "quiet category" encoding. The orchestrator treats `nothing_notable: false` with an empty `items` array as a sub-agent bug and logs it as an error.
- `errors` is a list of short strings (source URLs that 404'd, WebSearch queries that returned nothing useful, etc.). Non-fatal — surfaced in the email's "Run notes" footer.

**Model choice rationale:**
- Sub-agents do structured extraction + ranking against a tight spec. Sonnet handles this cheaply and fast.
- Orchestrator is the judgment layer (what goes in TL;DR, which patterns to name, how to phrase Sapiens-Lab angles). Opus earns its cost there.
- Estimated run cost: $0.15–0.40/day.

## Source Map

The config.yaml encodes sources per category using this shape:

```yaml
thresholds:
  degraded_run_category_failures: 6   # if >= this many sub-agents fail, subject gets [DEGRADED RUN]
  max_items_per_category: 5

categories:
  - id: model-releases
    primary:
      - { url: "https://anthropic.com/news", type: blog }
      - { url: "https://openai.com/news", type: blog }
      ...
    secondary:
      - "Qwen release last 24h"
      - "DeepSeek release last 24h"
```

Noise filtering is handled by the sub-agent prompt's narrative rules (see "Noise rules" below), not by YAML stopword lists. Stopwords were considered and rejected — LLM judgment is better than keyword blacklists for "is this hype or substance."

### Full category → source mapping

| # | Category | Primary sources | Secondary (WebSearch) |
|---|----------|----------------|----------------------|
| 1 | Model releases & updates | anthropic.com/news, openai.com/news, deepmind.google/discover/blog, ai.meta.com/blog, mistral.ai/news, huggingface.co/blog, huggingface.co/models (trending) | "Qwen release", "DeepSeek release", "open weights release" |
| 2 | Agents & multi-agent systems | github.com/anthropics/claude-agent-sdk/releases, github.com/langchain-ai/langgraph/releases, github.com/crewAIInc/crewAI/releases, github.com/microsoft/autogen/releases, github.com/openai/openai-agents-python/releases | simonwillison.net, latent.space, smol.ai |
| 3 | RAG | arxiv.org/list/cs.IR/new, arxiv.org/list/cs.CL/new | eugeneyan.com, hamel.dev, "graph RAG paper", "agentic RAG" |
| 4 | MCP | github.com/modelcontextprotocol/specification/releases, github.com/modelcontextprotocol/servers (recent commits/PRs), modelcontextprotocol.io (blog/changelog) | HN "mcp" tag, Simon Willison |
| 5 | Claude Skills & Claude Code | anthropic.com/news (filtered), docs.claude.com/en/release-notes/claude-code | Community skill repos, r/ClaudeAI top of day |
| 6 | LLM gateways / proxies | github.com/BerriAI/litellm/releases, github.com/Helicone/helicone/releases, github.com/Portkey-AI/gateway/releases, Kong AI Gateway changelog, Cloudflare AI Gateway changelog | HN |
| 7 | Evaluation frameworks | github.com/confident-ai/deepeval/releases, github.com/explodinggradients/ragas/releases, github.com/promptfoo/promptfoo/releases, github.com/UKGovernmentBEIS/inspect_ai/releases, Braintrust blog, LangSmith changelog | arXiv "LLM evaluation" |
| 8 | LLM observability | github.com/langfuse/langfuse/releases, github.com/Arize-ai/phoenix/releases, LangSmith changelog, Helicone blog, github.com/traceloop/openllmetry/releases | OTEL GenAI semconv updates |
| 9 | Self-hosting & deployment | github.com/vllm-project/vllm/releases, github.com/sgl-project/sglang/releases, github.com/huggingface/text-generation-inference/releases, github.com/ollama/ollama/releases | r/LocalLLaMA top of day, quantization news |
| 10 | AI platform / DevEx | (no single hub; lean on secondary) | latent.space, hamel.dev, huyenchip.com, "internal AI platform" HN |
| 11 | EU AI Act & regulation | digital-strategy.ec.europa.eu/en/policies/ai-act, AESIA (Agencia Española de Supervisión de IA) news, red.es news, BSC-CNS news | "EU AI Act", "ENS certification AI", "Reglamento IA"; Spanish-language sources explicitly required |

**Noise rules baked into every sub-agent prompt:**
- Ignore hype threads, "X is dead" takes, pure vendor positioning.
- A vendor blog post is only worth including if it describes new functionality, not marketing positioning.
- Prefer papers and production postmortems over vendor blogs for RAG.
- For category 11, always include Spanish-language sources (EU AI Act implementation often lands in national languages first).

## Email Template

### Subject line
```
[AI Brief] Fri 2026-04-24 — 14 items across 9 categories
```
Scannable preview, sortable by date, item count reflects slow vs busy days.

### HTML structure

Mobile-first (max-width 600px, centered). Inline CSS only (Gmail strips `<style>` blocks). No images, no web fonts, no tracking pixels.

```
Daily AI Brief — Fri Apr 24 2026     ← small header, system sans

┌─ TL;DR ──────────────────────────┐  ← bordered callout, Georgia serif
│ One paragraph, 3 big things.     │
└──────────────────────────────────┘

MODEL RELEASES                        ← section header, uppercase, thin border
──────────────

Claude Opus 4.8 ships with …          ← headline (bold sans, 16px)
Two-sentence summary. Two-sentence    ← body (Georgia 15px, 1.55 line height)
summary that reads cleanly.
→ For Sapiens Lab: could replace      ← Sapiens angle (italic, accent color)
  our orchestrator model.
anthropic.com/news/claude-4-8  ↗      ← link (underlined, accent)

[next item…]

AGENTS & MULTI-AGENT
...

─────────────────────────────
WORTH DEEPER RESEARCH                 ← dedicated footer section
• Item 1 (why)
• Item 2 (why)

EMERGING PATTERNS
One or two cross-cutting themes.

─────────────────────────────
Run notes: 2 categories failed        ← small grey, only if failures
today (MCP, eval). Will retry
tomorrow.
```

### Typography and palette
- Headers: `-apple-system, Segoe UI, sans-serif`.
- Body: `Georgia, 'Times New Roman', serif`.
- Text: `#1a1a1a` near-black.
- Secondary: `#666` grey.
- Accent (links, Sapiens angle): `#c14500` warm rust — readable in Gmail light and dark themes, avoids generic blue.

### Rendering
The orchestrator holds the HTML skeleton in its prompt (embedded in config.yaml's `email_template` key) and fills it with the collected items. Template is plain string substitution — no Jinja, Mustache, or dependencies.

## Failure Handling

| Failure | Behavior |
|---|---|
| One sub-agent throws / times out / returns malformed JSON | Category shown as `[fetch failed this run — check trace]`. Other categories unaffected. Error logged in "Run notes" footer. |
| `thresholds.degraded_run_category_failures` or more sub-agents fail (default: 6) | Email still sent, subject prefixed `[DEGRADED RUN]`. Signals investigation needed rather than silently delivering thin digests. Threshold is tunable in config.yaml. |
| Gmail MCP send fails | Orchestrator logs the full rendered HTML to its stdout so it's recoverable from the `/schedule` trace. No retry — `/schedule` provides run-level retry. Next day's run sends normally. |
| Orchestrator itself crashes before finishing | `/schedule` surfaces the failure in its UI. User notices no email by 09:00 and investigates manually. Acceptable for v1. |
| Zero categories had news (rare, e.g., holiday) | Send a short "Quiet day — nothing notable across all categories" email. Keeps delivery cadence predictable — silence would create ambiguity about whether the cron broke. |

**Observability in v1:** the only log retention is whatever `/schedule` keeps in its run history. There is no structured logging, no Langfuse trace, no archived HTML. If something goes wrong past the `/schedule` retention window, it's gone. This is acceptable for v1 — structured observability is a v2 feature when this migrates to K8s.

## Project Layout

```
daily-ai-brief/
├── README.md                   ← how to deploy + iterate
├── routine-prompt.md           ← thin shim pasted into `/schedule create`
├── config.yaml                 ← all knobs (sources, sub-agent prompt template,
│                                  email template, category prompts)
├── docs/ultrapowers/specs/
│   └── 2026-04-24-daily-ai-brief-design.md   ← this file
├── .claude/
│   └── ultrapowers-preferences.json
└── .gitignore
```

**Key architectural split:**
- `routine-prompt.md` (~40 lines) is frozen after initial deployment. Tells the orchestrator to fetch config.yaml and execute per its instructions.
- `config.yaml` is where day-to-day tuning happens. Editing it is a normal git workflow. Next run picks up the change automatically.

**Repo visibility:** public. Simpler (no PAT needed for WebFetch), and the repo doubles as reference material for Sapiens Lab customers.

## Deployment Plan

**First-time setup (one-off, ~20 min):**

1. `git init` locally, scaffold files, commit.
2. Create `github.com/techno1731/daily-ai-brief` (public), push.
3. In a Claude Code session, authenticate Gmail MCP via `mcp__claude_ai_Gmail__authenticate`.
4. Run `/schedule create`, paste contents of `routine-prompt.md`, set cron `30 7 * * 1-5`, timezone `Europe/Madrid`.
5. Trigger a manual run from the `/schedule` UI to validate end-to-end (email arrives, formatting correct, all categories populate).
6. Once validated, let it run autonomously from the next weekday 07:30.

**Iteration flow (ongoing):**
- Add/remove sources, tweak sub-agent prompts, adjust email styling → edit `config.yaml` → commit → push. No routine redeploy.
- Change orchestrator-level behavior (e.g., failure thresholds) → edit `routine-prompt.md` → `/schedule update`. Expected to be rare.

## v2 Considerations (deferred)

Logged here so they don't clutter v1 but aren't lost:

- **Langfuse tracing** — port to K8s-hosted runner so traces flow to self-hosted Langfuse. Required before this is a Sapiens Lab reference implementation, not just personal tooling.
- **Persisted since-last-run state** — gist or tiny repo storing last-successful-run timestamp.
- **Heartbeat** — separate cron that checks whether the 07:30 email arrived, pings user if not.
- **Local dry-run script** — run the orchestrator prompt against local Claude Code without sending email, for prompt tuning.
- **Dedup against recent digests** — avoid re-featuring items from the previous 3 days.
- **Archive page** — static site built from committed daily digests.
- **DeepEval LLM-as-judge regression tests** — evaluate synthesis quality against a held-out "gold digest" to catch prompt regressions.

## Open questions (resolved during design)

- Execution shape: `/schedule` routine + Gmail MCP. ✅
- Source strategy: hybrid (structured primary + WebSearch long-tail). ✅
- Cadence: daily 07:30 Europe/Madrid weekdays, 72h window on Monday, no persisted state. ✅
- Architecture: orchestrator + 11 parallel Sonnet sub-agents. ✅
- Email format: well-designed HTML. ✅
- Project home: `/Users/techno1731/Code/Personal/daily-ai-brief`. ✅
- Workflow prefs: auto-commit on, auto-push off, commit design docs on. ✅
