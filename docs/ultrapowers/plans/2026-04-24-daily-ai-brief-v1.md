# Daily AI Brief v1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use ultrapowers:subagent-driven-development (recommended) or ultrapowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a daily AI intelligence briefing agent that runs via Claude Code `/schedule`, dispatches 11 parallel sub-agents per category, and emails a well-designed HTML digest to `techno1731@gmail.com` via Resend.

**Architecture:** Thin `routine-prompt.md` shim pasted into `/schedule`. At run time, the shim fetches a fat `config.yaml` + `templates/email.html` from GitHub raw, then follows the `orchestrator_instructions` block in the YAML verbatim. Iteration is `git push`, not routine redeploy.

**Tech Stack:** Claude Code `/schedule`; Claude Opus (orchestrator) + Claude Sonnet (sub-agents); Task tool for parallel dispatch (sequential fallback); Bash + `curl` for Resend / GitHub REST / arXiv Atom / HN Algolia APIs; YAML config; single-file HTML email template.

**Skills to invoke during execution:**
- `ultrapowers:verification-before-completion` — before claiming any task complete
- `ultrapowers:dispatching-parallel-agents` — reference for Task tool usage shape
- `ultrapowers-dev:orchestrator-workers` — architecture pattern for the orchestrator prompt
- `ultrapowers-dev:parallelization` — parallel sub-agent dispatch
- `ultrapowers-dev:tool-calling` — structured JSON returns from sub-agents
- `ultrapowers-dev:background-jobs` — scheduled cron patterns
- `ultrapowers-dev:error-handling` — failure isolation + degraded runs
- `ultrapowers-dev:resilience` — Resend retry policy
- `ultrapowers-dev:auth-security` — API key hygiene in `/schedule` env vars

**Before starting:** Read the spec at `docs/ultrapowers/specs/2026-04-24-daily-ai-brief-design.md` and the research brief at `docs/ultrapowers/research/2026-04-24-research-brief.md`. They're the source of truth; this plan operationalizes them.

**TDD note:** This is a prompt-engineering project, not a traditional code project. There is no unit test suite — "tests" are (a) YAML/HTML parse checks, (b) manual dry-run of the orchestrator in an interactive Claude Code session, and (c) the first live `/schedule` run inspected end-to-end. Each file-creation task ends with a verification step that matches this reality.

**Commit policy:** Auto-commit is ON (per `.claude/ultrapowers-preferences.json`). Design docs are committable (spec, research, plan). Auto-push is OFF — the engineer pushes manually after reviewing. `.gitignore` excludes secrets.

---

## Task 1: Scaffold `config.yaml` skeleton

**Files:**
- Create: `config.yaml`

**Goal:** lock in the top-level schema so subsequent tasks fill in known slots. No real content yet.

- [ ] **Step 1: Create `config.yaml` with the following content**

```yaml
# Daily AI Brief — runtime configuration
# This file is WebFetched at the start of every /schedule run.
# Iterate freely: git commit + push. Next run uses the new config.

thresholds:
  degraded_run_category_failures: 6
  max_items_per_category: 5

meta:
  from_address: "AI Brief <onboarding@resend.dev>"
  to_address: "techno1731@gmail.com"
  reply_to: "techno1731@gmail.com"
  list_unsubscribe_mailto: "mailto:techno1731@gmail.com?subject=unsubscribe"
  subject_pattern: "[AI Brief] {{DAY}} {{YYYY_MM_DD}} — {{TOTAL_ITEMS}} items across {{ACTIVE_CATEGORY_COUNT}} categories"
  subject_pattern_degraded: "[DEGRADED RUN] [AI Brief] {{DAY}} {{YYYY_MM_DD}} — {{TOTAL_ITEMS}} items across {{ACTIVE_CATEGORY_COUNT}} categories"
  subject_pattern_quiet: "[AI Brief] {{DAY}} {{YYYY_MM_DD}} — Quiet day, nothing notable"

# Step-by-step instructions the orchestrator follows verbatim.
# Filled in by Task 4.
orchestrator_instructions: |
  (TBD — filled in by Task 4)

# Prompt template rendered once per category and dispatched to a sub-agent.
# Filled in by Task 5.
sub_agent_prompt_template: |
  (TBD — filled in by Task 5)

# 11 categories. Filled in by Tasks 6, 7, 8.
categories: []
```

- [ ] **Step 2: Verify YAML parses**

Run from the project root:
```bash
python3 -c "import yaml; yaml.safe_load(open('config.yaml')); print('ok')"
```
Expected: `ok`. If `yaml` module missing, `python3 -m pip install pyyaml` first, or use `pip install --user pyyaml`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Add config.yaml skeleton with thresholds and meta"
```

---

## Task 2: Write the thin routine prompt

**Files:**
- Create: `routine-prompt.md`

**Goal:** the ~10-line shim that gets pasted into `/schedule create`. It is intentionally trivial so it never needs to change.

- [ ] **Step 1: Create `routine-prompt.md` with this exact content**

```markdown
# Daily AI Brief — Routine Prompt

You are the Daily AI Brief orchestrator, running inside a Claude Code `/schedule` routine. This prompt is intentionally minimal. The real configuration lives in `config.yaml` in the project repo and is fetched fresh on every run, so the operator can iterate without redeploying the routine.

## Runtime expectations

- Environment variables expected:
  - `RESEND_API_KEY` (required)
  - `GITHUB_TOKEN` (recommended; raises GitHub REST rate limit from 60/hr to 5000/hr)
- Current UTC clock is available via the system.

## What to do on every run

1. WebFetch `https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/config.yaml`. Parse as YAML.
2. WebFetch `https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/templates/email.html`. Hold as a string.
3. Execute the `orchestrator_instructions` block from the config **verbatim**. Treat it as your full set of instructions for this run.
4. When that block finishes (or fails gracefully), the routine is done. Log a one-line summary to stdout.

Do not do anything else. If a step fails, surface the error in the stdout log so the operator can see it in the `/schedule` trace.
```

- [ ] **Step 2: Visual-inspect for correctness**

Read the file. Confirm it: (a) references the correct raw GitHub URL (`techno1731/daily-ai-brief/main`), (b) mentions both env vars, (c) references both `config.yaml` and `templates/email.html`.

- [ ] **Step 3: Commit**

```bash
git add routine-prompt.md
git commit -m "Add thin routine-prompt.md shim"
```

---

## Task 3: Write the email HTML template

**Files:**
- Create: `templates/email.html`

**Goal:** the full HTML skeleton with `{{PLACEHOLDERS}}` that the orchestrator fills at render time. All styles inline per Gmail safety; palette and font stack locked to spec.

- [ ] **Step 1: Create `templates/` directory and `email.html`**

```bash
mkdir -p templates
```

- [ ] **Step 2: Create `templates/email.html` with this exact content**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="light dark">
<meta name="supported-color-schemes" content="light dark">
<title>Daily AI Brief</title>
</head>
<body style="margin:0;padding:0;background:#fafafa;">
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:#fafafa;">
  <tr>
    <td align="center" style="padding:24px 12px;">
      <table role="presentation" width="600" cellpadding="0" cellspacing="0" border="0" style="max-width:600px;width:100%;background:#fafafa;">

        <!-- Header strip -->
        <tr>
          <td style="font-family:-apple-system,'Segoe UI',Roboto,Arial,sans-serif;font-size:13px;color:#666;padding:0 0 16px 0;letter-spacing:0.5px;">
            Daily AI Brief &mdash; {{DATE_HUMAN}}
          </td>
        </tr>

        <!-- TL;DR callout -->
        <tr>
          <td style="border:1px solid #c14500;padding:16px 18px;background:#ffffff;">
            <div style="font-family:-apple-system,'Segoe UI',Roboto,Arial,sans-serif;font-size:12px;color:#c14500;text-transform:uppercase;letter-spacing:1px;margin-bottom:8px;font-weight:bold;">TL;DR</div>
            <div style="font-family:Georgia,'Times New Roman',serif;font-size:15px;line-height:1.55;color:#1a1a1a;">
              {{TL_DR}}
            </div>
          </td>
        </tr>

        <!-- Category sections -->
        <tr>
          <td style="padding:24px 0 0 0;">
            {{SECTIONS}}
          </td>
        </tr>

        <!-- Worth deeper research (omitted by orchestrator if empty) -->
        {{DEEP_RESEARCH_BLOCK}}

        <!-- Emerging patterns (omitted by orchestrator if empty) -->
        {{EMERGING_PATTERNS_BLOCK}}

        <!-- Run notes (omitted by orchestrator if no failures) -->
        {{RUN_NOTES_BLOCK}}

      </table>
    </td>
  </tr>
</table>
</body>
</html>
```

- [ ] **Step 3: Verify the HTML parses and renders**

Open the file in a local browser:
```bash
open templates/email.html
```
Expected: the page renders (placeholders visible literally — that's fine), no unclosed tags, no error in browser dev console.

Additionally, run a basic HTML syntax check via Python:
```bash
python3 -c "from html.parser import HTMLParser; p = HTMLParser(); p.feed(open('templates/email.html').read()); print('ok')"
```
Expected: `ok`.

- [ ] **Step 4: Commit**

```bash
git add templates/email.html
git commit -m "Add HTML email template with Gmail-safe palette and inline CSS"
```

---

## Task 4: Write `orchestrator_instructions` in config.yaml

**Files:**
- Modify: `config.yaml` (fill the `orchestrator_instructions` block)

**Goal:** the 9-step procedure the orchestrator follows on every run. This is where most of the runtime logic lives.

- [ ] **Step 1: Replace the `orchestrator_instructions` placeholder with this exact block**

In `config.yaml`, replace:
```yaml
orchestrator_instructions: |
  (TBD — filled in by Task 4)
```
with:
```yaml
orchestrator_instructions: |
  You are the Daily AI Brief orchestrator. Follow these steps in order.

  ## Step 1: Establish run context
  - Read the current UTC date and time.
  - Determine local day of week in Europe/Madrid.
  - If Saturday or Sunday, abort immediately — the cron should not have fired on a weekend. Log and exit.
  - Verify `RESEND_API_KEY` is set in the environment. If not, abort with a clear error message.

  ## Step 2: Compute the lookback window
  - Monday  → window_hours = 72
  - Tue–Fri → window_hours = 24
  - window_start_iso = (now_utc - window_hours) as ISO 8601 with Z suffix.
  - Remember this value; every sub-agent will receive it.

  ## Step 3: Dispatch sub-agents
  For each entry in `categories`:
  - Render the sub-agent prompt by substituting into `sub_agent_prompt_template`:
      {{CATEGORY_ID}}         ← categories[i].id
      {{CATEGORY_NAME}}       ← categories[i].name
      {{PRIMARY_SOURCES}}     ← rendered bullet list of categories[i].primary
      {{SECONDARY_SEARCHES}}  ← rendered bullet list of categories[i].secondary
      {{FETCH_STRATEGY}}      ← categories[i].fetch_strategy
      {{WINDOW_START_ISO}}    ← computed in Step 2
      {{MAX_ITEMS}}           ← thresholds.max_items_per_category
  - Dispatch the sub-agent via the Task / Agent tool with subagent_type "general-purpose" and model preference "sonnet".

  **Attempt parallel dispatch first:** emit all 11 Task tool calls in a single response. If the platform rejects parallel dispatch or executes them serially, fall back to processing categories sequentially one at a time — the schema of each sub-agent's return value is the same either way. Do not block the run on parallelism — take whichever the platform gives.

  ## Step 4: Collect and validate results
  For each sub-agent return:
  - Attempt to parse as JSON matching the documented schema (category, nothing_notable, items, errors).
  - If JSON is malformed, record the category as failed with reason "malformed JSON" and continue to the next.
  - Canonical empty: `nothing_notable: true` with `items: []`. Treat any other combination (e.g., `nothing_notable: false` with `items: []`) as a sub-agent bug — mark the category failed.
  - Count failed categories. If failed_count >= thresholds.degraded_run_category_failures, set degraded = true.

  ## Step 5: Synthesize the meta-sections
  Using only the successful items across all categories, write:
  - **TL;DR**: one paragraph (3-5 sentences) naming the 3 most important things across all categories. No preamble, no "In today's rapidly evolving AI landscape" style openers. Senior engineer to senior engineer.
  - **Worth deeper research this week**: 1-3 items that warrant a dedicated deep-dive, each with a one-line "why". If you can't find 1 that genuinely warrants a deep-dive, omit the entire section.
  - **Emerging patterns**: any cross-cutting themes (e.g., "everyone shipping agent memory layers this week"). If you can't name a genuine pattern, omit the entire section.

  Do not fabricate. Do not pad. If there is nothing to say, say nothing.

  ## Step 6: Render the HTML body
  Start with the `templates/email.html` string you WebFetched.

  Substitute:
  - `{{DATE_HUMAN}}` → today in the form "Fri Apr 24 2026" (UTC date is fine).
  - `{{TL_DR}}` → the TL;DR paragraph.
  - `{{SECTIONS}}` → rendered HTML per category. For each category:
      * If nothing_notable: skip the category entirely.
      * If failed: render the category header plus one line: `<div style="font-family:Georgia,'Times New Roman',serif;font-size:14px;color:#666;padding:10px 0;">[fetch failed this run — check trace]</div>`
      * Otherwise: render the category header + each item.
  - `{{DEEP_RESEARCH_BLOCK}}` → if there's deep-research content, render the whole `<tr>…</tr>` block including header; else render the empty string.
  - `{{EMERGING_PATTERNS_BLOCK}}` → same rule.
  - `{{RUN_NOTES_BLOCK}}` → if any category failed, render a block with header "Run notes" and a short sentence listing the failed categories; else empty string.

  ### Category header HTML
  ```
  <div style="margin-bottom:24px;">
    <div style="font-family:-apple-system,'Segoe UI',Roboto,Arial,sans-serif;font-size:13px;color:#1a1a1a;text-transform:uppercase;letter-spacing:1px;padding:8px 0;border-bottom:1px solid #1a1a1a;font-weight:bold;">{{CATEGORY_NAME}}</div>
    {{ITEMS_HTML}}
  </div>
  ```

  ### Item HTML
  ```
  <div style="padding:14px 0;border-bottom:1px solid #eeeeee;">
    <div style="font-family:-apple-system,'Segoe UI',Roboto,Arial,sans-serif;font-size:16px;font-weight:bold;color:#1a1a1a;margin-bottom:6px;">{{HEADLINE}}</div>
    <div style="font-family:Georgia,'Times New Roman',serif;font-size:15px;line-height:1.55;color:#1a1a1a;margin-bottom:6px;">{{SUMMARY}}</div>
    {{SAPIENS_ANGLE_BLOCK}}
    <a href="{{URL}}" style="font-family:-apple-system,'Segoe UI',Roboto,Arial,sans-serif;font-size:13px;color:#c14500;text-decoration:underline;">{{URL_SHORT}} &#8599;</a>
  </div>
  ```

  ### Sapiens-angle block (render only if `sapiens_angle` is non-null)
  ```
  <div style="font-family:Georgia,'Times New Roman',serif;font-size:14px;line-height:1.5;color:#c14500;font-style:italic;margin-bottom:6px;">&rarr; For Sapiens Lab: {{SAPIENS_ANGLE}}</div>
  ```

  `{{URL_SHORT}}` is the host + first path segment of the full URL (e.g., `anthropic.com/news`). This is for display only; the `href` uses the full URL.

  ### Wrapping blocks for deep research / emerging patterns / run notes

  Each of these is rendered as a full `<tr><td>…</td></tr>` row with the same top border treatment as the other sections (see the rows in templates/email.html for reference). If a section is omitted, emit the empty string in its placeholder — do NOT emit an empty `<tr>`.

  ## Step 7: Compute the subject line
  - total_items = sum of items across non-failed, non-empty categories.
  - active_category_count = number of non-empty, non-failed categories.
  - day = three-letter abbreviation (Mon, Tue, ...).
  - yyyy_mm_dd = today in ISO date format.
  - If total_items == 0: use meta.subject_pattern_quiet.
  - Else if degraded: use meta.subject_pattern_degraded.
  - Else: use meta.subject_pattern.

  ## Step 8: Send via Resend
  Write the JSON payload to /tmp/digest-payload.json. The JSON must include:
  - from: meta.from_address
  - to: [meta.to_address]
  - reply_to: meta.reply_to
  - subject: (computed in Step 7)
  - html: the rendered HTML from Step 6 (as a JSON string — escape correctly)
  - headers: { "List-Unsubscribe": "<" + meta.list_unsubscribe_mailto + ">", "List-Unsubscribe-Post": "List-Unsubscribe=One-Click" }

  Then run via Bash:

  ```bash
  curl -sS -w "\nHTTP %{http_code}\n" -X POST https://api.resend.com/emails \
    -H "Authorization: Bearer $RESEND_API_KEY" \
    -H "Content-Type: application/json" \
    -d @/tmp/digest-payload.json
  ```

  ### Retry policy
  - HTTP 2xx → success. Extract the `id` field from the response body and log it. If you have the full response body in a shell variable or file, use `python3 -c 'import json,sys; print(json.load(sys.stdin).get("id",""))'` or `jq -r .id` — not regex on the raw JSON.
  - HTTP 4xx → payload is wrong. Log the response body. Do NOT retry.
  - HTTP 5xx or network error → sleep 5 seconds and retry exactly once. If the retry also fails, log loudly and proceed to Step 9 anyway.

  ## Step 9: Log and exit
  Emit a single-block run summary to stdout:
  - Window: 24h or 72h
  - Categories active / empty / failed (with names of failed ones)
  - Total items
  - Subject line used
  - Resend message_id (if delivered)
  - Total wall-clock seconds for the run

  Return successfully. The /schedule trace preserves the full session for the operator to review.
```

- [ ] **Step 2: Verify YAML still parses**

```bash
python3 -c "import yaml; c = yaml.safe_load(open('config.yaml')); assert 'orchestrator_instructions' in c; assert '(TBD' not in c['orchestrator_instructions']; print('ok')"
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Add orchestrator_instructions to config.yaml"
```

---

## Task 5: Write the sub-agent prompt template

**Files:**
- Modify: `config.yaml` (fill the `sub_agent_prompt_template` block)

**Goal:** one rendered prompt per category. The template is substituted with category-specific values by the orchestrator.

- [ ] **Step 1: Replace the `sub_agent_prompt_template` placeholder with this exact block**

In `config.yaml`, replace:
```yaml
sub_agent_prompt_template: |
  (TBD — filled in by Task 5)
```
with:
```yaml
sub_agent_prompt_template: |
  You are a research sub-agent for the Daily AI Brief. Your category is: **{{CATEGORY_NAME}}** (id: `{{CATEGORY_ID}}`).

  Your job: find the most signal-rich news items in your category from the time window below, and return ONE strict JSON object — no prose, no markdown fences, no commentary.

  ## Time window
  Include only items published or announced on or after `{{WINDOW_START_ISO}}`. Earlier items are stale and must be excluded. If a source page shows a date/timestamp, check it.

  ## Primary sources (check all of these)
  {{PRIMARY_SOURCES}}

  ## Secondary search directions
  Use WebSearch for these if the primary sources yield fewer than 3 items:
  {{SECONDARY_SEARCHES}}

  ## Fetch strategy
  {{FETCH_STRATEGY}}

  When using a REST/Atom API, prefer Bash + curl over WebFetch — the structured response is easier to parse reliably. Example GitHub call: `curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" "https://api.github.com/repos/OWNER/REPO/releases?per_page=10"`. If `$GITHUB_TOKEN` is unset, drop the header; you'll fall back to the 60/hr unauth limit (still fine for this volume).

  ## Judgment rules
  - **Signal over volume.** 3-5 great items beat 15 mediocre ones. **Hard cap: {{MAX_ITEMS}} items.**
  - Ignore hype threads, "X is dead" takes, and pure vendor positioning. No "in today's rapidly evolving AI landscape" content.
  - A vendor blog post counts ONLY if it describes new functionality — not positioning, partnerships, awards, or hiring.
  - Prefer papers and production postmortems over vendor blog posts.
  - Verify each URL returns a non-404 before including it.
  - If a primary source 404s or returns nothing new, add the URL string to `errors` and move on. Don't block the whole category on one source.

  ## Sapiens Lab relevance (optional, per item)
  The reader is building Sapiens Lab, a responsible AI tooling company in Spain targeting the EU public sector. Stack: LiteLLM Proxy, Langfuse (self-hosted), DeepEval, MLflow, Kubernetes on AWS. If an item has a **specific, concrete** angle (e.g., "could replace our orchestrator model", "affects our eval pipeline", "changes our EU AI Act compliance story"), include it as `sapiens_angle`. **Do not stretch.** Generic AI-industry commentary → `sapiens_angle: null`.

  ## Output format
  Return ONLY valid JSON matching this schema. No markdown, no commentary, no explanation:

  ```
  {
    "category": "{{CATEGORY_ID}}",
    "nothing_notable": false,
    "items": [
      {
        "headline": "One-line action-oriented headline.",
        "summary": "2-3 sentences stating the concrete takeaway. Not 'X announced Y' — instead 'X's Y reduces latency by N%'.",
        "sapiens_angle": "One line if concretely relevant; otherwise null.",
        "url": "https://..."
      }
    ],
    "errors": ["source URL or search query that failed — short strings"]
  }
  ```

  ### Canonical empty
  If there is genuinely nothing in your category for the window:
  ```
  { "category": "{{CATEGORY_ID}}", "nothing_notable": true, "items": [], "errors": [] }
  ```

  Return that — don't pad with weak items.
```

- [ ] **Step 2: Verify YAML still parses**

```bash
python3 -c "import yaml; c = yaml.safe_load(open('config.yaml')); assert '(TBD' not in c['sub_agent_prompt_template']; assert '{{CATEGORY_ID}}' in c['sub_agent_prompt_template']; print('ok')"
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Add sub_agent_prompt_template to config.yaml"
```

---

## Task 6: Populate categories 1–4 in config.yaml

**Files:**
- Modify: `config.yaml` (fill the `categories: []` list with the first 4 entries)

**Goal:** transcribe the spec's source-map table into YAML for categories 1–4 (model releases, agents, RAG, MCP).

- [ ] **Step 1: Replace `categories: []` with this block**

```yaml
categories:
  - id: model-releases
    name: "Model releases & updates"
    fetch_strategy: |
      All primary sources are HTML blog pages — use WebFetch. For Hugging Face trending models, WebFetch the trending page once and summarize top entries. For "DeepSeek release" / "Qwen release" / "open weights release" secondary queries, use WebSearch with a `last 24 hours` hint.
    primary:
      - { url: "https://anthropic.com/news", type: blog }
      - { url: "https://openai.com/news", type: blog }
      - { url: "https://deepmind.google/discover/blog", type: blog }
      - { url: "https://ai.meta.com/blog", type: blog }
      - { url: "https://mistral.ai/news", type: blog }
      - { url: "https://huggingface.co/blog", type: blog }
      - { url: "https://huggingface.co/models?sort=trending", type: trending-list }
    secondary:
      - "Qwen release last 24h"
      - "DeepSeek release last 24h"
      - "open weights release last 24h"

  - id: agents
    name: "Agents & multi-agent systems"
    fetch_strategy: |
      All primary sources are GitHub repos — use GitHub REST API via Bash curl: `curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" "https://api.github.com/repos/OWNER/REPO/releases?per_page=5"`. Parse the JSON and extract `tag_name`, `name`, `body`, `published_at`, `html_url`. Include only releases where `published_at >= {{WINDOW_START_ISO}}`. For secondary sources (Simon Willison, Latent Space, Smol AI), use WebFetch against their landing pages.
    primary:
      - { repo: "anthropics/claude-agent-sdk", type: gh-releases }
      - { repo: "langchain-ai/langgraph", type: gh-releases }
      - { repo: "crewAIInc/crewAI", type: gh-releases }
      - { repo: "microsoft/autogen", type: gh-releases }
      - { repo: "openai/openai-agents-python", type: gh-releases }
    secondary:
      - "site:simonwillison.net agents last 24h"
      - "latent.space agents this week"
      - "smol.ai agent news"

  - id: rag
    name: "RAG"
    fetch_strategy: |
      arXiv Atom API is the primary fetch — use Bash curl: `curl -sS "https://export.arxiv.org/api/query?search_query=cat:cs.IR+OR+cat:cs.CL&sortBy=submittedDate&sortOrder=descending&max_results=50"`. Parse the Atom XML, filter entries where `<published>` is within the window, and keep items whose title or abstract contains retrieval-adjacent terms (RAG, retrieval-augmented, vector search, reranking, hybrid search, graph RAG, agentic RAG, chunking). For secondary sources, WebFetch eugeneyan.com and hamel.dev landing pages.
    primary:
      - { url: "https://export.arxiv.org/api/query?search_query=cat:cs.IR+OR+cat:cs.CL&sortBy=submittedDate&sortOrder=descending&max_results=50", type: arxiv-atom }
    secondary:
      - "eugeneyan.com RAG this week"
      - "hamel.dev RAG this week"
      - "graph RAG paper last 24h"
      - "agentic RAG paper"

  - id: mcp
    name: "Model Context Protocol (MCP)"
    fetch_strategy: |
      GitHub REST API for spec releases and `servers` repo commits. For commits, use `curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" "https://api.github.com/repos/modelcontextprotocol/servers/commits?since={{WINDOW_START_ISO}}&per_page=30"`. WebFetch modelcontextprotocol.io for blog/changelog. For HN MCP-tag items, compute the epoch timestamp from `{{WINDOW_START_ISO}}` with Python (`python3 -c "import datetime; print(int(datetime.datetime.fromisoformat('{{WINDOW_START_ISO}}'.replace('Z','+00:00')).timestamp()))"`) and use it with the Algolia API: `curl -sS "https://hn.algolia.com/api/v1/search?query=mcp&tags=story&numericFilters=created_at_i>EPOCH,points>20"`. This is cross-platform (no BSD-vs-GNU `date` ambiguity).
    primary:
      - { repo: "modelcontextprotocol/specification", type: gh-releases }
      - { repo: "modelcontextprotocol/servers", type: gh-commits }
      - { url: "https://modelcontextprotocol.io", type: blog }
    secondary:
      - "site:news.ycombinator.com MCP last 24h"
      - "Simon Willison MCP"
```

- [ ] **Step 2: Verify YAML still parses and has 4 categories**

```bash
python3 -c "import yaml; c = yaml.safe_load(open('config.yaml')); assert len(c['categories']) == 4; assert [x['id'] for x in c['categories']] == ['model-releases','agents','rag','mcp']; print('ok')"
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Populate categories 1-4 (models, agents, RAG, MCP) in config.yaml"
```

---

## Task 7: Populate categories 5–8 in config.yaml

**Files:**
- Modify: `config.yaml` (append 4 more category entries)

- [ ] **Step 1: Append these 4 entries to the `categories:` list, after the MCP entry**

```yaml
  - id: claude-skills-code
    name: "Claude Skills & Claude Code"
    fetch_strategy: |
      WebFetch anthropic.com/news and filter for Claude Code / Skills / Cowork mentions. WebFetch docs.claude.com/en/release-notes/claude-code. For community skills, WebSearch "Claude Code skill" and filter for GitHub repos with recent commits. r/ClaudeAI top-of-day via WebFetch against the subreddit (if HTML returns useful content) or WebSearch.
    primary:
      - { url: "https://anthropic.com/news", type: blog }
      - { url: "https://docs.claude.com/en/release-notes/claude-code", type: changelog }
    secondary:
      - "Claude Code skill new release"
      - "Claude Code routine this week"
      - "r/ClaudeAI top post today"

  - id: llm-gateways
    name: "LLM gateways & proxies"
    fetch_strategy: |
      GitHub REST API for the three OSS gateways (LiteLLM, Helicone, Portkey). Kong and Cloudflare changelogs via WebFetch. For HN, Algolia API with query "LLM gateway" OR "LiteLLM".
    primary:
      - { repo: "BerriAI/litellm", type: gh-releases }
      - { repo: "Helicone/helicone", type: gh-releases }
      - { repo: "Portkey-AI/gateway", type: gh-releases }
      - { url: "https://docs.konghq.com/hub/kong-inc/ai-proxy/changelog/", type: changelog }
      - { url: "https://developers.cloudflare.com/ai-gateway/reference/changelog/", type: changelog }
    secondary:
      - "LiteLLM release notes this week"
      - "LLM gateway production"

  - id: evals
    name: "Evaluation frameworks"
    fetch_strategy: |
      GitHub REST API for the OSS eval libraries. WebFetch Braintrust blog and LangSmith changelog. For arXiv, narrow Atom query to eval-related keywords via title/abstract filter after fetch: `curl -sS "https://export.arxiv.org/api/query?search_query=cat:cs.CL+AND+abs:evaluation&sortBy=submittedDate&sortOrder=descending&max_results=30"`.
    primary:
      - { repo: "confident-ai/deepeval", type: gh-releases }
      - { repo: "explodinggradients/ragas", type: gh-releases }
      - { repo: "promptfoo/promptfoo", type: gh-releases }
      - { repo: "UKGovernmentBEIS/inspect_ai", type: gh-releases }
      - { url: "https://www.braintrust.dev/blog", type: blog }
      - { url: "https://changelog.langchain.com", type: changelog }
    secondary:
      - "LLM evaluation paper last 24h"
      - "LLM-as-judge new technique"

  - id: observability
    name: "LLM observability"
    fetch_strategy: |
      GitHub REST API for Langfuse, Arize Phoenix, OpenLLMetry. LangSmith changelog via WebFetch. Helicone blog via WebFetch. For OpenTelemetry GenAI semconv, check github.com/open-telemetry/semantic-conventions commits (gh-commits).
    primary:
      - { repo: "langfuse/langfuse", type: gh-releases }
      - { repo: "Arize-ai/phoenix", type: gh-releases }
      - { repo: "traceloop/openllmetry", type: gh-releases }
      - { repo: "open-telemetry/semantic-conventions", type: gh-commits }
      - { url: "https://changelog.langchain.com", type: changelog }
      - { url: "https://www.helicone.ai/blog", type: blog }
    secondary:
      - "LLM observability OTEL update"
      - "GenAI semconv release"
```

- [ ] **Step 2: Verify YAML still parses and has 8 categories**

```bash
python3 -c "import yaml; c = yaml.safe_load(open('config.yaml')); assert len(c['categories']) == 8; ids = [x['id'] for x in c['categories']]; assert ids[-4:] == ['claude-skills-code','llm-gateways','evals','observability']; print('ok')"
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Populate categories 5-8 (Claude Code, gateways, evals, observability)"
```

---

## Task 8: Populate categories 9–11 in config.yaml

**Files:**
- Modify: `config.yaml` (append the final 3 category entries)

- [ ] **Step 1: Append these 3 entries to the `categories:` list**

```yaml
  - id: self-hosting
    name: "Self-hosting & deployment"
    fetch_strategy: |
      GitHub REST API for vllm, sglang, TGI, ollama. For r/LocalLLaMA top-of-day, WebFetch https://old.reddit.com/r/LocalLLaMA/top/?t=day — old.reddit is much more WebFetch-friendly than new Reddit. Filter posts with score > 200.
    primary:
      - { repo: "vllm-project/vllm", type: gh-releases }
      - { repo: "sgl-project/sglang", type: gh-releases }
      - { repo: "huggingface/text-generation-inference", type: gh-releases }
      - { repo: "ollama/ollama", type: gh-releases }
    secondary:
      - "r/LocalLLaMA top today"
      - "GGUF quantization new technique"
      - "vLLM performance benchmark"

  - id: ai-platform-devex
    name: "AI platform & developer experience"
    fetch_strategy: |
      No single primary hub; this category leans on WebSearch against the named writers + HN. WebFetch latent.space, hamel.dev, and huyenchip.com landing pages for fresh content. Use HN Algolia API with query "internal AI platform" OR "prompt management" filtered to the window.
    primary:
      - { url: "https://latent.space", type: blog }
      - { url: "https://hamel.dev/posts.html", type: blog }
      - { url: "https://huyenchip.com", type: blog }
    secondary:
      - "internal AI platform architecture"
      - "prompt registry production"
      - "feature flags for LLM models"

  - id: eu-ai-act
    name: "EU AI Act & regulation (EU + Spain)"
    fetch_strategy: |
      All primary sources are HTML — use WebFetch on /actualidad or /news index pages, then WebFetch the individual article pages for substance. AESIA and Red.es content is in Spanish — summarize the findings in English for the digest body, but note the Spanish-language origin. No RSS on any of these three; HTML scraping is required. For EU-level news, the Commission's digital-strategy page is usually English + multilingual.
    primary:
      - { url: "https://digital-strategy.ec.europa.eu/en/policies/ai-act", type: blog }
      - { url: "https://aesia.digital.gob.es/es/actualidad", type: blog-es }
      - { url: "https://www.red.es/es/actualidad", type: blog-es }
      - { url: "https://www.bsc.es/news", type: blog }
    secondary:
      - "EU AI Act implementation last week"
      - "ENS certification AI España"
      - "Reglamento IA España nuevo"
      - "AESIA resolución"
```

- [ ] **Step 2: Verify YAML parses and has all 11 categories with unique ids**

```bash
python3 -c "
import yaml
c = yaml.safe_load(open('config.yaml'))
ids = [x['id'] for x in c['categories']]
assert len(ids) == 11, f'expected 11, got {len(ids)}'
assert len(set(ids)) == 11, 'duplicate ids'
assert ids[-3:] == ['self-hosting','ai-platform-devex','eu-ai-act']
print('ok')
"
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add config.yaml
git commit -m "Populate final 3 categories (self-hosting, DevEx, EU AI Act)"
```

---

## Task 9: Write `README.md`

**Files:**
- Create: `README.md`

**Goal:** a public-facing README that documents what this is, how to deploy it, and how to iterate. Doubles as Sapiens-Lab-adjacent portfolio material since the repo is public.

- [ ] **Step 1: Create `README.md` with this exact content**

```markdown
# daily-ai-brief

A recurring Claude Code `/schedule` routine that produces a daily AI intelligence digest and emails it as well-designed HTML via Resend.

Covers 11 categories weighted toward what matters to EU public-sector AI tooling: model releases, agents, RAG, MCP, Claude Skills, LLM gateways, eval frameworks, observability, self-hosting, AI platform/DevEx, and EU AI Act regulation.

## Architecture

- **Thin routine prompt** (`routine-prompt.md`) — pasted once into `/schedule`, never changes.
- **Fat config** (`config.yaml`) — sources, sub-agent prompt template, orchestrator step-by-step, email metadata. Iterate via `git commit` + `git push`; next day's run picks up the change automatically.
- **HTML template** (`templates/email.html`) — Gmail-safe palette, inline CSS, 600px max width, dark-mode friendly.
- **Runtime**: Claude Opus (orchestrator) + 11 parallel Claude Sonnet sub-agents (one per category).
- **Delivery**: Resend API via Bash `curl` from the orchestrator.

See `docs/ultrapowers/specs/2026-04-24-daily-ai-brief-design.md` for the full design rationale and `docs/ultrapowers/research/2026-04-24-research-brief.md` for the state-of-the-art research that shaped v1.

## First-time setup (~25 min)

1. Clone this repo and push it to your own GitHub (must be public — the routine WebFetches `config.yaml` from `raw.githubusercontent.com`).
2. Sign up for [Resend](https://resend.com) (free tier: 100 emails/day, 3000/month). Create an API key. For v1 you can use their shared `onboarding@resend.dev` sender — no domain verification needed.
3. (Recommended) Create a GitHub Personal Access Token with `public_repo` scope. This raises the GitHub REST API rate limit from 60/hr to 5000/hr.
4. In Claude Code, run `/schedule create` and paste the contents of `routine-prompt.md`.
   - Cron: `30 7 * * 1-5`
   - Timezone: `Europe/Madrid`
   - Env vars: `RESEND_API_KEY=re_...` (required), `GITHUB_TOKEN=ghp_...` (optional).
5. Trigger a manual run ("Run now") from the `/schedule` UI and confirm the email arrives at your inbox. Check Resend's dashboard for delivery status.
6. If the first delivery lands in Gmail's Promotions tab: star it or drag it to Primary. Gmail learns per-sender; subsequent runs will land in Primary.

## Iteration

Day-to-day tuning is a normal git workflow:

- **Add or remove a source** → edit `config.yaml` under `categories[].primary` or `.secondary` → commit + push.
- **Tweak a sub-agent's judgment** → edit `config.yaml` under `sub_agent_prompt_template` → commit + push.
- **Adjust email styling** → edit `templates/email.html` → commit + push.
- **Change an orchestrator step** → edit `config.yaml` under `orchestrator_instructions` → commit + push.
- **Change the routine prompt itself** (rare) → edit `routine-prompt.md`, then `/schedule update` and paste the new content.

## What's NOT in v1

- Custom sending domain (uses Resend's shared `onboarding@resend.dev`).
- Persisted "since last run" state — the window is always 24h (72h on Mondays).
- Heartbeat / alerting if a run fails — operator notices missing email by 09:00 Madrid.
- Langfuse tracing (deferred to v2 when this ports to K8s).
- Archive / search of historical digests.

## Cost

Estimated $0.15–$0.40 per run on Max-tier usage. 20 weekday runs/month → ~$3-$8/month in LLM cost. Resend free tier covers delivery.

## License

Personal project by techno1731. Use freely; no warranty.
```

- [ ] **Step 2: Verify the README renders cleanly**

```bash
open README.md
# or
glow README.md  # if you have glow installed
```
Visual check: headings render, code blocks render, no broken links in the body text.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Add README with architecture overview and deployment guide"
```

---

## Task 10: Local dry-run smoke test

**Files:**
- None created/modified; this is a validation-only task.

**Goal:** catch structural / logical bugs in the orchestrator and sub-agent prompts before the routine goes live. This is the one "test" that matters for a prompt-engineering project.

- [ ] **Step 1: Validate all three artifacts parse and cross-reference correctly**

Run this block from the project root:
```bash
python3 <<'PY'
import yaml, re, html.parser, sys

# 1. Load config
with open('config.yaml') as f:
    cfg = yaml.safe_load(f)

# 2. Check top-level keys
for k in ['thresholds','meta','orchestrator_instructions','sub_agent_prompt_template','categories']:
    assert k in cfg, f'missing top-level key: {k}'

# 3. Check thresholds
assert cfg['thresholds']['degraded_run_category_failures'] > 0
assert cfg['thresholds']['max_items_per_category'] > 0

# 4. Check meta has all subject patterns + list-unsubscribe
for k in ['from_address','to_address','reply_to','list_unsubscribe_mailto','subject_pattern','subject_pattern_degraded','subject_pattern_quiet']:
    assert k in cfg['meta'], f'missing meta.{k}'

# 5. Check all 11 categories
assert len(cfg['categories']) == 11, f"expected 11 categories, got {len(cfg['categories'])}"
ids = [c['id'] for c in cfg['categories']]
assert len(set(ids)) == 11, f"duplicate category ids: {ids}"

# 6. Check every category has the fields the template substitutes
for c in cfg['categories']:
    for k in ['id','name','fetch_strategy','primary','secondary']:
        assert k in c, f"category {c.get('id','?')} missing {k}"
    assert len(c['primary']) >= 1, f"category {c['id']} has no primary sources"

# 7. Check sub_agent_prompt_template uses the placeholders the orchestrator will substitute
tmpl = cfg['sub_agent_prompt_template']
for ph in ['{{CATEGORY_ID}}','{{CATEGORY_NAME}}','{{PRIMARY_SOURCES}}','{{SECONDARY_SEARCHES}}','{{FETCH_STRATEGY}}','{{WINDOW_START_ISO}}','{{MAX_ITEMS}}']:
    assert ph in tmpl, f"sub_agent_prompt_template missing placeholder {ph}"

# 8. Check email.html has all placeholders the orchestrator will fill
with open('templates/email.html') as f:
    email_html = f.read()
for ph in ['{{DATE_HUMAN}}','{{TL_DR}}','{{SECTIONS}}','{{DEEP_RESEARCH_BLOCK}}','{{EMERGING_PATTERNS_BLOCK}}','{{RUN_NOTES_BLOCK}}']:
    assert ph in email_html, f"email.html missing placeholder {ph}"

# 9. Check HTML parses without complaint
p = html.parser.HTMLParser()
p.feed(email_html)

print('all structural checks passed')
PY
```
Expected: `all structural checks passed`.

- [ ] **Step 2: Manual dry-run of the orchestrator prompt in an interactive Claude Code session**

This tests the PROMPT (not the delivery). Open a fresh Claude Code session pointed at this repo and send:

```
I want to dry-run the Daily AI Brief orchestrator. Do NOT send an email — stop after rendering the HTML.

1. Read config.yaml and templates/email.html from the working directory (skip the WebFetch step — they're local).
2. Follow the `orchestrator_instructions` block from config.yaml verbatim, with one change: in Step 8, instead of sending via Resend, write the rendered HTML to /tmp/dry-run-digest.html and stop.
3. For sub-agents: dispatch them for real (this validates that Task tool + the sub-agent prompt template actually works). Let them do real WebFetch/WebSearch/curl calls. Accept that this costs a few cents in usage.

Use today as the run date. Window = 24h (pretend it's Tuesday).
```

- [ ] **Step 3: Inspect the dry-run output**

```bash
open /tmp/dry-run-digest.html
```

Manual checks — all must be true before proceeding:
- [ ] Every non-empty category has at least one rendered item with headline + summary + link.
- [ ] The TL;DR paragraph names 3 concrete things from the body content (not generic boilerplate).
- [ ] The `#c14500` accent is visible on links and TL;DR border — not pure black or blue.
- [ ] "Worth deeper research" and "Emerging patterns" sections either have real content or are absent (not present-but-empty).
- [ ] Body text is Georgia serif; headers are sans.
- [ ] Total width is 600px, centered on a `#fafafa` background.
- [ ] At least one item has a non-null `sapiens_angle` rendered in italic rust-color (this validates the per-item conditional rendering).

If any check fails, fix the root cause (usually in `orchestrator_instructions` or the email template) and re-run Step 2. Do NOT move to Task 11 with known bugs in the rendering pipeline.

- [ ] **Step 4: No commit — this task produced no artifact changes.**

If the dry-run surfaced bugs you fixed in config.yaml / templates/email.html / routine-prompt.md, commit those fixes now as `fix: [specific issue found in dry-run]` (one commit per logical fix).

---

## Task 11: Publish to GitHub

**Files:**
- None created; remote setup only.

- [ ] **Step 1: Create the public GitHub repo**

In a browser: create `github.com/techno1731/daily-ai-brief` (public, no README — we already have one, no .gitignore — we already have one, no license is fine for v1). Copy the remote URL.

- [ ] **Step 2: Add remote and push**

```bash
git remote add origin git@github.com:techno1731/daily-ai-brief.git
git push -u origin main
```

- [ ] **Step 3: Verify raw URLs are fetchable**

```bash
curl -sI https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/config.yaml | head -n 1
curl -sI https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/templates/email.html | head -n 1
curl -sI https://raw.githubusercontent.com/techno1731/daily-ai-brief/main/routine-prompt.md | head -n 1
```
Expected: all three return `HTTP/2 200`.

- [ ] **Step 4: No commit step** — `git push` is the action; `.claude/ultrapowers-preferences.json` has auto-push OFF, so this `push` is an explicit operator action, deliberate and manual, one-off for the initial publish.

---

## Task 12: Set up Resend

**Files:**
- None; external platform setup.

- [ ] **Step 1: Create a Resend account**

Go to https://resend.com and sign up. Free tier: 100 emails/day, 3000/month. No domain verification needed for v1 — we'll use their shared `onboarding@resend.dev` sender.

- [ ] **Step 2: Create an API key**

In the Resend dashboard → API Keys → "Create API Key". Scope: "Sending access" is sufficient. Copy the key (starts with `re_`). You won't see it again.

- [ ] **Step 3: Test the API key with a one-shot email**

```bash
export RESEND_API_KEY="re_YOUR_KEY_HERE"
curl -X POST https://api.resend.com/emails \
  -H "Authorization: Bearer $RESEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "AI Brief <onboarding@resend.dev>",
    "to": ["techno1731@gmail.com"],
    "subject": "[AI Brief] Resend setup test",
    "html": "<p>If you see this, Resend is working.</p>",
    "headers": {
      "List-Unsubscribe": "<mailto:techno1731@gmail.com?subject=unsubscribe>",
      "List-Unsubscribe-Post": "List-Unsubscribe=One-Click"
    }
  }'
```
Expected: JSON response with an `id` field (message id). Check techno1731@gmail.com — the email should arrive within ~30 seconds.

- [ ] **Step 4: If the test email lands in Promotions, star it or drag to Primary**

This is the critical Gmail classifier nudge documented in the research brief. It trains the per-sender classifier so the daily digest goes to Primary.

- [ ] **Step 5: Unset the key from the local shell**

```bash
unset RESEND_API_KEY
```
The key belongs in `/schedule`'s env-var config, not your shell history.

---

## Task 13: Create the GitHub PAT (optional but recommended)

**Files:**
- None; external platform setup.

- [ ] **Step 1: Create the PAT**

Go to https://github.com/settings/tokens?type=beta (fine-grained tokens). Create a new token:
- Name: `daily-ai-brief-routine`
- Expiration: 90 days (set a calendar reminder to rotate)
- Repository access: "Public Repositories (read-only)" — this is sufficient for the API calls we make
- Permissions: No specific permissions needed for the public-repo read-only scope

Click "Generate token" and copy the value (starts with `github_pat_`).

- [ ] **Step 2: Test it**

```bash
export GITHUB_TOKEN="github_pat_YOUR_TOKEN"
curl -sS -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/BerriAI/litellm/releases?per_page=1" \
  | python3 -c "import json,sys; r = json.load(sys.stdin); print(r[0]['tag_name'] if isinstance(r,list) else r)"
```
Expected: a tag like `v1.xx.x`. If you get a rate-limit error, the token isn't being applied correctly.

- [ ] **Step 3: Unset from local shell**

```bash
unset GITHUB_TOKEN
```

---

## Task 14: Create the `/schedule` routine

**Files:**
- None; configuration in Claude Code's `/schedule` UI.

- [ ] **Step 1: Open Claude Code and run `/schedule create`**

Paste the entire contents of `routine-prompt.md` as the prompt.

- [ ] **Step 2: Configure the schedule**

- Cron: `30 7 * * 1-5`
- Timezone: `Europe/Madrid`
- Description: `Daily AI Brief — weekdays 07:30 Madrid`

- [ ] **Step 3: Add environment variables**

- `RESEND_API_KEY` = `re_...` (from Task 12)
- `GITHUB_TOKEN` = `github_pat_...` (from Task 13, if created)

- [ ] **Step 4: Save and verify the routine appears in the list**

Check that `/schedule` or the web UI at `claude.ai/code/routines` shows the new routine. Confirm the "Connectors" list shows (at minimum) no connectors required by our routine — we don't use MCP servers in v1.

---

## Task 15: First live run & validate

**Files:**
- None; runtime validation.

- [ ] **Step 1: Trigger a manual run**

From the `/schedule` UI, click "Run now" on the Daily AI Brief routine.

- [ ] **Step 2: Watch the run**

Open the session URL from the `/schedule` detail page. Watch the orchestrator:
- WebFetches config.yaml and templates/email.html (Step 1 + 2)
- Computes the window
- Dispatches sub-agents (confirm whether parallel OR sequential — note the behavior for the record)
- Collects results
- Renders HTML
- POSTs to Resend, logs the message_id

- [ ] **Step 3: Validate the email arrived**

Check `techno1731@gmail.com`. The email should arrive within ~5 minutes of the run starting.

- [ ] **Step 4: End-to-end QA checklist**

Confirm against the same visual rules from Task 10 Step 3, applied to the real email in Gmail:
- [ ] Subject line follows the expected pattern (date + item count + categories)
- [ ] Gmail placement (Primary / Promotions) — if Promotions, star+move to Primary now
- [ ] TL;DR callout visible with rust border
- [ ] All non-empty categories rendered with items
- [ ] At least one item shows a Sapiens-Lab-relevant angle (if the day has such content)
- [ ] Link URLs are live and load correctly
- [ ] Resend dashboard shows `delivered` status for the message id logged by the orchestrator
- [ ] Mobile rendering (check in Gmail iOS app if available) — no horizontal scroll, colors not inverted in dark mode

- [ ] **Step 5: Record observations in `docs/ultrapowers/runs/2026-04-24-first-run.md`**

Create `docs/ultrapowers/runs/` (new directory) and write a short post-run note:

```markdown
# First run — YYYY-MM-DD

**Parallel vs sequential:** observed <parallel|sequential>
**Total runtime:** ~N minutes
**Categories active / empty / failed:** X / Y / Z
**Subject delivered:** [...]
**Gmail placement:** Primary | Promotions (moved)
**Issues found:** [none | list]
**Follow-ups:** [none | list]
```

- [ ] **Step 6: Commit the run note**

```bash
git add docs/ultrapowers/runs/
git commit -m "Record first live run observations"
```

- [ ] **Step 7: If issues were found, file them as GitHub issues and either fix in config.yaml (hot-fix via git push) or accept into a v1.1 backlog.**

The routine is now autonomous. It will fire tomorrow at 07:30 Madrid and every weekday after.

---

## Task 16 (optional): Push the initial commits to GitHub

**Files:**
- None; git push only.

Since auto-push is OFF, the engineer manually pushes when they're ready. This is separate from Task 11's initial publish — it ensures the fix commits from Task 10 (if any) and the run-note commit from Task 15 land on the remote.

- [ ] **Step 1: Review unpushed commits**

```bash
git log origin/main..HEAD --oneline
```

- [ ] **Step 2: Push**

```bash
git push
```

Done. The remote `raw.githubusercontent.com` now serves the latest config.yaml, and the next routine run (tomorrow 07:30 Madrid) will pick up any config fixes made during Task 10.

---

## Post-deployment: what to watch in the first week

1. **Signal quality** — does the TL;DR name actually important things, or is it noisy? If noisy, tune `sub_agent_prompt_template` (stricter noise rules, higher bar on "new functionality").
2. **Gmail placement** — if the digest ever drifts to Promotions after being trained to Primary, that's a sign the content pattern looks promotional. Review the rendered HTML for SALE-like language or image-heavy sections.
3. **Sub-agent failures** — if the same category fails repeatedly, the primary source probably moved or went behind auth. Update `categories[].primary` in config.yaml.
4. **Cost** — check Anthropic console after a week. If per-run cost is materially > $0.40, the sub-agents are over-fetching; tighten fetch_strategy instructions.
5. **Runtime** — if runs exceed 10 minutes, sub-agents are spinning. Add explicit "don't follow more than 2 levels of link depth" to the sub-agent template.

## v1.5 / v2 — already in the spec, don't do yet

- Custom sending domain (v1.5)
- Langfuse tracing
- Persisted since-last-run state
- Heartbeat monitoring
- Local dry-run helper script
- Historical dedup
- Archive static site
- DeepEval regression tests
