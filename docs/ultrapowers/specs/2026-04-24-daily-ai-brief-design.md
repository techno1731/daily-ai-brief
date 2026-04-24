# Daily AI Brief — Design

**Date:** 2026-04-24
**Owner:** Ennio (techno1731@gmail.com)
**Project:** `daily-ai-brief`
**Status:** Design approved, research complete, pending skills-audit and implementation planning

## Context

Ennio is building Sapiens Lab, a responsible AI tooling company in Spain. Stack: LiteLLM Proxy, Langfuse (self-hosted), DeepEval, MLflow, Kubernetes on AWS. Target market: Spanish/EU public sector, so EU AI Act and sovereignty are first-class concerns.

This project builds a recurring daily AI intelligence briefing delivered by email. The briefing is a senior-engineer-to-senior-engineer digest biased toward signal over volume (target: 8 great items > 30 mediocre), covering 11 topic areas relevant to Sapiens Lab's stack and customer base.

## Goals

- Daily email digest of the last 24h (72h on Mondays) of AI news relevant to Sapiens Lab.
- Covers 11 prioritized categories (see Source Map).
- Grounded in concrete primary sources (GitHub releases, official blogs, arXiv API), not hype threads.
- Calls out EU AI Act / Spanish public sector relevance explicitly.
- Delivered as a well-designed HTML email to `techno1731@gmail.com` by 07:30 Europe/Madrid on weekdays, sent via **Resend** (transactional email API) from a branded `brief@ultrapowers.dev` verified sender (DKIM + DMARC aligned).
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

Claude Code `/schedule` routine. Cron: `30 7 * * 1-5`, timezone `Europe/Madrid`. Weekends are skipped. Scheduled routines run on Anthropic-managed cloud infrastructure — the user's laptop does not need to be open.

**Window computation rule:** Monday → 72h lookback (covers the weekend). Tue–Fri → 24h lookback. **The window does not widen after holidays or missed runs.** If a Monday run fails or Tuesday is a Spanish public holiday, the next successful run still uses its normal window; coverage gaps are accepted rather than retroactively filled. Persisted-state-based recovery is an explicit v2 feature.

The routine prompt (`routine-prompt.md`) is a thin shim. At run time, the orchestrator WebFetches `https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/config.yaml` to load the full config: source list, per-category sub-agent prompts, email template skeleton. This means iteration happens in git — edit YAML, commit, push, next run uses the new config — without touching `/schedule`.

### Runtime secrets (routine environment variables)

Configured once when the routine is created in the `/schedule` UI:

| Var | Purpose | Required? |
|---|---|---|
| `RESEND_API_KEY` | Outbound HTML email via Resend API | Required |
| `GITHUB_TOKEN` | PAT for GitHub REST API calls (raises rate limit from 60/hr unauth to 5000/hr) | Recommended |

No OAuth-refresh workflow. No Gmail API credentials. No domain-level DNS setup in v1 (using Resend's shared sender; migration to a custom domain is v2).

### Orchestrator + sub-agents

```
/schedule cron (daily 07:30 Europe/Madrid, Mon–Fri)
          │
          ▼
┌───────────────────────────────────────────┐
│        Orchestrator Agent (Opus)          │
│                                           │
│  1. WebFetch config.yaml from GitHub      │
│  2. Compute window:                       │
│       Mon → 72h, Tue–Fri → 24h            │
│  3. Dispatch 11 sub-agents in parallel    │
│     (fallback: sequential if Task tool    │
│      unavailable in /schedule — see note) │
│  4. Collect JSON results                  │
│  5. Write TL;DR + emerging patterns       │
│  6. Render HTML template                  │
│  7. Send via Resend API (Bash curl)       │
└───────────────────────────────────────────┘
          │                 ▲
          │ Task tool       │ JSON
          ▼                 │
  ┌─────────┐ ┌─────────┐ ┌─────────┐  ...11 total
  │ Cat 1   │ │ Cat 2   │ │ Cat 3   │
  │ Models  │ │ Agents  │ │  RAG    │
  │ (Sonnet)│ │ (Sonnet)│ │ (Sonnet)│
  └─────────┘ └─────────┘ └─────────┘
       │           │           │
   WebFetch     WebFetch    WebFetch
   WebSearch    WebSearch   WebSearch
   Bash(curl)   Bash(curl)  Bash(curl)  ← for GitHub/arXiv/HN APIs
```

**Orchestrator (Opus):** scheduling/window logic, sub-agent dispatch, final synthesis (TL;DR, "Worth deeper research", "Emerging patterns" cross-category analysis), HTML rendering, Resend API delivery via Bash `curl`. Does NOT fetch sources directly.

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

**Parallelism fallback:** Whether the Task/Agent tool dispatches sub-agents in parallel from inside a `/schedule` remote session is not explicitly documented by Anthropic as of April 2026. Community reports confirm up to ~10 concurrent sub-agents from regular Claude Code sessions; `/schedule` sessions are described as "full Claude Code cloud sessions" which suggests parity. **If verification during initial deployment reveals Task is unavailable or single-threaded**, the orchestrator falls back to processing categories sequentially in a `for` loop over the same sub-agent prompts. Wall-clock time goes from ~2min to ~8min — still well inside `/schedule`'s budget and invisible to the end recipient. This fallback is handled in the orchestrator prompt, not as a separate code path.

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

### Fetch strategy per source type

Per research (verified 2026-04-24), different source types use different fetch strategies. Sub-agents are instructed in their prompts to choose the right one:

- **GitHub repos** → **REST API** (`api.github.com/repos/OWNER/REPO/releases` or `/commits?since=ISO8601`) via Bash `curl` with `GITHUB_TOKEN`. HTML fallback only if API is unreachable.
- **arXiv** → **Atom API** (`export.arxiv.org/api/query?search_query=cat:cs.CL+OR+cat:cs.AI+OR+cat:cs.IR+OR+cat:cs.LG&sortBy=submittedDate&sortOrder=descending&max_results=50`). HTML listings (`arxiv.org/list/cs.CL/new`) still work but the API returns cleaner structured data. arXiv publishes ~00:00 UTC Mon–Fri; our 05:30 UTC cron catches the fresh day.
- **Hacker News** → **Algolia API** (`hn.algolia.com/api/v1/search?tags=story&numericFilters=created_at_i>EPOCH,points>50&query=...`). Gives scores and timestamps for free.
- **Blogs / official news pages** → **WebFetch** (converts HTML to markdown, ~100KB cap per call).
- **Spanish public-sector sites** → **WebFetch** on `/actualidad` or `/news` index, then WebFetch each article URL. No RSS available on any of the three.

### Full category → source mapping

| # | Category | Primary sources (and fetch method) | Secondary (WebSearch) |
|---|----------|-----------------------------------|----------------------|
| 1 | Model releases & updates | anthropic.com/news (WebFetch), openai.com/news (WebFetch), deepmind.google/discover/blog (WebFetch), ai.meta.com/blog (WebFetch), mistral.ai/news (WebFetch), huggingface.co/blog (WebFetch), huggingface.co/models trending (WebFetch) | "Qwen release", "DeepSeek release", "open weights release" |
| 2 | Agents & multi-agent systems | GitHub REST API for: anthropics/claude-agent-sdk, langchain-ai/langgraph, crewAIInc/crewAI, microsoft/autogen, openai/openai-agents-python (releases endpoint) | simonwillison.net, latent.space, smol.ai |
| 3 | RAG | arXiv Atom API (cats cs.IR + cs.CL, sorted by submittedDate desc, max_results 50) | eugeneyan.com, hamel.dev, "graph RAG paper", "agentic RAG" |
| 4 | MCP | GitHub REST API for modelcontextprotocol/specification (releases) and modelcontextprotocol/servers (commits?since=). WebFetch modelcontextprotocol.io blog. | HN "mcp" Algolia query, Simon Willison |
| 5 | Claude Skills & Claude Code | WebFetch anthropic.com/news (filtered), WebFetch docs.claude.com/en/release-notes/claude-code | Community skill repos, r/ClaudeAI top of day |
| 6 | LLM gateways / proxies | GitHub REST API for BerriAI/litellm, Helicone/helicone, Portkey-AI/gateway (releases). WebFetch Kong AI Gateway changelog, Cloudflare AI Gateway changelog. | HN Algolia |
| 7 | Evaluation frameworks | GitHub REST API for confident-ai/deepeval, explodinggradients/ragas, promptfoo/promptfoo, UKGovernmentBEIS/inspect_ai. WebFetch Braintrust blog, LangSmith changelog. | arXiv Atom API with "LLM evaluation" keyword filter |
| 8 | LLM observability | GitHub REST API for langfuse/langfuse, Arize-ai/phoenix, traceloop/openllmetry. WebFetch LangSmith changelog, Helicone blog. | OTEL GenAI semconv updates |
| 9 | Self-hosting & deployment | GitHub REST API for vllm-project/vllm, sgl-project/sglang, huggingface/text-generation-inference, ollama/ollama | r/LocalLLaMA top of day, quantization news |
| 10 | AI platform / DevEx | (no single hub; lean on secondary) | latent.space, hamel.dev, huyenchip.com, "internal AI platform" HN Algolia |
| 11 | EU AI Act & regulation | WebFetch digital-strategy.ec.europa.eu/en/policies/ai-act, **aesia.digital.gob.es/es/actualidad** (Spanish, no RSS — scrape HTML), red.es/es/actualidad (Spanish), bsc.es/news (English) | "EU AI Act", "ENS certification AI", "Reglamento IA"; Spanish-language sources explicitly required |

**Noise rules baked into every sub-agent prompt:**
- Ignore hype threads, "X is dead" takes, pure vendor positioning.
- A vendor blog post is only worth including if it describes new functionality, not marketing positioning.
- Prefer papers and production postmortems over vendor blogs for RAG.
- For category 11, always include Spanish-language sources (EU AI Act implementation often lands in national languages first). Summarize Spanish content in English for the digest body.

## Email Template

### Subject line
```
[AI Brief] Fri 2026-04-24 — 14 items across 9 categories
```
Scannable preview, sortable by date, item count reflects slow vs busy days.

### Required email headers

Resend sets the following via API parameters or auto-applies. All are required for 2026 Gmail deliverability (Gmail's bulk-sender rules from Feb 2024 are effectively required even at low volume):

- `From: "AI Brief" <brief@ultrapowers.dev>` (verified domain in the Resend account; DKIM/DMARC aligned automatically)
- `To: techno1731@gmail.com`
- `Reply-To: techno1731@gmail.com`
- `List-Unsubscribe: <mailto:techno1731@gmail.com?subject=unsubscribe>`
- `List-Unsubscribe-Post: List-Unsubscribe=One-Click`
- SPF + DKIM auto-applied by Resend on their shared domain; DMARC alignment achieved via Resend's default config. Not the implementer's problem in v1.

### HTML structure

Mobile-first (max-width 600px, centered). Inline CSS on all critical style-dependent elements (belt-and-braces: Gmail-to-Gmail does preserve `<style>` in `<head>`, but we're sending via Resend's shared domain so we play safe). `<style>` in `<head>` may be used for structural reset rules only, kept under 8KB, with no nested `@import` or `@font-face` inside `@media`. No images, no web fonts, no tracking pixels.

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
- Headers: `-apple-system, Segoe UI, Roboto, Arial, sans-serif` (Gmail-safe stack).
- Body: `Georgia, 'Times New Roman', serif`.
- Text: `#1a1a1a` near-black (not pure `#000`; survives Gmail mobile's partial-invert).
- Background: `#fafafa` (not pure `#fff`; same reason).
- Secondary: `#666` grey.
- Accent (links, Sapiens angle): `#c14500` warm rust — saturated mid-luminance, confirmed to survive Gmail mobile dark-mode partial-invert.
- Include both meta tags for dark-mode-aware clients (Gmail itself does not honor them, but no-cost signal to Apple Mail etc.):
  ```html
  <meta name="color-scheme" content="light dark">
  <meta name="supported-color-schemes" content="light dark">
  ```

### Rendering
The orchestrator holds the HTML skeleton in its prompt (embedded in config.yaml's `email_template` key) and fills it with the collected items. Template is plain string substitution — no Jinja, Mustache, or dependencies. The orchestrator writes the final HTML to a temp file, then Bash-curls it to Resend:

```bash
curl -X POST https://api.resend.com/emails \
  -H "Authorization: Bearer $RESEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/digest-payload.json
```

## Failure Handling

| Failure | Behavior |
|---|---|
| One sub-agent throws / times out / returns malformed JSON | Category shown as `[fetch failed this run — check trace]`. Other categories unaffected. Error logged in "Run notes" footer. |
| `thresholds.degraded_run_category_failures` or more sub-agents fail (default: 6) | Email still sent, subject prefixed `[DEGRADED RUN]`. Signals investigation needed rather than silently delivering thin digests. Threshold is tunable in config.yaml. |
| Resend API send fails (non-2xx, network error) | Orchestrator logs the full rendered HTML to its stdout so it's recoverable from the `/schedule` trace. **One retry with 5s backoff**, then give up. No queue. `/schedule` run is marked succeeded so the next day's run proceeds normally. |
| Resend API rejects the payload (4xx validation) | Log the error body. Do not retry. User investigates via `/schedule` trace. |
| Orchestrator itself crashes before finishing | `/schedule` surfaces the failure in its UI. User notices no email by 09:00 and investigates manually. Acceptable for v1. |
| Zero categories had news (rare, e.g., holiday) | Send a short "Quiet day — nothing notable across all categories" email. Keeps delivery cadence predictable — silence would create ambiguity about whether the cron broke. |
| `RESEND_API_KEY` env var unset | Orchestrator aborts before sub-agent dispatch, logs loud error. This is a setup bug, not a runtime condition. |

**Observability in v1:** the only log retention is whatever `/schedule` keeps in its run history, plus Resend's own dashboard (shows delivered/bounced/opened per message). There is no structured logging, no Langfuse trace, no archived HTML. If something goes wrong past the `/schedule` retention window, it's gone. Acceptable for v1 — structured observability is a v2 feature when this migrates to K8s.

## Project Layout

```
daily-ai-brief/
├── README.md                   ← how to deploy + iterate
├── routine-prompt.md           ← thin shim pasted into `/schedule create`
├── config.yaml                 ← all knobs (sources, sub-agent prompt template,
│                                  email template, category prompts)
├── docs/ultrapowers/
│   ├── specs/
│   │   └── 2026-04-24-daily-ai-brief-design.md      ← this file
│   └── research/
│       └── 2026-04-24-research-brief.md             ← findings that shaped v1
├── .claude/
│   └── ultrapowers-preferences.json
└── .gitignore
```

**Key architectural split:**
- `routine-prompt.md` (~40 lines) is frozen after initial deployment. Tells the orchestrator to fetch config.yaml and execute per its instructions.
- `config.yaml` is where day-to-day tuning happens. Editing it is a normal git workflow. Next run picks up the change automatically.

**Repo visibility:** public. Simpler (no PAT needed for `raw.githubusercontent.com` WebFetch of config.yaml), and the repo doubles as reference material for Sapiens Lab customers.

## Deployment Plan

**First-time setup (one-off, ~25 min):**

1. `git init` locally, scaffold files, commit.
2. Create `github.com/techno1731/daily-ai-brief` (public), push.
3. Resend account: API key from `ultrapowers.dev`-verified account (reused from existing ultrapowers-web Vercel prod env). Verified domain means unrestricted recipients and DKIM/DMARC-aligned delivery.
4. (Optional but recommended) Create a GitHub Personal Access Token with `public_repo` scope (read-only is sufficient since all our GitHub sources are public). Raises unauth rate limit from 60/hr to 5000/hr.
5. Run `/schedule create`, paste contents of `routine-prompt.md`. Set cron `30 7 * * 1-5`, timezone `Europe/Madrid`. Add env vars:
   - `RESEND_API_KEY=re_...` (required)
   - `GITHUB_TOKEN=ghp_...` (optional)
6. Trigger a manual run from the `/schedule` UI ("Run now"). Validate end-to-end: email arrives at techno1731@gmail.com, formatting correct, all categories populate, Resend dashboard shows delivery.
7. Gmail placement training: **star the first digest** or drag to Primary tab if it lands in Promotions. Gmail learns per-sender from this one action; subsequent digests go straight to Primary.
8. Once validated, let it run autonomously from the next weekday 07:30.

**Iteration flow (ongoing):**
- Add/remove sources, tweak sub-agent prompts, adjust email styling → edit `config.yaml` → commit → push. No routine redeploy.
- Change orchestrator-level behavior (e.g., failure thresholds, fetch-strategy logic) → edit `routine-prompt.md` → `/schedule update`. Expected to be rare.

## v2 Considerations (deferred)

Logged here so they don't clutter v1 but aren't lost:

<!-- Custom sending domain (brief@ultrapowers.dev) was promoted from v1.5 to v1 during implementation when the existing Resend account was found to have ultrapowers.dev already verified. Future: if Sapiens Lab gets its own domain, switch From to brief@sapiens.lab to make this portfolio-visible under that brand. -->
- **Langfuse tracing** — port to K8s-hosted runner so traces flow to self-hosted Langfuse. Required before this is a Sapiens Lab reference implementation, not just personal tooling.
- **Persisted since-last-run state** — gist or tiny repo storing last-successful-run timestamp.
- **Heartbeat** — separate cron that checks whether the 07:30 email arrived, pings user if not.
- **Local dry-run script** — run the orchestrator prompt against local Claude Code without sending email, for prompt tuning.
- **Dedup against recent digests** — avoid re-featuring items from the previous 3 days.
- **Archive page** — static site built from committed daily digests.
- **DeepEval LLM-as-judge regression tests** — evaluate synthesis quality against a held-out "gold digest" to catch prompt regressions.

## Open questions (resolved during design and research)

- Execution shape: `/schedule` routine, delivery via Resend API (Gmail MCP rejected — draft-only, no send). ✅
- Source strategy: hybrid (structured primary + WebSearch long-tail), with API-first for GitHub/arXiv/HN. ✅
- Cadence: daily 07:30 Europe/Madrid weekdays, 72h window on Monday, no persisted state. ✅
- Architecture: orchestrator + 11 parallel Sonnet sub-agents, with sequential fallback if Task tool parallelism is unavailable inside `/schedule`. ✅
- Email format: well-designed HTML, Gmail-safe palette, List-Unsubscribe + DMARC (via Resend). ✅
- Project home: `/Users/techno1731/Code/Personal/daily-ai-brief`. ✅
- Workflow prefs: auto-commit on, auto-push off, commit design docs on. ✅
- AESIA URL: `aesia.digital.gob.es/es` (not `aesia.gob.es`, which doesn't exist). ✅
- Sending domain: `brief@ultrapowers.dev` — verified during implementation using the existing Resend account from ultrapowers-web (promoted from v1.5 to v1 since the domain was already DKIM-ready). ✅
