# FitnessCaptain

A fitness and exercise logging app: log workouts fast, build reusable
routines, and see progress over time. Part of the Forever Apps portfolio.

Single-file React PWA (`index.html`, React via CDN + Babel standalone, no
build step) served from GitHub Pages. Data syncs through the
`fitnesscaptain-backend` FastAPI service (Railway) to Supabase; Supabase is
used browser-side for AUTH ONLY.

## Development

- Edit `index.html`. There is no build step.
- Compile gate before every deploy: `npm run check` (runs the same Babel +
  react preset the browser uses, in Node, so a JSX typo never white-screens
  production).
- Bump `APP_VERSION` and `BUILD` in index.html AND `VERSION` in `sw.js` on
  every deployed change - the update banner and the service worker cache both
  key off them.

## Configuration

Three public-safe values at the top of the script block in `index.html`:
`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `API_BASE`. Until they are set the app
boots to the sign-in screen with a "not configured" notice. Real secrets live
only on the backend.

## Deploy

Push to `main`; GitHub Pages serves the repo root.
