# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview
Spectra is a CVE dashboard. Pick a vendor (e.g. Cisco, VMware, Palo Alto) and it shows summary stats for the vendor, a list of its products, and the selected product's CVEs in a sortable table with a CVSS severity breakdown bar. It has two independent parts:

- `cve-backend/`: a single-file FastAPI app (`main.py`). Dependencies are in `requirements.txt` (`fastapi`, `uvicorn`, `requests`), installed in `cve-backend/.venv`. Dev tools (ruff, mypy, pytest, pytest-cov) are in `requirements-dev.txt`. Their config is in `pyproject.toml`, which holds tool config only (see Deployment). On this machine bare `python` can resolve to the Microsoft Store stub, so use `.venv/Scripts/python.exe`.
- `cve-frontend/`: React 19, TypeScript, Vite 8 and Tailwind v4 (via `@tailwindcss/postcss`; there's no `tailwind.config.js`). Fonts are self-hosted through `@fontsource` (Inter, JetBrains Mono). There's no chart library. Prettier formats the code (`printWidth` 120).

Tests never touch the network:
- Backend tests live in `cve-backend/tests/` (pytest) and use hand-written CVE records.
- Frontend tests sit next to the source as `*.test.ts(x)` (Vitest, Testing Library, jsdom). `src/test/setup.ts` unmounts rendered components after each test.

## Commands
Backend (from `cve-backend/`):
- `.venv/Scripts/python.exe main.py`: runs uvicorn on 0.0.0.0:8000. You can also use `.venv/Scripts/python.exe -m uvicorn main:app --reload --port 8000`.
- `.venv/Scripts/python.exe -m pip install -r requirements-dev.txt`: installs the runtime deps plus the dev tools
- `.venv/Scripts/python.exe -m pytest`: runs the tests. Add `--cov` for coverage. Run a single test with `-m pytest tests/test_parsing.py -k display_name`.
- `.venv/Scripts/python.exe -m ruff check .` and `-m ruff format .`: lint and format
- `.venv/Scripts/python.exe -m mypy`: type-checks `main.py` and `tests/` (paths are set in `pyproject.toml`). The config is lenient for now.

Frontend (from `cve-frontend/`):
- `npm install`
- `npm run dev`: Vite dev server on http://localhost:5173
- `npm run build`: `tsc -b && vite build`, so it also type-checks (test files included)
- `npm run lint`: ESLint flat config (typescript-eslint, react-hooks, react-refresh)
- `npm test`: runs Vitest once. `npm run test:watch` keeps it running; `npx vitest run src/severity.test.ts` runs one file.
- `npm run format` and `npm run format:check`: Prettier

CI (`.github/workflows/ci.yml`) runs on pull requests and pushes to `main`:
- backend on Python 3.12, Vercel's default: ruff check, ruff format --check, mypy, pytest --cov
- frontend on Node 24: npm ci, lint, format:check, `tsc -b`, vitest run, build

Dependabot (`.github/dependabot.yml`) opens weekly update PRs for pip, npm and GitHub Actions.

## Architecture
- **Single endpoint:** `GET /api/vendor/{vendor_name}` returns `{status, vendor_label, data}`.
  - `data` has the shape `{ [productName]: [{id, summary, score, cvss_version?, published?}] }`. Mock data has no `cvss_version` or `published`.
  - `status` is `success`, `fallback` or `offline_mode`, and the UI shows it as a "Live" or "Sample data" badge.
  - An unknown vendor comes back as a single `"System Error"` product, which the UI treats as "not found".
- **Live vs. mock:** `USE_LIVE_API` in `main.py` switches modes. Live mode uses the CIRCL Vulnerability-Lookup API (`https://vulnerability.circl.lu/api`, spec at `/api/swagger.json`). It has no vendor-wide CVE search, so `fetch_live_products`:
  1. `resolve_vendor` maps the input to a CIRCL vendor id through `/browse/vendors?q=`. It prefers the most recently updated close match, because CIRCL keeps stale duplicates such as `palo alto` next to `palo alto networks`.
  2. It lists the vendor's 15 most recently updated products (`/browse/vendors/{vendor}/products`).
  3. It fetches `/vulnerability/search/{vendor}/{product}` for each product in parallel.
  - Records are in CVE JSON 5 format. The score comes from `containers.cna.metrics` (or the `adp` metrics) and prefers CVSS 3.1. `belongs_to_vendor` drops other vendors' CVEs, which the search sometimes returns. Display names come from `cna.affected[].product`.
  - Any vendor works live, with no per-vendor code. Results are cached in memory for 10 minutes.
  - **On any exception it quietly returns `MOCK_DATA`** with status `fallback`. Unknown vendors get a single "System Error" product.
- **Frontend:** a single-screen master-detail dashboard. The top bar has vendor search plus presets, then a vendor summary, then the product list (left) and the selected product's CVE table (right).
  - `src/api.ts` holds the response types and `fetchVendor`. The base URL comes from `VITE_API_URL` and defaults to `http://localhost:8000`.
  - `src/severity.ts` defines the CVSS bands and their colors.
  - `src/App.tsx` owns the state and layout; `src/components/` holds the presentational pieces.
  - Request state is keyed by `vendor#attempt` rather than set synchronously inside effects, because eslint-plugin-react-hooks v7 flags `set-state-in-effect`.
  - Design tokens (surfaces, text, accent, severity colors) live in `@theme` in `src/index.css`. Use them as utilities (`bg-surface`, `border-line`, `text-muted`) instead of raw hex values.
  - The style is deliberately restrained: no glows, gradients, animations or uppercase tracking.
- **CORS:** `http://localhost:5173` is always allowed. Deployed frontends go in the `ALLOWED_ORIGINS` env var (comma-separated exact origins, lowercased and trailing slashes stripped). `ALLOWED_ORIGIN_REGEX` can match Vercel preview URLs; anchor it on the team slug, because anyone can claim a `spectra-*.vercel.app` name. An invalid regex makes the app refuse to start with a clear error.
- `cve-backend/venv/`, `.venv/`, `__pycache__/`, `server.*.log` and the tool caches (`.coverage`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`) are gitignored local files, not source.

## Deployment (Vercel)
Two Vercel projects from this repo:
- **Backend:** Root Directory `cve-backend`. Vercel detects the FastAPI `app` in `main.py` and installs `requirements.txt`. Env vars: `ALLOWED_ORIGINS` (the frontend URL), optional `ALLOWED_ORIGIN_REGEX`.
  - `cve-backend/.vercelignore` keeps `pyproject.toml`, `requirements-dev.txt` and `tests/` out of the deployment. Vercel's Python builder prefers `pyproject.toml` over `requirements.txt` when it sees both, and ours has no dependencies, so the deploy would install nothing. If runtime dependencies ever move into `pyproject.toml`, remove that line.
- **Frontend:** Root Directory `cve-frontend`, Vite preset. Env var: `VITE_API_URL` = backend URL (trailing slashes are stripped). It's baked in at build time, so redeploy after changing it. Only `npm run dev` falls back to `http://localhost:8000`; a production build without it shows a "no API URL" error.
- The root `.env.example` documents every env var and which project it belongs to.
- `requirements.txt` is pinned to the versions tested locally. Bump them deliberately.
- The 10-minute cache is per serverless instance, so cold instances refetch from CIRCL.
