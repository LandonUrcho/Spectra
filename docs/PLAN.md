# Spectra → resume-grade vulnerability intelligence app

## Context

Spectra (repo `LandonUrcho/Spectra`, branch `main` @ `05c1b1c`) is a working CVE dashboard: a FastAPI backend (`cve-backend/main.py`, 363 lines, one endpoint `GET /api/vendor/{name}`, data from CIRCL Vulnerability-Lookup) and a React 19 / Vite 8 / Tailwind 4 / TypeScript frontend (`cve-frontend/src`, about 800 lines). Both are live on Vercel Hobby:
- Frontend project `spectra-frontend`: https://spectra-frontend-theta.vercel.app
- Backend project `cve-backend`: https://cve-backend.vercel.app

Both auto-deploy from `main`.

Today it is a good-looking, single-screen demo, with:
- no tests, CI, routing, persistence, enrichment or README;
- a 10-minute in-memory cache;
- a silent fallback to mock data;
- up to 17 sequential CIRCL calls per request.

**Goal:** make it a standout project for a **full-stack / generalist SWE** résumé. The work should show product thinking, a typed end-to-end contract, real system design (database, cache, rate limits, scheduled ingestion, graceful degradation), one well-built AI feature, tests and CI, and a recruiter-ready README.

**Decisions made with the user:**
- **Persistence:** Postgres (Neon) and Redis (Upstash) through the Vercel Marketplace. No login; watchlists live in the browser and in share links.
- **AI feature:** an AI **CVE explainer** on **`claude-haiku-5-5`** with structured output, cached once per CVE in Postgres.
- **Layout:** an **app shell with a nav rail**, plus a **CVE inspector drawer** that has its own URL.
- **Delivery:** **phased PRs**. Each PR can be merged and deployed on its own, and the app stays coherent if work stops after any PR.

## Target architecture

```
Browser (React SPA on Vercel)
  ├─ React Router (URL state) + TanStack Query (client cache, dedupe, retries)
  └─ typed client generated from FastAPI's OpenAPI (openapi-typescript + openapi-fetch)
        │ HTTPS + CORS
FastAPI on Vercel Python functions (cve-backend)
  ├─ api/        resource routes, Pydantic response models, proper status codes
  ├─ services/   vendor aggregation, enrichment, priority scoring, explainer
  ├─ clients/    CIRCL, FIRST EPSS, CISA KEV, Anthropic (shared httpx.AsyncClient, retries, UA)
  ├─ cache       Upstash Redis (cache-aside, stale-if-error, request coalescing, rate limits)
  │              └─ falls back to an in-memory TTL/LRU implementation when env vars are absent
  ├─ db          Neon Postgres via SQLAlchemy 2 async + psycopg3, Alembic migrations
  │              └─ DB features disable gracefully when DATABASE_URL is absent
  └─ cron        Vercel daily cron → /api/cron/ingest (KEV sync, EPSS refresh, vendor snapshots)
```

**Degradation rule:** every new dependency is optional at runtime. With no `DATABASE_URL`, no Redis env vars and no `ANTHROPIC_API_KEY`, the app still runs: it uses in-memory caching, skips persistence, and hides the AI button. This means:
- PRs can merge before the user provisions anything;
- local development and CI need no external services;
- tests never touch the network.

## Key decisions and why (alternatives considered)

Each entry gives what was chosen, why, and what was rejected and why. The implementing session copies this reasoning into ADRs (`docs/adr/`) and a short "Design decisions" section in the README. Recruiters and interviewers ask about exactly these trade-offs.

### System design

**1. Hosting: stay on Vercel, with two projects (frontend + Python functions).**
- **Why:** it already works, auto-deploys from `main`, costs nothing on Hobby, and gives preview deploys per PR. Changing hosts adds work without making the résumé story better.
- **Rejected:**
  - *Rewrite as Next.js full-stack:* throws away the FastAPI backend, which shows a second language and API design.
  - *Render, Fly or AWS (ECS/Lambda):* more ops story, but adds IaC, cost and cold-start trade-offs that would eat the session's time.
  - *Merging into one Vercel project with `/api` rewrites:* removes CORS, but it's uncertain whether the client IP is forwarded correctly, which rate limiting needs. The two-project setup is already verified.

**2. Every external dependency is optional at runtime (graceful degradation).**
- **Why:**
  - PRs can merge and deploy before the user provisions Neon, Upstash or Anthropic;
  - local development and CI need zero services;
  - it shows deliberate resilience design: the app degrades instead of crashing.
- **Rejected:** *hard dependencies.* Simpler code, but one missing env var takes production down, and every contributor would need cloud accounts to run tests.

**3. Postgres (Neon) as the system of record.**
- **Why:** the data is relational:
  - CVEs ↔ KEV entries ↔ daily EPSS scores ↔ cached AI explanations;
  - an EPSS history keyed on (cve, date);
  - joins for "KEV CVEs with rising EPSS".

  Postgres is also what most job descriptions list, and JSONB holds the AI output. Neon is serverless (scales to zero, free tier), is one click in the Vercel Marketplace, and has a pooled endpoint suited to serverless functions.
- **Rejected:**
  - *No DB, cache only:* can't keep EPSS history or ingestion runs, and paid AI output would be lost when the cache evicts it.
  - *Supabase:* also Postgres, but bundles auth and storage we don't need.
  - *SQLite/Turso:* fine technically, but less recognizable and a weaker system-design story.
  - *MongoDB:* the data isn't document-shaped, and joins matter here.

**4. Redis (Upstash) in addition to Postgres.**
- **Why:**
  - Vercel functions are many short-lived instances that **don't share memory**. Today's in-process dict cache is mostly cold, a weakness CLAUDE.md already notes.
  - Rate-limit counters and request-coalescing locks need **atomic, shared, TTL-based** state, which is Redis's core use case.
  - Upstash's HTTP API avoids running out of TCP connections in serverless.
- **Rejected:**
  - *Caching in Postgres:* no native TTL, and every cache hit becomes a DB round trip plus connection churn.
  - *Vercel edge cache only:* we still use it (decision 6), but it can't do rate limiting, locks or stale-if-error fallback per key.
  - *Vercel KV:* no longer a first-party product; it's served through Upstash anyway.

**5. Cache strategy: cache-aside + stale-if-error + request coalescing.**
- **Why:**
  - CIRCL is a free community API: slow (a cold vendor is about 17 calls) and sometimes down.
  - Cache-aside keeps the code simple.
  - A long-lived "last good" copy lets us serve slightly stale data with a visible `stale` badge instead of the current *mock data presented as live*, which is a correctness bug.
  - Coalescing (a `SET NX` lock) stops a cold vendor from causing 50 parallel fan-outs (thundering herd).
- **Rejected:**
  - *Pre-warming every vendor:* there are tens of thousands of vendors, so the cron can't cover the long tail.
  - *Write-through:* we don't own the source data, so there's nothing to write through.
  - *Keeping the mock fallback:* it silently lies to users.

**6. HTTP `Cache-Control: s-maxage` + `stale-while-revalidate` on read endpoints.**
- **Why:** Vercel's CDN then serves repeat requests without invoking a function at all. This is nearly free, cuts latency and cost, and is the cheapest layer in a layered caching design.
- **Rejected:** *no HTTP caching:* every page view would invoke a function.

**7. Async I/O with `httpx.AsyncClient` + `asyncio.Semaphore(8)`.**
- **Why:** the backend does I/O-bound fan-out. Async gives concurrency without threads, one shared client reuses connections, and the semaphore keeps us polite to CIRCL. Retries with backoff handle transient 5xx errors.
- **Rejected:**
  - *Keep `requests` + `ThreadPoolExecutor`:* works, but doesn't reuse connections, blocks FastAPI's event loop in sync routes, and is the less modern pattern.
  - *aiohttp:* similar, but httpx has a cleaner API and `respx` for test mocking.

**8. SQLAlchemy 2 (async) + psycopg3 + Alembic.**
- **Why:** these are the industry-standard Python data stack, with typed ORM models and versioned migrations. Serverless needs two settings:
  - `NullPool`, because each function instance mustn't hold a pool;
  - `prepare_threshold=None`, because Neon's pgbouncer runs in transaction mode and breaks prepared statements.
- **Rejected:**
  - *SQLModel:* thinner, but adds an abstraction over SQLAlchemy and weaker Alembic support.
  - *Raw SQL:* no migrations story.
  - *`metadata.create_all()` at startup:* not how production schemas evolve.

**9. Migrations run from GitHub Actions on push to `main`.**
- **Why:** Vercel has no "release phase". Running migrations in CI makes them auditable, runs them once (not per function instance), and uses the direct, non-pooled Neon URL that DDL needs. It's also a real CI/CD story.
- **Rejected:**
  - *Running at app startup:* every cold start would race to migrate.
  - *Manual migrations from a laptop:* not reproducible.

**10. Ingestion: lazy on-request enrichment + a daily Vercel cron.**
- **Why:**
  - EPSS publishes once a day and KEV changes a few times a week, so a daily schedule loses nothing. Vercel Hobby only allows daily crons anyway.
  - Lazy enrichment (batch-fetching EPSS for unseen CVEs while serving a request) keeps any vendor fresh even if it was never ingested.
  - The cron route is a GET, because Vercel cron sends GET, and it's protected by `CRON_SECRET`.
- **Rejected:**
  - *GitHub Actions schedule:* hourly is possible, but adds a second secret store and no data benefit.
  - *Upstash QStash or a worker queue:* overkill at this scale.
  - *On-demand only:* no EPSS history and no "new in KEV this week".

**11. Data sources: FIRST EPSS + CISA KEV directly, CIRCL for CVE records.**
- **Why:**
  - These are the canonical sources.
  - FIRST's EPSS API takes **100 CVEs per request**, while CIRCL's `/epss/{id}` takes one.
  - KEV is a single roughly 1.5k-entry JSON file that we sync whole.
  - CIRCL stays the source for CVE details, timeline and vendor browsing, which nothing else offers for free.
- **Rejected:**
  - *All through CIRCL:* N+1 EPSS calls and an extra hop for KEV.
  - *NVD API:* stricter rate limits without a key, and no vendor browse.

**12. Rate limiting: a sliding window per IP in Redis, plus a global daily AI budget.**
- **Why:** each uncached request fans out to about 17 upstream calls, and AI calls cost money. A sliding window avoids the burst-at-boundary problem fixed windows have. A separate global budget caps the worst-case daily spend, whatever the IP mix.
- **Rejected:**
  - *Fixed window:* burst problems at each window boundary.
  - *The Vercel Firewall:* its rate-limiting rules are a paid feature and not visible in code.
  - *No limits:* a single scraper could exhaust CIRCL's goodwill or the AI budget.

**13. Risk prioritization: transparent tier rules (KEV → EPSS → CVSS) plus a sortable score.**
- **Why:**
  - CVSS measures severity, not likelihood. Most CVEs with high CVSS are never exploited, while KEV entries are known to be. Combining them is current industry practice (SSVC, EPSS guidance).
  - Rules are explainable (each row has `priority_reasons`) and unit-testable.
- **Rejected:**
  - *Sorting by CVSS only:* what every tutorial dashboard does.
  - *A full SSVC decision tree:* needs per-organization inputs we don't have, such as asset exposure and mission impact.
  - *An ML model:* no labeled data, and it can't be explained in an interview.

### API and frontend

**14. A typed contract: Pydantic `response_model` → OpenAPI → `openapi-typescript`, with a CI drift check.**
- **Why:** one source of truth for types. A backend change that breaks the frontend fails CI instead of production. FastAPI already emits OpenAPI, so it costs almost nothing.
- **Rejected:**
  - *Hand-written TS types (today):* they drift silently.
  - *Zod validation at runtime only:* duplicates the schema.
  - *GraphQL or tRPC:* tRPC needs a TS backend, and GraphQL is overkill for about 10 read endpoints.

**15. Resource-oriented REST endpoints with real status codes.**
- **Why:** `404`, `422`, `429`, `502` and `503` let the frontend and the CDN behave correctly. Today every outcome is a `200`, so "vendor doesn't exist" looks the same as "CIRCL is down".
- **Rejected:** *keeping one catch-all `/api/vendor/{name}` that always returns 200:* errors hidden in response bodies can't be cached or monitored.

**16. React Router + TanStack Query.**
- **Why:**
  - **URL state:** shareable links, a working Back button, deep links into products and CVEs.
  - **Query** gives caching, deduplication, retries, background refetch and cancellation. It replaces the hand-rolled `vendor#attempt` effect pattern, which exists only to work around a lint rule.
- **Rejected:**
  - *TanStack Router:* excellent type safety, but less familiar to reviewers and a steeper migration.
  - *SWR:* fine, but Query's devtools and mutation support suit the AI POST better.
  - *Redux:* this is server state, not client state.

**17. The CVE drawer as a nested route, not local modal state.**
- **Why:** the drawer has its own URL, so Back closes it, links are shareable and reload works. Context (the table) stays visible, which matches how security analysts triage. The same `CveDetail` component renders full-page on mobile and on direct links.
- **Rejected:**
  - *Local `useState` modal:* no URL and no Back.
  - *The "background location" modal-route pattern:* more fragile than plain nested routes.
  - *Full page only:* loses triage context.

**18. Watchlist in `localStorage` + share links (no accounts).**
- **Why (the user's choice):** delivers about 80% of the value of saved lists at about 20% of the cost. No auth surface to secure, and nothing for a recruiter to sign up for. Share links (`/watchlist?v=…`) make lists portable.
- **Rejected:** *GitHub OAuth plus a server-side watchlist:* about 2x the work, adds session and CSRF security concerns, and adds friction for demo viewers.

**19. Hand-rolled SVG for the sparkline (no chart library).**
- **Why:** only one small chart is needed. Inline SVG with design tokens stays on-brand in both themes, adds about 0 KB, and continues the existing CSS-bar approach.
- **Rejected:** *Recharts or Chart.js:* 50–100 KB+ for one sparkline, and harder to theme.

### AI feature

**20. CVE explainer, not natural-language search (the user's choice).**
- **Why:** each output is tied to a stable key (the CVE ID), so it can be cached once in Postgres and the cost is bounded. Output is grounded in data we fetched. It is easy to evaluate and to defend in an interview.
- **Rejected:** *NL search:* every query is unique (poor caching), and errors in the generated filters are harder to detect.

**21. Model: `claude-haiku-5-5`, configurable via `ANTHROPIC_MODEL` (the user's choice).**
- **Why:** about $0.001 per explanation versus about $0.03–0.05 on `claude-opus-5-5`. A summary-and-remediation task doesn't need frontier reasoning. The env var allows a switch without a code change.
- **Note:** Haiku 5.5 has no server-side refusal fallback, so refusals map to a friendly `422`.

**22. Structured output via `client.messages.parse(output_format=PydanticModel)`.**
- **Why:** guaranteed-valid JSON matching a Pydantic schema, so the UI renders typed sections with no parsing errors, and the same model documents the API.
- **Rejected:**
  - *Free-text markdown:* can't render it in a structured way or validate it.
  - *Forced tool use:* not supported on current models (400), and structured output is the intended mechanism.

**23. Cost control: a DB cache per (cve, model, prompt_version), not prompt caching.**
- **Why:** the expensive part is generating the same explanation again. Postgres turns repeat requests into $0 reads. Prompt caching wouldn't help: our system prompt is below the minimum cacheable prefix size, and each CVE's input is unique. `prompt_version` in the key lets us regenerate cleanly when the prompt changes.

**24. Grounding and safety: inputs only from fetched data, CVE text treated as untrusted, references filtered.**
- **Why:**
  - CVE descriptions are third-party text: a prompt-injection vector.
  - The model is asked for defensive, non-operational guidance.
  - Any reference URL it returns that wasn't in the input is dropped.

  Together these show AI-safety awareness, which is a 2026 hiring signal.

### Process

**25. Eight phased PRs, foundation first.**
- **Why:**
  - Tests, CI, typed contracts and the data layer come first, so every later feature lands on a safety net.
  - The order follows dependencies: enrichment needs the DB, the drawer needs enrichment, the AI needs the drawer.
  - Each PR deploys a coherent app, so stopping early still leaves a strong project.
  - A readable PR history is itself a résumé artifact.
- **Rejected:** *one big PR:* hard to review, roll back or demo partially.

**26. Test strategy: recorded fixtures, with no live network in tests.**
- **Why:** deterministic CI that doesn't depend on CIRCL uptime or rate limits.
  - Backend: `respx` mocks httpx.
  - Frontend: MSW for component and hook tests.
  - E2E: Playwright `page.route()`, which needs no extra tooling.
  - A fake injected Anthropic client means no API spend in CI.
- **Rejected:**
  - *E2E against Vercel previews:* they're behind Vercel Authentication, so CI would need a bypass secret.
  - *Live API tests:* flaky.

## One-time manual steps for the user

The cloud session cannot do these: they need Vercel dashboard, marketplace terms or GitHub settings. Each step is needed before the PR named in brackets is **deployed with full features**. The code does not depend on them.

1. **[PR 3] Neon Postgres:** Vercel → `cve-backend` → Storage → add **Neon** (free) and connect it to all environments. This injects `DATABASE_URL` (pooled) and `DATABASE_URL_UNPOOLED`.
2. **[PR 3] Upstash Redis:** add **Upstash for Redis** (free) the same way. This injects `KV_REST_API_URL` and `KV_REST_API_TOKEN`; some versions name them `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN`, so the code accepts both.
3. **[PR 3] Migration secret:** GitHub repo → Settings → Secrets → `DATABASE_URL_UNPOOLED` (the Neon direct URL). The migrate workflow uses it.
4. **[PR 4] Cron secret:** add `CRON_SECRET` (a random 32+ character string) to the `cve-backend` env.
5. **[PR 6] Anthropic key:** add `ANTHROPIC_API_KEY` to the `cve-backend` env. Optionally set a spend limit in the Anthropic Console.
6. After the env changes, redeploy `cve-backend`. Vercel only applies env vars on the next deploy.

## Conventions for the implementing session

- **Decisions are binding but not blind:** before each PR, re-read the decision entries it relies on. If reality contradicts one (a library is missing, an API changed, a version isn't supported on Vercel), take the next listed alternative and record the change and its reason in the PR description and the relevant ADR.
- **Read first:** `CLAUDE.md`. Keep its conventions:
  - design tokens in `src/index.css` `@theme`;
  - the restrained style (no glows, gradients or uppercase tracking);
  - pinned `requirements.txt`.
- **Branches and PRs:** one branch and one PR per phase, named `feat/NN-slug`, each based on the previous PR merged into `main` (or stacked if not merged yet). Each PR updates `README.md`, `CLAUDE.md` and `.env.example` for what it adds.
- **Dependency versions:** pin to the latest stable version **at implementation time**, checked with `pip index versions` and `npm view`. Never guess versions.
- **Network access:** if the sandbox can't reach CIRCL, FIRST or CISA, write test fixtures by hand from the documented shapes below. Tests must not need the network.
- **Keep the existing API working:** keep `GET /api/vendor/{name}` as a deprecated alias until PR 2's frontend has shipped, then remove it in PR 8. The two projects deploy independently, so a brief contract mismatch would break production.
- **Done for every PR:**
  - CI is green;
  - `npm run build` and `npm run lint` pass;
  - `pytest`, `ruff check` and `mypy` pass;
  - the Vercel preview builds;
  - the PR description says what changed and how it was verified.

## External APIs (verified)

- **CIRCL** `https://vulnerability.circl.lu/api`:
  - `GET /browse/vendors?q=&per_page=`: vendor autocomplete.
  - `GET /browse/vendors/{v}/products`
  - `GET /vulnerability/search/{vendor}/{product}?page=&per_page=`
  - `GET /vulnerability/{id}?with_meta=true`
  - `GET /vulnerability/{id}/timeline`
  - `GET /stats/vendors/ranking?limit=&period=`
  - `POST /vulnerability/bulk`: up to 100 IDs.
  - `GET /epss/{id}`
- **FIRST EPSS** `https://api.first.org/data/v1/epss`:
  - `?cve=ID1,ID2,…`: batch of up to 100. Scores come back as **strings**.
  - `?cve=ID&scope=time-series`: last 30 days.
- **CISA KEV** `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`: one JSON file, about 1.5k entries, with `cveID`, `dateAdded`, `dueDate`, `knownRansomwareCampaignUse`, `shortDescription` and `requiredAction`.

---

## PR 1 — Backend foundation (no visible feature change)

Restructure `cve-backend/main.py` into a package while keeping `main.py` as the Vercel entry point (`from app.main import app`):

```
cve-backend/app/{main.py, config.py, logging.py, errors.py, schemas.py}
cve-backend/app/api/{health.py, vendors.py}
cve-backend/app/clients/circl.py
cve-backend/app/services/vendor_service.py
cve-backend/tests/{conftest.py, fixtures/*.json, test_parsing.py, test_vendor_api.py}
cve-backend/{pyproject.toml (ruff, mypy, pytest config), requirements-dev.txt}
```

**Moves and replacements:**
- Move the pure helpers out of `main.py` unchanged first, then add tests for them: `extract_score`, `extract_summary`, `belongs_to_vendor`, `display_name`, `resolve_vendor`, `_quote`.
- Replace `requests` and the `ThreadPoolExecutor` with one shared **`httpx.AsyncClient`**:
  - timeouts;
  - retries with exponential backoff on 5xx errors and timeouts;
  - an `asyncio.Semaphore(8)` around fan-out;
  - a `User-Agent: Spectra/0.x (+github.com/LandonUrcho/Spectra)` header.
- `config.py`: use `pydantic-settings`. Move `ALLOWED_ORIGINS` and `ALLOWED_ORIGIN_REGEX` there unchanged, keep the localhost dev origins, and add `POST` to the CORS methods now (PR 6 needs it).
- **Pydantic response models** with `response_model=` on every route, so the OpenAPI schema is real.
- **Error semantics:**
  - Remove `MOCK_DATA` and `USE_LIVE_API`.
  - Unknown vendor → `404`.
  - CIRCL down → serve the last good cached payload with `stale: true`; if there is none, `502`.
  - Invalid input (vendor name longer than 64 characters or with odd characters) → `422`.
- **New routes:**
  - `GET /api/health`: shows which dependencies are configured and reachable.
  - `GET /api/meta`: returns `{ai_enabled, db_enabled, last_ingest_at}`, all false or null for now.
  - `GET /api/vendors?q=`: autocomplete, proxying CIRCL browse with a cache.
  - `GET /api/vendors/{vendor}`: the same payload as today plus `stale` and `fetched_at`.
- **Observability:** JSON structured logging, a request-ID middleware (`X-Request-ID`), and a `Server-Timing` header.
- **Tests:** `pytest` + `respx` with recorded CIRCL fixtures. Cover the parsing helpers, 404/502/stale paths, and vendor resolution.
- **CI:** `.github/workflows/ci.yml` with a backend job that runs `ruff check`, `ruff format --check`, `mypy app`, and `pytest --cov`. Add `.github/dependabot.yml` for pip, npm and github-actions.
- **Docs:** root `README.md` skeleton (pitch, live links, stack, local setup), `LICENSE` (MIT), `.env.example`. Fix the stale `CLAUDE.md` line 6 (donut chart and pagination no longer exist).

## PR 2 — Frontend foundation and app shell

- **Libraries:** `react-router` (library mode, `createBrowserRouter`), `@tanstack/react-query`, `openapi-typescript` and `openapi-fetch`.
- **Typed API contract:**
  - `cve-backend/scripts/export_openapi.py` writes `openapi.json`.
  - `npm run gen:api` turns it into `src/api/schema.d.ts`.
  - The CI job **fails if the generated types drift** from the backend.
- **Replace hand-written types:** swap the types in `src/api.ts` for the generated ones. Replace the `vendor#attempt` effect pattern in `App.tsx` with Query hooks (`useVendor`, `useVendorSearch`).
- **Routes:**
  - `/` Overview (placeholder until PR 7).
  - `/vendors/:vendor` and `/vendors/:vendor/:product`: today's dashboard, with the selected product in the URL.
  - `/watchlist` (placeholder).
  - `/methodology` (placeholder).
  - A 404 page.
  - Add `cve-frontend/vercel.json` with an SPA rewrite `{"source":"/(.*)","destination":"/index.html"}`. Without it, deep links 404 on Vercel.
- **App shell:**
  - **Left nav rail** (Overview, Vendors, Watchlist, Methodology), which becomes a bottom bar under `md`.
  - The top bar keeps search, which becomes a **debounced typeahead** over `/api/vendors?q=`. Typing a `CVE-YYYY-NNNN` jumps to that CVE, and `⌘K` and `/` focus the search.
  - A **light/dark theme toggle**: add a light token set under `[data-theme="light"]` in `index.css`, persist the choice in `localStorage` with try/catch, and default to `prefers-color-scheme`.
- **Fixes:**
  - Raise the contrast of `--color-subtle` to at least 4.5:1 (it's about 3.9 today).
  - Add a skip link.
  - Wrap the app in a React error boundary.
  - Lazy-load the route chunks.
- **Cleanup:** delete the unused `tailwind.config.js` and the redundant `autoprefixer`, and replace the boilerplate `cve-frontend/README.md` with a pointer to the root README.
- **Tests:** Vitest + Testing Library + MSW for components and hooks (`CveTable` sort, severity bands, search keyboard behavior). Add Prettier.
- **CI:** a frontend job that runs `npm ci`, lint, `tsc -b`, `vitest run`, build, and the OpenAPI drift check.

## PR 3 — Data layer: Postgres, Redis, caching, rate limits

- `app/db/`:
  - SQLAlchemy 2 async with `postgresql+psycopg`, `NullPool` (serverless), and `prepare_threshold=None`, which the Neon pooler (transaction mode) requires.
  - Alembic in `cve-backend/migrations/`.
- **Tables:**
  - `cves`: id PK, vendor, product, summary, cvss_score, cvss_version, cvss_vector, cwe, published, updated_at, fetched_at.
  - `kev_entries`
  - `epss_scores`: cve_id, date, epss, percentile; PK (cve_id, date).
  - `vendor_snapshots`: vendor, date, totals by severity, kev_count.
  - `ai_explanations`: cve_id, model, prompt_version, content jsonb, tokens, created_at.
  - `ingest_runs`: started, finished, status, counts, error.
- **`app/cache.py`:**
  - A `Cache` protocol with two implementations: `UpstashCache` (`upstash-redis` async) and `MemoryCache` (TTL + LRU, bounded size).
  - Cache-aside for vendor payloads (TTL 10 minutes), with a long-lived "last good" key for stale-if-error.
  - **Request coalescing** via `SET NX` locks, so 50 concurrent cold requests make one CIRCL fan-out.
  - `Cache-Control: public, s-maxage=300, stale-while-revalidate=600` on read endpoints, so Vercel's edge cache absorbs traffic.
- **`app/ratelimit.py`:**
  - A sliding window per client IP, taken from the first `x-forwarded-for` hop; use `upstash-ratelimit`, or a memory fallback.
  - 60 requests/min on read endpoints; return `429` with `Retry-After`.
- **Persistence:** upsert CVEs fetched for a vendor into `cves`; this fire-and-forget write must not block the response.
- **Pagination:** `GET /api/vendors/{vendor}/products/{product}/cves?page=` passes CIRCL's `page` through. The frontend adds a "Load more" button.
- **CI:** `.github/workflows/migrate.yml` runs `alembic upgrade head` on push to `main` when migration files change, using the `DATABASE_URL_UNPOOLED` secret. Backend tests use the memory cache plus SQLite or a mocked repository; no external Postgres in CI.
- **ADRs:**
  - `docs/adr/001-data-stores.md`: decisions 3, 4, 8 and 9.
  - `docs/adr/002-caching-rate-limits-and-degradation.md`: decisions 2, 5, 6 and 12.
  - Each is half a page: context, decision, alternatives, consequences.

## PR 4 — Enrichment and risk-based prioritization (the core differentiator)

- **Clients:** `clients/epss.py` (batch of 100, parse strings to floats) and `clients/kev.py`.
- **`services/enrichment.py`:**
  - Every CVE payload gains `epss`, `epss_percentile`, `kev` (with `date_added`, `due_date` and the ransomware flag), `priority` and `priority_score`.
  - EPSS is read from the DB or cache, and unknown CVEs are fetched in one batch.
  - KEV membership is read from the DB or a cached set of IDs.
- **`services/priority.py`:** a pure function with a full unit-test table. Tiers:
  - **P1 Act now:** in CISA KEV.
  - **P2 High:** EPSS ≥ 0.10, or (CVSS ≥ 9.0 and EPSS percentile ≥ 0.90).
  - **P3 Moderate:** CVSS ≥ 7.0, or EPSS percentile ≥ 0.75.
  - **P4 Low:** everything else.
  - `priority_score` (0–100, for sorting within a tier) = KEV ? 100 : round(100 × (0.6·epss_percentile + 0.4·cvss/10)).
  - The response also includes a `priority_reasons: list[str]` field, used in tooltips.
- **Cron:**
  - `cve-backend/vercel.json`: `{"crons":[{"path":"/api/cron/ingest","schedule":"0 7 * * *"}]}`. Hobby plans only allow daily crons, and EPSS publishes daily.
  - The route is a **GET** (Vercel cron sends GET) that requires `Authorization: Bearer ${CRON_SECRET}`.
  - Steps: sync KEV, then refresh EPSS for KEV CVEs plus CVEs seen in the last 30 days (capped at 2,000, in batches), then snapshot the preset vendors, then write an `ingest_runs` row.
  - `/api/meta.last_ingest_at` reads from `ingest_runs`.
- **Frontend, vendor view:**
  - Table columns become **Priority · CVE · CVSS · EPSS (percentile bar) · KEV badge · Published**.
  - A filter bar: severity chips, a "KEV only" toggle, a text filter, a date range. Filters live in URL search params.
  - Summary tiles add a KEV count and a P1 count.
  - **Export CSV/JSON** of the filtered rows.
- **`/methodology` page:**
  - The data sources, refresh cadence, the priority formula and its rationale (CVSS alone is a poor predictor of exploitation; EPSS plus KEV, in the spirit of SSVC), and the known limitations.
- **ADR:** `docs/adr/003-ingestion-and-risk-prioritization.md`: decisions 10, 11 and 13.

## PR 5 — CVE inspector drawer and detail page

- **Backend** `GET /api/cves/{id}` combines four sources:
  - CIRCL `/vulnerability/{id}`: description, CVSS vector, CWE, affected versions, references;
  - `/vulnerability/{id}/timeline`;
  - EPSS 30-day `scope=time-series`, cached for 24 hours;
  - KEV.
  - It returns `404` for unknown IDs and validates the ID with `^CVE-\d{4}-\d{4,}$`.
- **Frontend routing:**
  - The nested route `/vendors/:vendor/:product/cve/:id` renders the **drawer** (right panel, at least `lg`) next to the table via `<Outlet/>`. Selecting a row is a link, so Back closes the drawer and URLs are shareable.
  - The standalone `/cve/:id` renders the same `CveDetail` component as a full page. Under `lg`, the nested route also renders full-page.
  - The drawer is accessible: focus moves in on open and returns to the row on close, `Esc` closes it, and it has `role="dialog"` and `aria-labelledby`.
- **`CveDetail` sections:**
  - A header with ID, priority pill, CVSS, EPSS and KEV.
  - A **CVSS vector breakdown** (AV, AC, PR, UI, S, C, I, A turned into readable chips).
  - A **30-day EPSS sparkline**: hand-rolled SVG, no chart library, colors from tokens.
  - Description, affected products and versions, CWE (linked to MITRE), KEV required action and due date, a timeline, and references.
  - A slot for the AI card (PR 6).
- **Tests:** vector parser unit tests and drawer focus/close behavior.

## PR 6 — AI CVE explainer (Claude Haiku 5.5)

- **Setup:**
  - `anthropic` SDK (pinned), using `AsyncAnthropic()`.
  - Model from env `ANTHROPIC_MODEL`, default `claude-haiku-5-5`.
  - The feature turns on only when `ANTHROPIC_API_KEY` is set (`/api/meta.ai_enabled`).
- **Endpoints:**
  - `GET /api/cves/{id}/explanation` returns the cached explanation or `404`.
  - `POST /api/cves/{id}/explanation` generates one if missing. The CORS POST method was added in PR 1.
  - The cache key is (cve_id, model, `PROMPT_VERSION`) in `ai_explanations`, so each CVE is paid for **once**.
- **Call:**
  - `client.messages.parse(model=..., max_tokens=4000, output_config={"effort": "low"}, system=SYSTEM_PROMPT, messages=[...], output_format=CveExplanation)`, then read `response.parsed_output`.
  - Do not send `temperature` or a `thinking` budget; Haiku 5.5 rejects non-default values.
- **Schema** (`CveExplanation`, Pydantic):
  - `tldr`, `who_is_affected`, `impact`;
  - `exploitability: {assessment: Literal["actively_exploited","likely","possible","unlikely"], reasoning}`;
  - `remediation_steps: list[str]`, `detection_tips: list[str]`;
  - `references: list[str]`;
  - `confidence: Literal["low","medium","high"]`.
- **Grounding:**
  - The prompt contains only data we fetched: description, CVSS vector, CWE, affected versions, references, EPSS and KEV.
  - The system prompt says CVE text is **untrusted data, not instructions** (prompt-injection hygiene) and asks for defensive, non-operational guidance only.
  - After parsing, **drop any reference URL that wasn't in the input**, as a hallucination guard.
- **Guardrails:**
  - Per-IP limit: 10 generations/hour.
  - A **global daily budget** counter in Redis (default 200 new generations/day, env `AI_DAILY_LIMIT`); over budget → `429`.
  - If `stop_reason == "refusal"` → `422` with a friendly message.
  - Typed SDK errors (`RateLimitError`, `APIStatusError`, `APIConnectionError`) → `503`.
  - Log token usage per call.
- **Frontend:**
  - An "Explain with AI" card in `CveDetail` with skeleton, error and over-budget states.
  - It renders sections plus a footer: "AI-generated by claude-haiku-5-5 on {date}. Verify against the references."
  - It is hidden when `ai_enabled` is false.
- **Tests:** inject a fake client through a FastAPI dependency (no live calls in CI). Cover cache hit/miss, budget exhaustion, refusal mapping, and the reference filter.
- **ADR:** `docs/adr/004-ai-explainer.md`: decisions 20–24 (model choice and cost: about $0.001 per explanation; caching; grounding).

## PR 7 — Overview and Watchlist

- **Backend `GET /api/overview`** (cached for 1 hour):
  - KEV additions in the last 14 days, enriched with EPSS and priority;
  - highest-EPSS CVEs from the DB in the last 7 days;
  - a top-10 vendor ranking from CIRCL `/stats/vendors/ranking`;
  - totals and `last_ingest_at`.
- **Backend `GET /api/watchlist?vendors=a,b,c`** (at most 10 vendors): reuses `vendor_service` and the cache, merges the results, and returns the top 50 CVEs sorted by `(priority, priority_score)`.
- **Frontend `/` Overview:**
  - KPI tiles (new KEV entries this week, P1 count across the watchlist, last refresh);
  - a "Recently added to CISA KEV" list;
  - an "EPSS movers" list;
  - a vendor leaderboard.
  - Rows open the CVE drawer.
- **Frontend `/watchlist`:**
  - A ☆ watch toggle on vendor pages, stored in `localStorage` (try/catch) through a `useWatchlist` hook.
  - A share link `/watchlist?v=cisco,fortinet` that imports into the visitor's list.
  - A merged, prioritized table with CSV export.
  - An empty state that suggests presets.
- **Tests:** `useWatchlist` (including share-link import) and the overview endpoint with fixtures.

## PR 8 — Showcase polish: e2e, security, analytics, README

- **Playwright e2e** (`cve-frontend/e2e/`):
  - Runs against `vite preview` with `page.route('**/api/**')` serving fixture JSON.
  - Covers: search → vendor → filter KEV-only → open drawer → AI card visible (mocked), plus deep-link reload and theme toggle.
  - `@axe-core/playwright` checks each main route and fails on serious violations.
  - Runs as a CI job.
- **Security headers** in `cve-frontend/vercel.json`: CSP (`connect-src` limited to the API origin and Vercel analytics), `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`. Add `@vercel/analytics` and `@vercel/speed-insights`; both are free on Hobby.
- **Cleanup:** remove the deprecated `GET /api/vendor/{name}` alias.
- **README.md** (the recruiter-facing artifact):
  - a one-line pitch, live demo link and screenshots;
  - a Mermaid **architecture diagram**;
  - a feature list;
  - "Engineering highlights": priority model, caching and degradation, typed contract, AI grounding and cost controls, CI;
  - a "Design decisions" section: a table with one row per major decision (choice, why, alternative rejected), condensed from this plan's decisions section and linking to the ADRs;
  - `docs/adr/005-typed-contract-and-frontend-architecture.md`: decisions 14–17.
  - local setup, a link to the API docs (`/docs`), the ADR index, and a roadmap.
- **Screenshots:** `npm run screenshots` (Playwright script against the production URL) writes `docs/screenshots/*.png` in light and dark themes. If the sandbox has no network access, leave the script and ask the user to run it.

---

## Explicitly out of scope (to keep the session efficient)

- user accounts or auth, and email/webhook alerts;
- Next.js or SSR migrations, microservices, Docker or Kubernetes;
- chart libraries (hand-rolled SVG is enough);
- rewriting git history to purge the old committed venv (destructive; the user can decide later);
- merging the two Vercel projects into one.

## Verification (each PR, plus end-to-end at the end)

1. **Local checks:**
   - backend: `cd cve-backend && pip install -r requirements-dev.txt && ruff check . && mypy app && pytest`;
   - frontend: `cd cve-frontend && npm ci && npm run lint && npm run build && npx vitest run`;
   - PR 8 and later: `npx playwright test`.
2. **Local run with zero config:**
   - `python main.py` and `npm run dev`;
   - load `http://localhost:5173/vendors/cisco`; it must work with no DB, Redis or AI env vars;
   - `/api/health` reports each dependency as "not configured".
3. **CI:** green on the PR. The OpenAPI drift check fails if the backend changed without `npm run gen:api`.
4. **Vercel preview:** the preview builds on both projects. Previews sit behind Vercel Authentication, so check them while logged in.
5. **After merging and deploying to production:**
   - `curl https://cve-backend.vercel.app/api/health`
   - `curl -i https://cve-backend.vercel.app/api/vendors/cisco`: check `Cache-Control`, `X-Request-ID`, `Server-Timing`, and the `priority`, `epss` and `kev` fields.
   - An unknown vendor returns `404`.
   - Load https://spectra-frontend-theta.vercel.app:
     - deep-link `/vendors/palo%20alto%20networks` and reload;
     - open a CVE drawer and check the URL changes and Back closes it;
     - check the console has no errors and the network tab has no CORS failures.
6. **Data features (after the user's manual steps):**
   - Trigger the cron once with `curl -H "Authorization: Bearer $CRON_SECRET" https://cve-backend.vercel.app/api/cron/ingest`.
   - Check `/api/meta.last_ingest_at` and the KEV badges.
   - Click "Explain with AI" twice on one CVE: the second click must come from cache (no new token usage logged).

## Sources consulted

- CIRCL Vulnerability-Lookup OpenAPI: https://vulnerability.circl.lu/api/swagger.json
- FIRST EPSS API: https://api.first.org/epss and https://www.first.org/epss/api
- CISA KEV feed: https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json
- Vercel cron limits (Hobby allows daily jobs only): https://vercel.com/docs/cron-jobs/usage-and-pricing
- OpenCVE features, for competitive context (subscriptions, EPSS/KEV filters): https://www.opencve.io/

## Résumé talking points this enables

- Built a full-stack vulnerability intelligence app (React/TypeScript and FastAPI on Vercel) that ranks CVEs by real-world exploitation risk (CISA KEV, FIRST EPSS, CVSS) instead of CVSS alone.
- Designed a serverless data layer with Postgres and Redis: cache-aside with stale-if-error, request coalescing, edge caching, per-IP rate limiting, and a daily ingestion cron.
- Shipped a grounded AI CVE explainer (Claude, structured output) with per-CVE caching, hallucination filtering and a global cost budget, at about $0.001 per explanation.
- Kept frontend and backend in sync with OpenAPI-generated types; CI covers lint, types, unit, e2e and accessibility tests.
