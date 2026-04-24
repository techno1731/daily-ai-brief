# Research Brief — Daily AI Brief v1

**Date:** 2026-04-24
**Driven by spec:** `docs/ultrapowers/specs/2026-04-24-daily-ai-brief-design.md`
**Research scope:** Claude Code `/schedule`, Gmail MCP, Agent-tool-in-scheduled-session, Resend, Gmail HTML email 2026, arXiv/GitHub/HN fetch strategies, Spanish public-sector source URLs.

## Context

The brainstorming-approved spec specified delivery via Anthropic's hosted Gmail MCP and an orchestrator-plus-parallel-sub-agents execution pattern inside a `/schedule` routine. Before implementation planning, research validated both assumptions against current documented capabilities and community reports. Two findings required v1 architecture revisions; several others refined source-fetch strategies.

## Key findings

### 1. Anthropic's hosted Gmail MCP is draft-only — rejected for v1

**Finding:** The hosted Gmail connector exposed via `mcp__claude_ai_Gmail__*` exposes `create_draft`, `search_messages`, `read_message`, `read_thread`, `list_labels`, and the two auth tools. **There is no `send_email` tool.** Anthropic's help center confirms: *"Claude only reads emails and creates drafts with your explicit approval. The send function is not enabled — all emails must be sent manually through your Gmail account."* Open feature request: `anthropics/claude-code#28575` (Feb 2026, no Anthropic reply).

Additionally: OAuth tokens are held per-process in-memory. Even if send existed, sub-agents or scheduled remote sessions would not inherit a parent's auth — they'd start with only `authenticate` visible.

**Sources:**
- https://support.claude.com/en/articles/10166901-use-google-workspace-connectors
- https://github.com/anthropics/claude-code/issues/28575
- https://github.com/anthropics/claude-code/issues/46228 (auth persistence)

**Decision:** v1 uses **Resend** (transactional email API) via Bash `curl` from the orchestrator. Chosen over self-hosted Gmail MCP (`GongRzhe/Gmail-MCP-Server`) and direct Gmail REST API for setup simplicity and unattended reliability.

### 2. Agent/Task tool inside `/schedule` — assumed available, sequential fallback specified

**Finding:** Anthropic's docs describe scheduled routines as "full Claude Code cloud sessions" with "all connected MCP connectors included by default," strongly suggesting tool parity with interactive sessions. However, there is **no explicit documented confirmation** that the Task tool dispatches sub-agents successfully inside a scheduled remote session. Community reports of ~10 concurrent sub-agent max are from regular Claude Code, not routines.

**Sources:**
- https://code.claude.com/docs/en/routines
- https://code.claude.com/docs/en/scheduled-tasks
- https://www.mindstudio.ai/blog/claude-code-agent-teams-parallel-agents

**Decision:** v1 attempts parallel dispatch (best case: ~2 min runtime). If the first `/schedule` trial run shows Task is unavailable or single-threaded, the orchestrator prompt includes a fallback branch that processes categories sequentially (~8 min runtime). Either way, the end user receives the same email by 07:35 Europe/Madrid. Fallback is an in-prompt conditional, not a separate code path.

### 3. arXiv Atom API preferred over HTML listing

**Finding:** `arxiv.org/list/cs.CL/new` returns usable HTML (~570 KB, `<dt>`/`<dd>` entries) and WebFetch parses it cleanly. However, the Atom API at `export.arxiv.org/api/query?search_query=cat:cs.CL+OR+cat:cs.AI+OR+cat:cs.IR+OR+cat:cs.LG&sortBy=submittedDate&sortOrder=descending` returns structured `<entry>` elements per paper — much easier for an LLM to reliably extract title/abstract/ID. arXiv's API ToS mandates using `export.arxiv.org` (not `arxiv.org`) for programmatic access.

**Publish cadence gotcha:** arXiv announces daily papers around 00:00 UTC Mon–Fri, no weekend announcements. The v1 cron at 07:30 Europe/Madrid (≈05:30 UTC Tue–Fri, 06:30 CEST in summer) catches the fresh batch. Monday runs pull Friday+weekend via the 72h window.

**Sources:**
- https://info.arxiv.org/help/api/index.html

**Decision:** sub-agent for category 3 (RAG) uses the Atom API. Category 7 (eval) uses a keyword-filtered Atom query as its secondary source.

### 4. Spanish public-sector site URLs — one correction, all need HTML scraping

**Finding:** The spec originally listed `aesia.gob.es` — this domain does not resolve (NXDOMAIN). The correct canonical URL is **`aesia.digital.gob.es/es`** (launched Feb 2025, HQ La Coruña). News index at `/es/actualidad`, Spanish-default, EN/CA/VA/GL/EU available.

`red.es/es/actualidad` and `bsc.es/news` both exist and render server-side, but **none of the three publish an RSS or Atom feed**. All three require HTML scraping via WebFetch, then article-page WebFetch for substance. AESIA and Red.es content is Spanish; BSC publishes in English by default.

**Sources:**
- https://aesia.digital.gob.es/es (verified 2026-04-24)
- https://www.red.es/es (verified)
- https://www.bsc.es/news (verified)

**Decision:** spec updated with correct URL. Category 11 sub-agent prompt explicitly instructs: summarize Spanish-language content in English for the digest body.

### 5. GitHub REST API preferred over HTML; HN Algolia API preferred over front-page scraping

**Finding:** `github.com/owner/repo/releases` renders enough in initial HTML for WebFetch to extract recent release titles and bodies, but the REST API (`api.github.com/repos/OWNER/REPO/releases?per_page=10`) returns clean JSON. Unauth rate limit is 60/hr — tight if any per-commit enrichment is added later. Authenticated (PAT, `public_repo` scope) raises this to 5000/hr.

HN front page at `news.ycombinator.com` WebFetches cleanly, but the Algolia API (`hn.algolia.com/api/v1/search?tags=story&numericFilters=created_at_i>{epoch-86400},points>50`) returns scored, timestamped results in one call — no need to parse HTML or visit individual threads to get signal.

**Sources:**
- https://docs.github.com/en/rest/releases/releases
- https://hn.algolia.com/api

**Decision:** `GITHUB_TOKEN` added to the routine environment (recommended, not required). Sub-agents for categories 2, 4, 6, 7, 8, 9 use the GitHub REST API. Category 4's secondary HN query uses Algolia.

### 6. Gmail-safe HTML email in 2026 — DMARC-alignment now effectively mandatory

**Finding:** Gmail's bulk-sender rules (Feb 2024) are still enforced in 2026 and effectively apply even at low volume: SPF + DKIM aligned with a DMARC record (even `p=none`) is required for reliable Primary-tab placement. `List-Unsubscribe` + `List-Unsubscribe-Post: List-Unsubscribe=One-Click` are deliverability signals (not just compliance).

- 600px max-width remains standard.
- Gmail (web + mobile) does NOT honor `@media (prefers-color-scheme: dark)`. Design a single palette that survives Gmail mobile's partial-invert (light bgs flipped, darks left, mid-saturated hues preserved). `#c14500` warm rust is confirmed dark-mode safe. Avoid pure `#000`/`#fff` block fills — use `#1a1a1a` text and `#fafafa` background.
- `<style>` in `<head>` IS preserved for Gmail-to-Gmail sends, but entire block is stripped if >8KB or contains nested `@import`/`@font-face` inside `@media`. Since Resend sends from `resend.dev` (not Gmail-to-Gmail), inline critical styles regardless.
- Only web fonts that render natively in Gmail are Google Sans and Roboto. Georgia + system-sans stack are the safe picks.
- User action (star/reply/drag-to-Primary) trains Gmail's per-sender classifier — a one-time nudge after the first delivery is the fastest fix for Promotions-tab misclassification.

**Sources:**
- https://developers.google.com/workspace/gmail/design/css
- https://support.google.com/a/answer/81126
- https://www.litmus.com/blog/coding-emails-for-dark-mode
- https://www.litmus.com/blog/the-ultimate-guide-to-list-unsubscribe

**Decision:** Resend's shared sender (`onboarding@resend.dev`) has SPF/DKIM/DMARC pre-configured. v1 ships with `List-Unsubscribe` headers set via Resend's API parameters. Palette locked to `#1a1a1a` / `#fafafa` / `#c14500` / `#666`. Deliverability will be validated on the first test run; if Promotions-tab placement is persistent, the star-then-move user action is the documented fix.

### 7. `/schedule` capabilities confirmed; some limits remain undocumented

**Finding:**
- Runs on Anthropic-managed cloud infrastructure ✅ (laptop can be closed)
- Standard 5-field cron, **minimum interval 1 hour** (sub-hourly rejected)
- Timezones handled via local-to-cron conversion; `Europe/Madrid` is supported
- Billed against the user's subscription tier (Pro 5 runs/day, Max 15/day) — no separate metered API bill layered on
- Manual "Run now" from the routine detail page for testing ✅
- Routine environment supports env vars (for `RESEND_API_KEY`, `GITHUB_TOKEN`)
- All connected MCP connectors auto-included by default
- **Max runtime per execution** and **prompt size limit**: not publicly documented
- **Log retention window**: not publicly documented

**Sources:**
- https://code.claude.com/docs/en/routines
- https://code.claude.com/docs/en/scheduled-tasks

**Decision:** 1-hour minimum interval is not a problem for a daily cron. Undocumented runtime/log-retention limits are an acceptable v1 unknown — the orchestrator's stdout logs are the only observability surface, and if they're truncated we'll find out during the first run and address in v2 with Langfuse tracing.

## Changes applied to spec

- **Delivery**: Gmail MCP → Resend via Bash `curl`.
- **Environment variables**: added `RESEND_API_KEY` (required) and `GITHUB_TOKEN` (recommended).
- **arXiv**: HTML listings → Atom API (`export.arxiv.org/api/query`).
- **AESIA URL**: `aesia.gob.es` → `aesia.digital.gob.es/es/actualidad`.
- **GitHub sources**: HTML scraping → REST API (`api.github.com/repos/OWNER/REPO/releases`).
- **HN**: front-page scraping → Algolia API.
- **Email headers**: `List-Unsubscribe` + `List-Unsubscribe-Post: List-Unsubscribe=One-Click` required.
- **Palette**: confirmed `#1a1a1a` / `#fafafa` / `#c14500`; added `color-scheme` meta tags.
- **Parallelism**: orchestrator prompt includes a sequential fallback for the case where Task-tool parallelism doesn't work inside `/schedule`.
- **Deployment**: setup flow replaces Gmail MCP auth with Resend signup + API key; adds Gmail placement-training nudge after first delivery.
- **v2 backlog**: added "custom sending domain (v1.5)".

## Open risks that survive into implementation

1. **Task-tool parallelism inside `/schedule` not documentation-confirmed.** Mitigation: in-prompt sequential fallback. Validated on first run.
2. **`/schedule` max runtime undocumented.** Mitigation: sequential mode's ~8-min runtime is well inside any plausible ceiling for a Max-tier scheduled task. Validated on first run.
3. **Gmail Promotions-tab placement on first delivery.** Mitigation: documented manual nudge (star/move to Primary) in the deployment flow.
4. **Resend's shared sender (`onboarding@resend.dev`) may show as "via resend.dev" in Gmail.** Cosmetic for v1; resolved in v1.5 with a verified custom domain.

## Sources

Full source set consolidated from four parallel research agents dispatched 2026-04-24:

- **Claude Code / /schedule / Task tool / WebFetch**: code.claude.com/docs, platform.claude.com/docs, mindstudio.ai reports
- **Gmail MCP / alternatives**: support.claude.com, anthropics/claude-code GitHub issues, developers.google.com/gmail/api, GongRzhe/Gmail-MCP-Server, composio.dev, resend.com
- **Gmail HTML email 2026**: Litmus, Email on Acid, Mailtrap, developers.google.com/workspace/gmail/design/css, support.google.com/a/answer/81126
- **arXiv / GitHub / HN / Spanish gov sites**: direct URL verification 2026-04-24 against info.arxiv.org, api.github.com, hn.algolia.com, aesia.digital.gob.es, red.es, bsc.es
