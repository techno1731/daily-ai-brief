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
