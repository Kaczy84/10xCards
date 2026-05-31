# Repository Guidelines

10xCards is an AI-powered flashcard web-app on Astro 6 (SSR, Cloudflare adapter) with React 19 islands, TypeScript strict, Tailwind CSS 4, and Supabase for auth and storage, deploying to Cloudflare Workers.

## Hard Rules

- Never read `SUPABASE_URL` or `SUPABASE_KEY` in client-side code. Both are declared `context: "server"` secrets in `@astro.config.mjs`; any client-side access is a security failure caught by the Astro env schema.
- Never use `set:html` in `.astro` files — ESLint (`astro/no-set-html-directive`) treats it as an error. Use slot composition instead.
- Prefix intentionally unused variables with `_` to satisfy `@typescript-eslint/no-unused-vars` (args, vars, caught errors, and destructured arrays all follow the same rule).
- Extend route protection by adding paths to `PROTECTED_ROUTES` in `src/middleware.ts` only — nowhere else.
- All date/time values are UTC. Use the `formatDate()` helper from `@src/lib/utils.ts` for any date formatting — never call `new Date().toISOString()` directly.

## Project Structure

Source lives under `src/`: pages and API endpoints in `pages/` (API routes follow `pages/api/<resource>/<verb>.ts`), Astro and React components in `components/` (`auth/` and `ui/` subdirs), layout wrappers in `layouts/`, shared utilities in `lib/` (`supabase.ts`, `utils.ts`, `config-status.ts`), route guard in `middleware.ts`, and global styles in `styles/global.css`. Use `@/` (alias for `src/`) for all internal imports.

## Build & Dev Commands

- `npm run dev` — Astro dev server (Cloudflare workerd runtime)
- `npm run build` — production build; requires `SUPABASE_URL` and `SUPABASE_KEY` in env
- `npm run lint` — ESLint with type-checked rules; run before pushing
- `npm run lint:fix` — auto-fix lint issues
- `npm run format` — Prettier

No test framework is configured yet. CI: `@.github/workflows/ci.yml` — runs lint + build on every push and PR to `master`; requires `SUPABASE_URL` and `SUPABASE_KEY` as GitHub repository secrets.

## Coding Style & Naming

TypeScript strict mode via `@tsconfig.json`. Astro files: `PascalCase.astro`; React components: `PascalCase.tsx`; lib utilities: `camelCase.ts`. No `console` statements (ESLint warns). React Compiler plugin is active (`react-compiler/react-compiler: error`) — do not write manual `useMemo`/`useCallback` where the compiler handles memoization. JSX a11y rules are enforced on all components.

## Secrets & Local Dev

Copy `.env.example` to `.dev.vars` for local Cloudflare dev secrets. Never commit either file. See `@README.md` for Supabase local stack setup (Docker-based, Studio at `http://localhost:54323`).

## Commit & PR Guidelines

No commit convention established yet (project has no prior history). husky + lint-staged run ESLint on `*.{ts,tsx,astro}` and Prettier on `*.{json,css,md}` at every commit — hooks cannot be skipped.
