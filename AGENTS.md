# AGENTS.md

## What this project is
A Cloudflare Worker (`index.js`, deployed via `wrangler.toml`) that serves an HTTP surface for a
VLESS proxy service: landing page `/`, a per-user QR/config panel at `/<uuid>`, QR helper APIs
(`POST /api/qr-generate`, `POST /api/qr-validate`), `robots.txt`, `security.txt`, and a custom `/404`.

- `worker.js` and `_worker.js` are **not** the entry point — `wrangler.toml` sets `main = "index.js"`.
  `_worker.js` is the obfuscated build of `index.js` produced by `.github/workflows/obfs.yml`.
- Edit `index.js`; the dev server reloads it.

## Running it locally (Base44 sandbox)
```
docker compose -f docker-compose.base44.yml up -d --build
```
- Single service `worker` on `node:22-slim`, source bind-mounted at `/app`, deps installed into the
  `node_modules` named volume on startup (there is no committed lockfile — `.gitignore` excludes
  `*-lock.*`, so `npm install` runs each boot).
- Runs `npx wrangler dev` (workerd). Entry point is host port **3000** (`3000:3000`).
- Wrangler state/SQLite lives in `/tmp/wrangler-state` inside the container via `--persist-to`, so no
  `.wrangler/` directory is written into the repo. It is not persistent across container recreation.

## Sandbox-only config
- `wrangler.base44.toml` is the config the compose service uses (`--config wrangler.base44.toml`).
  It mirrors `wrangler.toml` and adds a **local D1 binding** `DB`. Without `env.DB` the worker still
  serves the landing page, but `getUserData()` returns `null` and `/<uuid>` responds `403 User not found`.
  With the binding, `ensureTablesExist()` creates the tables on the first request and seeds the
  configured `UUID` as a test user, so the user panel renders.
- `UUID` / `PROXYIP` are Worker vars from the config, not secrets. No external credentials are needed to
  boot. Optional env vars read by the code: `SCAMALYTICS_*`, `SOCKS5`, `SOCKS5_RELAY`, `PROXYIPS`.

## Verification
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:3000/` → 200 (landing page).
- `curl -s http://localhost:3000/<UUID>` → 200 with `<title>🚀 Intelligent QR Manager ...` (requires D1).
- `curl -s -X POST http://localhost:3000/api/qr-validate -H 'content-type: application/json' -d '{"text":"vless://<uuid>@host:443?security=tls&type=ws#n"}'`
  → JSON `validation.valid: true`.

## Known quirks
- The landing page links to `/admin`, but `index.js` has no `/admin` route, so that URL returns a plain
  `OK` body. (Admin panel code exists in the unrelated, unused `worker.js`.)
- Wrangler resolves to 3.114.x from `package.json` (`^3.1.0`); its bundled workerd only supports
  compatibility dates up to 2025-07-18, so the requested `2025-11-09` falls back with a warning. Harmless
  here; bump to `wrangler@4` to silence it.
- There is no WebSocket/VLESS data-plane handler in `index.js`; only the HTTP surface runs locally.
- `POST /api/qr-generate` fails server-side in a Worker (`document is not defined` → Google Charts fallback →
  relative self-fetch `Invalid URL: /api/qr-generate`). Pre-existing app bug: `QRIntelligence` mixes
  browser-only canvas code with the Worker route. The user panel generates QRs client-side, so the UI does
  not depend on this endpoint; calling it just logs those warnings.
- No test suite in the repo.
