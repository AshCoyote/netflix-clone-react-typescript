# Notes for working in this repo

## What the app is
A pure client-side Vite + React + Redux Toolkit SPA (MUI, react-router). There is no
backend in this repository — all movie/TV data comes from The Movie DB (TMDB) v3 over
the browser. `vite.config.ts` has no server config; the dev server is started with
`--host 0.0.0.0 --port 3000`.

## Running it here
- `docker compose -f docker-compose.base44.yml up -d --build`
- The `web` service runs `npm ci` on startup against the bind-mounted checkout, then
  `vite` in dev mode on host port 3000 (the preview port). `node_modules` lives in a
  named Docker volume, so a dependency change only needs a service restart.
- Vite caches pre-bundled deps in `node_modules/.vite`; if a dependency change seems
  ignored, restart the service.
- The repo's own `Dockerfile` is a production nginx build (and references a `yarn.lock`
  that is gitignored). Do not use it for local work — it bakes the bundle and hides edits.

## Configuration
- `VITE_APP_API_ENDPOINT_URL` and `VITE_APP_TMDB_V3_API_KEY` are read via
  `import.meta.env` in `src/constant/index.ts`. Vite's `loadEnv` also forwards
  `VITE_*` variables from the process environment, which is how compose supplies them.
- Placeholders live in `.env.base44-defaults` (committed, listed first in compose).
  Real values are delivered by the platform at `/run/base44/app.env`, listed last so
  they win. There is intentionally no repo-root `.env`.
- `VITE_APP_TMDB_V3_API_KEY` is a real credential: create one at
  https://www.themoviedb.org/settings/api (the "API Key (v3 auth)" value). With a
  missing/invalid key TMDB returns 401 and the pages render empty (no error UI), so a
  blank browse page usually means a bad key, not a broken build.
- Env values are read by the Vite dev server process at startup: after a key change the
  `web` service must be recreated before the browser bundle sees the new value.

## How to verify
- `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3000/` → 200.
- `curl -s http://127.0.0.1:3000/src/constant/index.ts` should return unhashed source
  (confirms dev server, not a prebuilt bundle).
- In the browser, `/browse` should show a hero plus ~20+ rows of posters (network calls
  to `api.themoviedb.org` returning 200, no 401s).

## Known non-issues
- Some TMDB hero trailers disallow embedding (video.js logs `CODE:1150 "Playback on other
  Websites has been disabled by the video owner"`). That is a per-trailer YouTube restriction,
  not a setup problem; `HeroSection.tsx` hides the player on error and shows the TMDB backdrop.
- `/watch` is a demo player: it streams a public HLS test video (`test-streams.mux.dev`,
  Big Buck Bunny), not the movie picked on the home page. The previously hardcoded Bitmovin
  sample URL now returns 403, so "The media could not be loaded" there means the stream URL
  is dead again — swap in another HLS URL.
- Browsers block unmuted autoplay, so `/watch` can sit at 0:00 until the user presses play.
- Poster images are served over `http://image.tmdb.org`; they load fine through the
  preview proxy.
