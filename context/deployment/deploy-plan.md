# Deploy Plan: Pierwsze wdrożenie 10xCards na Cloudflare Workers

**Data**: 2026-05-31  
**Platforma**: Cloudflare Workers  
**Podstawa**: `context/foundation/infrastructure.md`, `context/foundation/tech-stack.md`

---

## Zewnętrzne integracje

| Integracja | Rola | Sekrety | Konfiguracja |
|---|---|---|---|
| **Cloudflare Workers** | Runtime + hosting | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` | `wrangler.jsonc` |
| **Supabase** | Auth (email/password) + baza danych | `SUPABASE_URL`, `SUPABASE_KEY` | Dashboard → Auth → URL Configuration |
| **GitHub Actions** | CI/CD auto-deploy | wszystkie powyższe jako repo secrets | `.github/workflows/ci.yml` |
| **OpenRouter** | AI (nie zaimplementowany) | `OPENROUTER_API_KEY` (dodać przy implementacji AI) | — |

---

## Znany bloker do naprawy przed deploy

**`SUPABASE_URL` w `.dev.vars` ma błędny suffix `/rest/v1/`.**  
`@supabase/ssr` buduje URL-e auth wewnętrznie (`${SUPABASE_URL}/auth/v1/...`).  
Z sufiksem → `https://xxx.supabase.co/rest/v1//auth/v1/user` → błąd 404 na każde wywołanie auth.

```diff
- SUPABASE_URL=https://fjjmtkirumucazxxvghr.supabase.co/rest/v1/
+ SUPABASE_URL=https://fjjmtkirumucazxxvghr.supabase.co
```

Poprawka dotyczy zarówno sekretu w Workers (`wrangler secret put`) jak i `.dev.vars` dla lokalnego dev.

---

## Fazy i checkpointy

Każdy checkpoint ma status: `[ ]` oczekujący / `[x]` ukończony / `[!]` problem.  
Agent wykonuje kroki oznaczone **(A)**, użytkownik kroki **(U)**.

---

### FAZA 0 — Weryfikacja lokalna `(A)`

Cel: potwierdzić że build działa lokalnie i bundle mieści się w limicie.

```bash
npm run build
npx wrangler deploy --dry-run
```

- [ ] **CP-0.1** `npm run build` kończy się bez błędów
- [ ] **CP-0.2** `wrangler deploy --dry-run` — rozmiar Worker bundle < 10 MB (skompresowany)
- [ ] **CP-0.3** Brak błędów TypeScript / `astro:env` w outputcie build

> Jeśli CP-0.2 fail: sprawdź największe zależności przez `npx wrangler deploy --dry-run --json | jq '.files'`.

---

### FAZA 1 — Cloudflare: logowanie i konto `(U)`

Cel: uwierzytelnić wrangler lokalnie i poznać Account ID (potrzebny do URL Workers i do GitHub Secret).

```
! npx wrangler login
! npx wrangler whoami
```

- [ ] **CP-1.1** `wrangler login` — przeglądarka otworzyła OAuth, token zapisany lokalnie
- [ ] **CP-1.2** `wrangler whoami` — wyświetla nazwę konta i **Account ID** (zapisz go — potrzebny w FAZA 4 i do skonstruowania URL produkcyjnego)

> URL produkcyjny po deploy: `https://10xcards.<account-subdomain>.workers.dev`  
> Account subdomain można sprawdzić: Cloudflare dashboard → Workers & Pages → dowolny worker → URL.

---

### FAZA 2 — Cloudflare Workers Secrets `(U)`

Cel: wgrać sekrety produkcyjne do Workers Secrets (runtime — nie plik, nie repo).

**Ważne**: podaj `SUPABASE_URL` BEZ `/rest/v1/` — tylko base URL.

```
! npx wrangler secret put SUPABASE_URL
  → wpisz: https://fjjmtkirumucazxxvghr.supabase.co

! npx wrangler secret put SUPABASE_KEY
  → wpisz: sb_publishable_gvXKHpaboyWljIOLUaeY9w_GrwZfb3v
```

Weryfikacja:

```
! npx wrangler secret list
```

- [ ] **CP-2.1** `SUPABASE_URL` widoczny w `wrangler secret list` (wartość ukryta — tylko nazwa)
- [ ] **CP-2.2** `SUPABASE_KEY` widoczny w `wrangler secret list`
- [ ] **CP-2.3** `SUPABASE_URL` NIE zawiera `/rest/v1/` (potwierdź wpisując wartość ponownie jeśli niepewny)

---

### FAZA 3 — Pierwszy deploy `(A)`

Cel: wypuścić Workers na produkcję i potwierdzić że aplikacja odpowiada.

```bash
npx wrangler deploy
```

- [ ] **CP-3.1** `wrangler deploy` kończy się bez błędów i zwraca URL (`https://10xcards.*.workers.dev`)
- [ ] **CP-3.2** Strona główna (`/`) ładuje się — HTTP 200, brak błędów JS w konsoli
- [ ] **CP-3.3** `npx wrangler tail` uruchomiony — obserwuj przez 60s, brak błędów `dynamic require`, `CPU time limit`, `TypeError`

> Jeśli CP-3.3 pokazuje `dynamic require of "stream" is not supported`:  
> Wersja `@supabase/ssr@0.10.x` powinna to obsługiwać przez `nodejs_compat` — sprawdź czy flaga jest w `wrangler.jsonc` (`"compatibility_flags": ["nodejs_compat"]`). Jeśli problem nadal występuje: `npm update @supabase/ssr` do najnowszej wersji i ponów deploy.

---

### FAZA 4 — Supabase: konfiguracja produkcyjna `(U)`

Cel: skonfigurować Supabase pod produkcyjną domenę, żeby email confirmation działał.

**Wymagany URL z FAZY 3** (np. `https://10xcards.xyz123.workers.dev`).

#### 4a. URL Configuration (Supabase Dashboard → Authentication → URL Configuration)

- [ ] **CP-4.1** **Site URL** ustawiony na URL z FAZY 3 (np. `https://10xcards.xyz123.workers.dev`)
- [ ] **CP-4.2** **Redirect URLs** — dodano `https://10xcards.xyz123.workers.dev/**`  
  *(wildcard `/**` pokrywa `/auth/confirm-email` i inne przyszłe callbacki)*

#### 4b. Email settings (Supabase Dashboard → Authentication → Providers → Email)

- [ ] **CP-4.3** **Confirm email** ustawiony na **ON** (domyślnie wyłączony na nowych projektach)
- [ ] **CP-4.4** (opcjonalne MVP) SMTP — Supabase free tier wysyła ~3 email/h wbudowanym mailem; jeśli potrzebujesz więcej: skonfiguruj zewnętrzny SMTP (SendGrid, Resend, Mailgun) w Authentication → SMTP Settings

#### 4c. Weryfikacja konfiguracji

- [ ] **CP-4.5** Email template dla `Confirm signup` zawiera link do właściwej domeny (Authentication → Email Templates → Confirm signup)

---

### FAZA 5 — Test pełnego auth flow `(U + A)`

Cel: potwierdzić end-to-end że auth działa na produkcji.

Otwórz `https://10xcards.*.workers.dev` w przeglądarce:

- [ ] **CP-5.1** `/auth/signup` — formularz rejestracji wyświetla się poprawnie
- [ ] **CP-5.2** Rejestracja testowego konta → przekierowanie do `/auth/confirm-email` → strona potwierdza "sprawdź email"
- [ ] **CP-5.3** Email potwierdzający przyszedł (sprawdź spam jeśli nie ma w ciągu 2 min)
- [ ] **CP-5.4** Kliknięcie linku w emailu → redirect do `https://10xcards.*.workers.dev/auth/confirm-email` → brak błędów
- [ ] **CP-5.5** `/auth/signin` — logowanie testowym kontem → redirect do `/dashboard` → działa
- [ ] **CP-5.6** `/auth/signout` → redirect do `/` → sesja wyczyszczona (powrót do signin po wejściu na `/dashboard`)
- [ ] **CP-5.7** `wrangler tail` podczas testu — brak błędów runtime w logach

> Jeśli CP-5.3 fail (email nie przychodzi): sprawdź CP-4.3 (Confirm email ON) i CP-4.4 (SMTP lub Supabase built-in).  
> Jeśli CP-5.4 fail (redirect loop): sprawdź CP-4.1 i CP-4.2 (Site URL + Redirect URLs w Supabase).

---

### FAZA 6 — GitHub Secrets + CI auto-deploy `(U + A)`

Cel: skonfigurować auto-deploy na push do `master`.

#### 6a. GitHub Secrets `(U)` — repo → Settings → Secrets and variables → Actions

| Secret | Wartość |
|---|---|
| `SUPABASE_URL` | `https://fjjmtkirumucazxxvghr.supabase.co` (bez `/rest/v1/`) |
| `SUPABASE_KEY` | `sb_publishable_gvXKHpaboyWljIOLUaeY9w_GrwZfb3v` |
| `CLOUDFLARE_API_TOKEN` | Cloudflare → My Profile → API Tokens → Create Token → **"Edit Cloudflare Workers"** template → scope: **Workers Scripts:Edit** tylko dla projektu `10xcards` |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare dashboard → prawy sidebar → Account ID (ten sam co z CP-1.2) |

- [ ] **CP-6.1** Wszystkie 4 sekrety widoczne w GitHub repo → Settings → Secrets → Actions

#### 6b. Test CI pipeline `(A)`

Wykonaj mały commit do `master` (np. dodaj komentarz do `README.md`):

```bash
git add README.md && git commit -m "chore: test CI deploy pipeline" && git push
```

- [ ] **CP-6.2** GitHub Actions → CI job (`ci`) → zielony (lint + build przeszły)
- [ ] **CP-6.3** GitHub Actions → Deploy job (`deploy`) → zielony (`wrangler deploy` przez CI)
- [ ] **CP-6.4** `npx wrangler deployments list` — widoczny nowy deployment (z GitHub Actions)
- [ ] **CP-6.5** Aplikacja nadal działa pod tym samym URL po auto-deploy

---

### FAZA 7 — Naprawa `.dev.vars` dla lokalnego dev `(U)`

Cel: ujednolicić format URL w lokalnym środowisku.

Zaktualizuj `.dev.vars`:

```diff
- SUPABASE_URL=https://fjjmtkirumucazxxvghr.supabase.co/rest/v1/
+ SUPABASE_URL=https://fjjmtkirumucazxxvghr.supabase.co
```

- [ ] **CP-7.1** `.dev.vars` zaktualizowany
- [ ] **CP-7.2** `npm run dev` → `wrangler dev` działa lokalnie z poprawionym URL
- [ ] **CP-7.3** Lokalny signup/signin działa (auth w dev mode z `enable_confirmations = false`)

---

## Rollback

```bash
npx wrangler deployments list          # lista deploymentów
npx wrangler rollbacks create          # rollback do poprzedniego
```

- Rollback cofa kod Workers — nie cofa zmian w Supabase
- Czas rollback: < 30 sekund globalnie

---

## Pliki zmienione przez agenta (już wykonane)

| Plik | Zmiana |
|---|---|
| `wrangler.jsonc` | `"name"` zmieniony z `10x-astro-starter` na `10xcards` |
| `.github/workflows/ci.yml` | Dodany job `deploy` (auto-deploy na push do master) |

---

## Powiązane dokumenty

- `context/foundation/infrastructure.md` — decyzja platformowa i pełny risk register
- `context/foundation/tech-stack.md` — stack constraints
