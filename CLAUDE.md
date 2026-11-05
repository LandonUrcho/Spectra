# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview
Spectra is a CVE dashboard. Pick a vendor (e.g. Cisco, VMware, Palo Alto) and it shows summary stats for the vendor, a list of its products, and the selected product's CVEs in a sortable table with a CVSS severity breakdown bar. It has two independent parts:

- `cve-backend/`: a FastAPI app in the `app/` package; `main.py` only re-exports `app` for Vercel and runs uvicorn locally. Dependencies are in `requirements.txt` (`fastapi`, `httpx`, `pydantic-settings`, `uvicorn`), installed in `cve-backend/.venv`. It needs Python 3.11+ (`pydantic-settings` does). Dev tools (ruff, mypy, pytest, pytest-cov, respx) are in `requirements-dev.txt`. Their config is in `pyproject.toml`, which holds tool config only (see Deployment). On this machine bare `python` can resolve to the Microsoft Store stub, so use `.venv/Scripts/python.exe`.
- `cve-frontend/`: React 19, TypeScript, Vite 8 and Tailwind v4 (via `@tailwindcss/postcss`; there's no `tailwind.config.js`). Fonts are self-hosted through `@fontsource` (Inter, JetBrains Mono). There's no chart library. Prettier formats the code (`printWidth` 120).

Tests never touch the network:
- Backend tests live in `cve-backend/tests/` (pytest). `respx` intercepts every request to CIRCL, and `tests/fixtures/` holds hand-written CIRCL responses. `conftest.py` builds a fresh app per test with `create_app()`, a `CirclClient` without backoff delays, and a fake clock for cache expiry. Async tests use the anyio pytest plugin (`pytestmark = pytest.mark.anyio`). Warnings fail the tests.
- Frontend tests sit next to the source as `*.test.ts(x)` (Vitest, Testing Library, jsdom). `src/test/setup.ts` unmounts rendered components after each test.

## Commands
Backend (from `cve-backend/`):
- `.venv/Scripts/python.exe main.py`: runs uvicorn on 0.0.0.0:8000. You can also use `.venv/Scripts/python.exe -m uvicorn main:app --reload --port 8000`.
- `.venv/Scripts/python.exe -m pip install -r requirements-dev.txt`: installs the runtime deps plus the dev tools
- `.venv/Scripts/python.exe -m pytest`: runs the tests. Add `--cov` for coverage. Run a single test with `-m pytest tests/test_parsing.py -k display_name`.
- `.venv/Scripts/python.exe -m ruff check .` and `-m ruff format .`: lint and format
- `.venv/Scripts/python.exe -m mypy`: type-checks `main.py`, `app/`, `scripts/` and `tests/` in strict mode (paths are set in `pyproject.toml`)
- `.venv/Scripts/python.exe -m scripts.export_openapi`: writes `cve-frontend/openapi.json`. Run it after changing a route or a response model, then `npm run gen:api` in `cve-frontend/`.

Frontend (from `cve-frontend/`):
- `npm install`
- `npm run dev`: Vite dev server on http://localhost:5173
- `npm run build`: `tsc -b && vite build`, so it also type-checks (test files included)
- `npm run lint`: ESLint flat config (typescript-eslint, react-hooks, react-refresh)
- `npm test`: runs Vitest once. `npm run test:watch` keeps it running; `npx vitest run src/severity.test.ts` runs one file.
- `npm run format` and `npm run format:check`: Prettier
- `npm run gen:api`: regenerates `src/api/schema.d.ts` from `openapi.json`. It runs `openapi-typescript` through a pinned `npx`, because the package's peer range (TypeScript 5) conflicts with our TypeScript 6 and can't be a dev dependency.

CI (`.github/workflows/ci.yml`) runs the lint, format, type-check, test and build commands above on pull requests and pushes to `main`, on Python 3.12 (Vercel's default) and Node 24. It also fails if `openapi.json` doesn't match the backend or `schema.d.ts` doesn't match `openapi.json`. Run them before pushing.

Dependabot (`.github/dependabot.yml`) opens weekly update PRs for pip, npm and GitHub Actions.

## Architecture
- **Backend layout** (`cve-backend/app/`):
  - `main.py`: `create_app(settings, circl=...)` wires middleware, routes and error handlers; the module-level `app` is what Vercel serves. State (the CIRCL client and `VendorService`) lives on `app.state`, and routes get it through the dependencies in `api/dependencies.py`.
  - `api/vendors.py`, `api/health.py`: routes. Every route declares a Pydantic `response_model` from `schemas.py`, so `/docs` and `/openapi.json` are exact.
  - `clients/circl.py`: `CirclClient` validates payload shapes at the boundary (vendor search returns typed `CirclVendor` entries) and uses one shared `httpx.AsyncClient` with an 8 s timeout, 3 attempts with exponential backoff on 5xx, timeouts and connection errors, a semaphore of 8 concurrent requests and a `Spectra/...` User-Agent. It starts a new client if the event loop changes (some serverless runtimes and every `TestClient` use a new loop). Every failure raises `UpstreamError`.
  - `services/vendor_service.py`: the lookup flow only; it hands caching to `MemoryCache.get_or_load`. `services/records.py`: pure functions over CIRCL payloads and CVE JSON 5 records (no I/O; test them directly).
  - `cache.py`: `MemoryCache.get_or_load(key, load)` owns the cache policy: a fresh hit, else `load()`, else (when `load` raises `UpstreamError`) the last good value as `Cached(value, stale=True)`. TTL plus LRU, per serverless instance. Callers never check freshness themselves.
  - `errors.py`: `VendorNotFoundError` → 404 and `UpstreamError` → 502, as `{"detail": ...}`.
  - `observability.py`: JSON logs to stdout, `X-Request-ID` (a safe incoming ID is reused) and `Server-Timing` (`app` total plus `circl` upstream time).
  - `config.py`: `Settings` (pydantic-settings) reads `ALLOWED_ORIGINS`, `ALLOWED_ORIGIN_REGEX`, `LOG_LEVEL` and `CIRCL_BASE_URL`.
- **Routes:**
  - `GET /api/vendors/{vendor}` returns `{vendor_label, data, stale, fetched_at}`. `data` has the shape `{ [productName]: [{id, summary, score, cvss_version, published}] }`, products with the most CVEs first.
  - `GET /api/vendors?q=` returns `{query, vendors}` for autocomplete. `GET /api/health` reports each dependency; `GET /api/meta` returns feature flags (all off until later phases).
  - Vendor names and queries are validated (1–64 characters: letters, digits, spaces and `.,&'+()-`); anything else is a 422.
  - **Deprecated:** `GET /api/vendor/{vendor_name}` keeps the old `{status, vendor_label, data}` shape for frontends deployed before the switch to `/api/vendors/{vendor}`. Unknown vendors and vendors without CVEs get `status: "fallback"` with a single `"System Error"` product. Remove it once the current frontend has been in production for a while (roadmap PR 8).
- **Lookup flow:** CIRCL (`https://vulnerability.circl.lu/api`, spec at `/api/swagger.json`) has no vendor-wide CVE search, so `VendorService`:
  1. resolves the input to a CIRCL vendor id through `/browse/vendors?q=` (`choose_vendor` prefers the most recently updated close match, because CIRCL keeps stale duplicates such as `palo alto` next to `palo alto networks`);
  2. lists the vendor's 15 most recently updated products (`/browse/vendors/{vendor}/products`);
  3. fetches `/vulnerability/search/{vendor}/{product}` for each product concurrently. A failed product is skipped; if all fail, it's an `UpstreamError`.
  - Records are in CVE JSON 5 format. The score comes from `containers.cna.metrics` (or the `adp` metrics) and prefers CVSS 3.1. `belongs_to_vendor` drops other vendors' CVEs, which the search sometimes returns. Display names come from `cna.affected[].product`.
  - Results are cached for 10 minutes. After that, if CIRCL fails, the last good copy is served with `stale: true`; with nothing cached it's a 502. Vendor suggestions get the same fallback (without a flag). There's no sample data any more.
- **Frontend:** a single-screen master-detail dashboard. The top bar has vendor search plus presets, then a vendor summary, then the product list (left) and the selected product's CVE table (right).
  - `src/api/index.ts` holds `fetchVendor` (an `openapi-fetch` client for `GET /api/vendors/{vendor}`) and re-exports the response types from the generated `src/api/schema.d.ts`; never edit that file or `openapi.json` by hand. `fetchVendor` returns `null` for a 404 and throws a readable error otherwise. The base URL comes from `VITE_API_URL` and defaults to `http://localhost:8000`.
  - The vendor summary badge shows "Live" or, when the backend served its last good copy, "Cached" (`stale: true`).
  - `src/severity.ts` defines the CVSS bands and their colors.
  - `src/App.tsx` owns the state and layout; `src/components/` holds the presentational pieces.
  - Request state is keyed by `vendor#attempt` rather than set synchronously inside effects, because eslint-plugin-react-hooks v7 flags `set-state-in-effect`.
  - Design tokens (surfaces, text, accent, severity colors) live in `@theme` in `src/index.css`. Use them as utilities (`bg-surface`, `border-line`, `text-muted`) instead of raw hex values.
  - The style is deliberately restrained: no glows, gradients, animations or uppercase tracking.
- **CORS:** `http://localhost:5173` is always allowed, and `GET` and `POST` are allowed methods (POST is for the AI explainer in roadmap PR 6). `X-Request-ID` is exposed to the browser. Deployed frontends go in the `ALLOWED_ORIGINS` env var (comma-separated exact origins, lowercased and trailing slashes stripped). `ALLOWED_ORIGIN_REGEX` can match Vercel preview URLs; anchor it on the team slug, because anyone can claim a `spectra-*.vercel.app` name. An invalid regex makes the app refuse to start with a clear error.
- `cve-backend/venv/`, `.venv/`, `__pycache__/`, `server.*.log` and the tool caches (`.coverage`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`) are gitignored local files, not source.

## Deployment (Vercel)
Two Vercel projects from this repo:
- **Backend:** Root Directory `cve-backend`. Vercel detects the FastAPI `app` in `main.py` and installs `requirements.txt`. Env vars: `ALLOWED_ORIGINS` (the frontend URL), optional `ALLOWED_ORIGIN_REGEX`, `LOG_LEVEL` and `CIRCL_BASE_URL`.
  - `cve-backend/.vercelignore` keeps `pyproject.toml`, `requirements-dev.txt` and `tests/` out of the deployment. Vercel's Python builder prefers `pyproject.toml` over `requirements.txt` when it sees both, and ours has no dependencies, so the deploy would install nothing. If runtime dependencies ever move into `pyproject.toml`, remove that line.
- **Frontend:** Root Directory `cve-frontend`, Vite preset. Env var: `VITE_API_URL` = backend URL (trailing slashes are stripped). It's baked in at build time, so redeploy after changing it. Only `npm run dev` falls back to `http://localhost:8000`; a production build without it shows a "no API URL" error.
- The root `.env.example` documents every env var and which project it belongs to.
- `requirements.txt` is pinned to the versions tested locally. Bump them deliberately.
- The 10-minute cache is per serverless instance, so cold instances refetch from CIRCL (and have no stale copy to fall back on).
