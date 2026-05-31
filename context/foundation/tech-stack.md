---
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
---

## Why this stack

10xCards is a solo web-app with a 5-week after-hours timeline, email/password auth, and an AI-generation flow as its core value proposition. The 10x-astro-starter is the recommended default for (web-app, js) and clears all four agent-friendly gates: TypeScript end-to-end, Astro file-based routing conventions, strong training-data presence, and current docs. Supabase covers auth and PostgreSQL storage out of the box, eliminating the auth wiring work that typically consumes early MVP time. Cloudflare Pages is the starter's default deployment target and keeps hosting costs near zero at small scale. LLM integration (has_ai: true) will be added manually as a server-side Astro route calling an external provider API; no starter in the registry bundles this first-class, and the user confirmed this manual step. CI runs on GitHub Actions with auto-deploy-on-merge — the standard flow for a solo project.
