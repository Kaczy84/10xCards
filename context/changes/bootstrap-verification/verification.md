---
bootstrapped_at: 2026-05-27T00:00:00Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: 10x-cards
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: 10x-cards
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
```

**Why this stack**: 10xCards is a solo web-app with a 5-week after-hours timeline, email/password auth, and an AI-generation flow as its core value proposition. The 10x-astro-starter is the recommended default for (web-app, js) and clears all four agent-friendly gates: TypeScript end-to-end, Astro file-based routing conventions, strong training-data presence, and current docs. Supabase covers auth and PostgreSQL storage out of the box, eliminating the auth wiring work that typically consumes early MVP time. Cloudflare Pages is the starter's default deployment target and keeps hosting costs near zero at small scale. LLM integration (has_ai: true) will be added manually as a server-side Astro route calling an external provider API; no starter in the registry bundles this first-class, and the user confirmed this manual step. CI runs on GitHub Actions with auto-deploy-on-merge — the standard flow for a solo project.

## Pre-scaffold verification

| Signal      | Value                                          | Severity | Notes                                        |
| ----------- | ---------------------------------------------- | -------- | -------------------------------------------- |
| npm package | not run                                        | n/a      | cmd_template starts with `git clone`; npm check skipped |
| GitHub repo | przeprogramowani/10x-astro-starter pushed 2026-05-17 | fresh    | checked via GitHub API; 10 days ago          |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`
**Strategy**: git-clone
**Exit code**: 0
**Files moved**: 20
**Conflicts (.scaffold siblings)**: CLAUDE.md → CLAUDE.md.scaffold
**.gitignore handling**: moved silently (no .gitignore in cwd before scaffold)
**.bootstrap-scaffold cleanup**: deleted

## Post-scaffold audit

**Tool**: `npm audit --json`
**Summary**: 0 CRITICAL, 1 HIGH, 9 MODERATE, 0 LOW
**Direct vs transitive**: 1/0 HIGH direct/transitive (devalue is a direct finding)

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** — severity: high; via: devalue (direct dependency); fix available: `npm audit fix`. Advisory: prototype pollution or similar deserialization concern in the `devalue` package. Run `npm audit` for full advisory text.

#### MODERATE findings

9 moderate findings present. Run `npm audit` from the project root for the full list. All are transitive. No fix required before first commit unless your risk policy demands it.

#### LOW / INFO findings

None.

## Hints recorded but not acted on

| Hint                    | Value             |
| ----------------------- | ----------------- |
| bootstrapper_confidence | first-class       |
| quality_override        | false             |
| path_taken              | standard          |
| self_check_answers      | null              |
| team_size               | solo              |
| deployment_target       | cloudflare-pages  |
| ci_provider             | github-actions    |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | true              |
| has_payments            | false             |
| has_realtime            | false             |
| has_ai                  | true              |
| has_background_jobs     | false             |

CI/CD scaffolding (GitHub Actions workflow files) and AGENTS.md / CLAUDE.md generation are deferred to a future skill (M1L4). The `deployment_target: cloudflare-pages` hint is preserved here for that skill to act on.

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review `CLAUDE.md.scaffold` — it's the starter's version; your existing `CLAUDE.md` was preserved. Merge any useful content manually.
- Run `npm audit fix` to address the 1 HIGH finding in `devalue` (fix available, non-breaking).
- Add your LLM SDK (`npm install @anthropic-ai/sdk` or equivalent) and wire up the AI generation route in `src/`.
- Configure Supabase: copy `.env.example` → `.env.local` and fill in your Supabase project URL and anon key.
