---
project: 10xCards
researched_at: 2026-05-29
recommended_platform: Cloudflare Workers
runner_up: Railway
context_type: mvp
tech_stack:
  language: TypeScript / JavaScript
  framework: Astro 6
  runtime: Cloudflare Workers (workerd)
  database: Supabase (external)
  ai_provider: OpenRouter (external)
---

## Recommendation

**Deploy on Cloudflare Workers.**

The project is already scaffolded for Cloudflare — `wrangler.jsonc`, `@astrojs/cloudflare` adapter, and `astro:env` secrets schema are all wired. The workload (stateless SSR, external Supabase + OpenRouter, solo dev, low QPS) stays entirely within the Workers Free tier at 10k–100k req/month. The developer has Cloudflare familiarity (interview Q3), wrangler covers the full operational loop from CLI, and the official MCP server is GA. No other platform scores 5/5 on agent-friendly criteria while also being $0, pre-configured, and familiar.

## Platform Comparison

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration | **Total** |
|---|---|---|---|---|---|---|
| **Cloudflare Workers** | Pass | Pass | Pass | Pass | Pass | **5/5** |
| **Railway** | Pass | Pass | Pass | Pass | Pass | **5/5** |
| **Vercel** | Pass | Pass | Partial | Pass | Pass | **4.5/5** |
| **Fly.io** | Pass | Pass | Pass | Pass | Partial | **4.5/5** |
| **Render** | Partial | Pass | Pass | Partial | Pass | **4/5** |
| **Netlify** | Fail | Pass | Pass | Partial | Partial | **2.5/5** |

**Scoring notes:**

- **Cloudflare** — wrangler v3 covers deploy, tail logs, rollback (`wrangler rollbacks create`), and secret management. Docs source on GitHub as MDX. Official MCP server (`@cloudflare/mcp-server-cloudflare`) GA. Scores Pass across all five criteria.
- **Railway** — railway CLI covers the full loop; `railway rollback` available (limited by image retention). llms.txt + agents.md + GitHub docs = best-in-class agent readability. GA MCP with local + remote modes and a Claude Code marketplace plugin. No free tier (trial only); realistic cost $5–10/month.
- **Vercel** — `@astrojs/vercel` adapter GA; vercel CLI covers deploy, rollback, logs. Remote MCP at mcp.vercel.com (GA, OAuth). Docs are markdown-renderable. Dropped to Partial on agent docs (no confirmed raw GitHub source). Hobby tier risk: 12-function limit per deployment (each SSR route = one function) and 60s max duration — an AI-heavy app may need Pro ($20/month) from day one.
- **Fly.io** — `fly launch` auto-detects Astro + generates Dockerfile and `fly.toml`. `flyctl` covers everything. llms.txt on docs site. No free tier (discontinued 2024); ~$2–5/month with auto-stop suspend. MCP (`fly mcp server`) is **experimental** — scored Partial. Best fit for persistent-process workloads.
- **Render** — $7/month Starter; persistent Node.js process; GA infrastructure MCP at mcp.render.com with 20+ tools and agent skills. CLI rollback not available (requires API call) → Partial on CLI-first and Stable deploy API. Free tier exists but has 15-min spin-down making SSR unusable.
- **Netlify** — CLI deploy GA; `netlify logs --follow` added May 2026. However, **rollback is dashboard-only** (no CLI command) → Fail on CLI-first. Each production deploy costs 15 credits on a 300-credit/month free plan. MCP exists (`@netlify/mcp`, GA Feb 2025) but rollback gap remains a hard blocker for agent-driven ops.

### Shortlisted Platforms

#### 1. Cloudflare Workers (Recommended)

The project was explicitly scaffolded for Cloudflare — `wrangler.jsonc` already exists, `@astrojs/cloudflare` is installed, and `astro:env` server schemas are declared in `astro.config.mjs`. The stack is a native fit: Astro 6 SSR on workerd, stateless request/response (no persistent connection requirement), and both external providers (Supabase, OpenRouter) are reachable from any Workers region. The Free tier covers 100k requests/day ($0), wrangler is the most capable single-tool CLI of all six platforms, and the official MCP server enables structured live-infra queries without parsing CLI output. Developer familiarity breaks the tie with Railway.

#### 2. Railway

The only platform besides Cloudflare to score 5/5 across all criteria. Container-based Node.js deployment means no workerd edge-runtime surprises — any npm package works as expected. Agent tooling is best-in-class: GA MCP (Aug 2025) with a dedicated Claude Code marketplace plugin at railway.com/agents/claude, llms.txt + agents.md, GitHub-hosted docs. Realistic cost for a solo low-QPS app on Hobby: $5–10/month. Primary reason it's runner-up: no pre-existing stack alignment (the project is scaffolded for Cloudflare, not Node/Railway), and the $5–10/month cost vs $0 on Cloudflare's Free tier.

#### 3. Vercel

Mature `@astrojs/vercel` SSR adapter, best-in-class remote MCP (OAuth-backed, full project/deploy/log tooling via mcp.vercel.com). Strong fallback if workerd runtime compatibility becomes a persistent problem. Main risk: the Hobby tier's **12-function-per-deployment limit** means an Astro SSR app with more than 12 routes will hit a hard ceiling without upgrading to Pro ($20/month). The 60-second max function duration (vs 300s on Pro) also creates a timeout risk for OpenRouter calls under load.

## Anti-Bias Cross-Check: Cloudflare Workers

### Devil's Advocate — Weaknesses

1. **workerd ≠ Node.js at runtime** — packages that use Node built-ins (`fs`, `path`, `child_process`, `stream`) require the `nodejs_compat` flag and may still fail silently on production workerd even when the flag is set. Every npm dependency update is a potential edge-runtime surprise that local `wrangler dev` may not reproduce.
2. **CPU time limit is 10ms on Free tier** — I/O awaits (Supabase, OpenRouter) don't count, but Astro's SSR renderer, request body parsing, and template logic do. A complex page or AI post-processing loop can exceed 10ms CPU regularly, forcing an upgrade to Workers Paid ($5/month).
3. **30-second wall-clock timeout (Paid plan)** — if OpenRouter is under load and an AI completion takes > 30s, the Worker is killed with no error recovery path. Streaming via `ReadableStream` is a mitigation, but requires explicit implementation in Astro routes.
4. **Secret management is a two-path friction point** — `SUPABASE_URL` and `SUPABASE_KEY` must be added via `wrangler secret put` for production AND maintained in `.dev.vars` locally. They are NOT read from `.env` at Workers runtime. Easy to forget in a new environment or after a team change.
5. **Workers vs Pages are separate products** — `wrangler.jsonc` indicates a Workers deploy (`wrangler deploy`), but `tech-stack.md` says `deployment_target: cloudflare-pages`. These are different products with different rollback mechanics and pricing. The commands are not interchangeable; this must be clarified before the first production deploy.

### Pre-mortem — How This Could Fail

The first production deploy was smooth — `wrangler.jsonc` was already configured and `wrangler deploy` succeeded in two minutes. Three weeks later, a dependency upgrade pulled in a package that used Node's `EventEmitter` internally. The `nodejs_compat` flag was set, but this specific API surface wasn't fully polished in the workerd emulation. The bug manifested only in production; local `wrangler dev` passed. Three days of debugging, eventually traced to a `wrangler dev` / production runtime divergence. The second crisis came when the AI generation endpoint started hitting the 10ms CPU limit as the LLM response post-processing became more sophisticated. Upgrading to Workers Paid fixed the CPU budget, but then OpenRouter responded slowly one afternoon — 35 seconds — and the Worker was killed mid-stream with no error recovery, leaving users with a blank response. Implementing streaming required refactoring the AI endpoint to use `ReadableStream` with `ctx.waitUntil()` for cleanup. The third issue: the team discovered that Cloudflare AI Gateway (which could cache and rate-limit OpenRouter calls) was significantly easier to set up with Workers than with Pages, and realized the Workers vs Pages distinction should have been decided before day one, not after.

### Unknown Unknowns

1. **Edge routing can make Supabase latency worse, not better** — a Worker runs at the edge nearest to the user (e.g., Warsaw), but Supabase's database may be in `eu-west-1` or `us-east-1`. Every DB query from a Warsaw Worker to a Frankfurt Supabase adds a network hop. At single-region scale, a Node.js server in the same region as Supabase would have lower tail latency.
2. **`waitUntil` is required for fire-and-forget work** — logging AI generation events, updating `last_seen` timestamps, or sending analytics after the response is sent cannot use `setTimeout`. They require `ctx.waitUntil()` accessed via `Astro.locals.runtime.ctx`, a non-obvious pattern specific to the Astro + Cloudflare integration.
3. **Workers and Pages have different feature surfaces** — Cloudflare AI Gateway is simpler to configure with Workers; KV/D1/R2 bindings in `wrangler.jsonc` behave differently between Workers and Pages Functions. Choosing one product and staying consistent is important; switching later is not trivial.
4. **Bundle size limit is 10MB compressed** — large embedded dependencies (e.g., prompt templates, tiktoken) can push the Worker bundle over the limit. There is no warning during `wrangler dev`; the limit surfaces only at `wrangler deploy`.
5. **Rollback reverts code only, not Supabase migrations** — `wrangler rollbacks create` reverts the Worker script to a prior deployment, but Supabase schema migrations do not roll back automatically. A deploy that included a DB migration followed by a code rollback leaves the schema in the new state with the old code running against it.

## Operational Story

- **Preview deploys**: With Cloudflare Pages, each Git branch automatically gets a preview URL at `<branch>.<project>.pages.dev`. With Workers (as currently configured via `wrangler.jsonc`), preview environments require separate `wrangler.toml` environment blocks and explicit `wrangler deploy --env staging` commands — no automatic branch previews. Protect preview URLs with Cloudflare Access (zero-config via the dashboard) if the app contains sensitive data.
- **Secrets**: Production secrets (`SUPABASE_URL`, `SUPABASE_KEY`, `OPENROUTER_API_KEY`) live in Cloudflare Workers Secrets, set via `wrangler secret put <KEY>`. Local dev secrets go in `.dev.vars` (gitignored). Rotation: `wrangler secret put <KEY>` with the new value — the old value is replaced atomically. Never commit secrets to `wrangler.toml` or `.env`.
- **Rollback**: `wrangler deployments list` to see versions; `wrangler rollbacks create` to revert to the prior deployment. Typical time-to-revert: < 30 seconds globally. Caveat: does not roll back Supabase migrations — database schema changes must be reverted manually via a new migration.
- **Approval**: Agent may run `wrangler deploy`, `wrangler secret put`, `wrangler tail`, `wrangler deployments list`, and `wrangler rollbacks create` unattended. Human-only actions: dropping Supabase tables, rotating the primary Supabase service key, changing Cloudflare account-level settings (billing, DNS, domain routing), and deleting the Workers project.
- **Logs**: `wrangler tail` streams structured JSON logs in real time (stdout, stderr, uncaught exceptions, request metadata). For Pages: `wrangler pages deployment tail`. Filter by invocation status, sampling rate, or search string: `wrangler tail --status error --sampling-rate 1`.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| npm dependency uses Node built-in not covered by `nodejs_compat` | Devil's advocate | M | H | Audit `node_modules` with `wrangler deploy --dry-run`; pin versions; test on `wrangler dev` (workerd) not `astro dev` (Node) |
| AI endpoint exceeds 30s wall-clock timeout (OpenRouter slow) | Pre-mortem | M | H | Implement streaming SSE response for AI generation endpoint; set OpenRouter `max_tokens` limit to bound completion time |
| CPU time exceeds 10ms Free tier limit | Devil's advocate | M | M | Profile CPU-heavy routes with `wrangler dev --local-protocol https`; upgrade to Workers Paid ($5/month) early if needed |
| Workers vs Pages product confusion at deploy | Unknown unknowns | H | M | Decide before first deploy: use Workers (`wrangler deploy`) or Pages (`wrangler pages deploy`). Current config (`wrangler.jsonc`) = Workers. Document in CLAUDE.md |
| Secret missing from Workers Secrets after environment change | Devil's advocate | M | H | Maintain a `.dev.vars.example` that mirrors all required keys; checklist in CLAUDE.md for new environment setup |
| Supabase latency elevated due to edge routing | Unknown unknowns | M | L | Deploy Supabase in `eu-central-1` (Frankfurt); use Cloudflare Hyperdrive to pool Supabase connections if latency becomes noticeable |
| Rollback doesn't revert DB migration | Unknown unknowns | L | H | Always write reversible migrations (add columns before removing old ones); test rollback path in staging before prod |
| Bundle size exceeds 10MB | Devil's advocate | L | M | Run `wrangler deploy --dry-run` to check bundle size before merge; avoid embedding large static assets in JS modules |
| `waitUntil` pattern missed for async tasks | Unknown unknowns | M | L | Add `ctx.waitUntil()` usage to CLAUDE.md as a required pattern for any fire-and-forget work in Astro+CF routes |

## Getting Started

1. **Confirm Workers vs Pages**: The project has `wrangler.jsonc` → this is a Workers deploy. First deploy command: `npx wrangler deploy`. If you prefer Pages (automatic branch previews), migrate to Pages by removing `wrangler.jsonc` and using `wrangler pages deploy ./dist` after `npm run build`. Do not mix the two.

2. **Set production secrets**: For each secret in `.dev.vars`, run:
   ```
   npx wrangler secret put SUPABASE_URL
   npx wrangler secret put SUPABASE_KEY
   npx wrangler secret put OPENROUTER_API_KEY
   ```
   These are stored in Cloudflare's secret store, not in any file.

3. **Verify `nodejs_compat` flag**: Check `wrangler.jsonc` contains `"compatibility_flags": ["nodejs_compat"]`. This is required for Supabase's SSR client which uses Node crypto internally.

4. **Deploy and verify**: 
   ```
   npm run build
   npx wrangler deploy
   npx wrangler tail
   ```
   Open the deployed URL, sign in, and watch `wrangler tail` for any runtime errors.

5. **Optional: add Cloudflare MCP to Claude Code** for structured live-infra queries:
   ```
   npx @cloudflare/mcp-server-cloudflare
   ```
   Then add to `.mcp.json` in the project root with your Cloudflare API token scoped to this Workers project only (no DNS, no billing).

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup (GitHub Actions deploy step)
- Production-scale architecture (multi-region, HA, DR)
- Cloudflare Pages vs Workers migration guide
