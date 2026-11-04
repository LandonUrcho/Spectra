# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview
Spectra is a CVE dashboard. Pick a vendor (e.g. Cisco, VMware, Palo Alto) and it shows the vendor's products, each product's CVEs (paginated), and a donut chart of CVSS severity. It has two independent parts:

- `cve-backend/`: a single-file FastAPI app (`main.py`). Dependencies are in `requirements.txt` (`fastapi`, `uvicorn`, `requests`), installed in `cve-backend/.venv`. On this machine bare `python` can resolve to the Microsoft Store stub, so use `.venv/Scripts/python.exe`.
- `cve-frontend/`: React 19, TypeScript, Vite 8 and Tailwind v4 (via `@tailwindcss/postcss`; `tailwind.config.js` is a v3 leftover and isn't used). Fonts are self-hosted through `@fontsource` (Inter, JetBrains Mono). There's no chart library.

There are no tests in either part.

## Commands
Backend (from `cve-backend/`):
- `.venv/Scripts/python.exe main.py`: runs uvicorn on 0.0.0.0:8000. You can also use `.venv/Scripts/python.exe -m uvicorn main:app --reload --port 8000`.

Frontend (from `cve-frontend/`):
- `npm install`
- `npm run dev`: Vite dev server on http://localhost:5173
- `npm run build`: `tsc -b && vite build`, so it also type-checks
- `npm run lint`: ESLint flat config (typescript-eslint, react-hooks, react-refresh)

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
- `cve-backend/venv/`, `.venv/`, `__pycache__/` and `server.*.log` are gitignored local files, not source.

## Deployment (Vercel)
Two Vercel projects from this repo:
- **Backend:** Root Directory `cve-backend`. Vercel detects the FastAPI `app` in `main.py` and installs `requirements.txt`. Env vars: `ALLOWED_ORIGINS` (the frontend URL), optional `ALLOWED_ORIGIN_REGEX`.
- **Frontend:** Root Directory `cve-frontend`, Vite preset. Env var: `VITE_API_URL` = backend URL (trailing slashes are stripped). It's baked in at build time, so redeploy after changing it. Only `npm run dev` falls back to `http://localhost:8000`; a production build without it shows a "no API URL" error.
- `requirements.txt` is pinned to the versions tested locally. Bump them deliberately.
- The 10-minute cache is per serverless instance, so cold instances refetch from CIRCL.
