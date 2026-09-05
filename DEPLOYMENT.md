# Deploying to Vercel

This fork adds a [`vercel.json`](vercel.json) so the app builds and deploys on
Vercel out of the box. This document explains exactly what you get, and what
you don't.

## What works immediately

Vercel runs `npm install && npm run build` (`vite build`) and serves the
static `dist/` output. That gets you:

- The full 3D globe UI, styling, and voice-annotation/whiteboard tooling.
- The keyless Esri World Imagery basemap and OSM fallback (no API keys
  required — see the project [README Quick Start](README.md#-quick-start)).
- Photorealistic 3D Tiles / Google Maps place search / Cesium ion terrain,
  **if** you set `GOOGLE_MAPS_API_KEY` and/or `CESIUM_ION_TOKEN` as Vercel
  Environment Variables (Project Settings → Environment Variables). Both are
  read at **build time** via Vite's `loadEnv` and inlined into the client
  bundle (they are client-exposed by design — restrict them per
  [SECURITY.md](SECURITY.md)).
- Client-only, keyless features that call third-party APIs directly from the
  browser (no proxy involved).

## What does NOT work yet on a plain Vercel static deploy

Upstream, most of the live-data layers — flights (OpenSky), military
tracking and traces (adsb.lol), live vessels (AISStream), satellites
(CelesTrak), traffic cameras (CCTV/TfL/Caltrans/Austin), bike-share (GBFS),
active fires (NASA FIRMS), rocket launches (Launch Library 2), traffic flow
(TomTom), terrain height lookups, regional news/weather briefings, Overpass
road/military-installation context, Google Places/nearby search, and the
OpenAI Realtime voice-agent token minting — are all implemented as **Vite
dev-server middleware** registered via `configureServer()` in
[`vite.config.js`](vite.config.js) (see the `/api/*` routes there and in
[`DATA_SOURCES.md`](DATA_SOURCES.md)).

That middleware only runs under `vite dev` / `vite preview`. It is **not**
part of the production `vite build` output, and Vercel's static hosting
does not execute it. On a plain static deploy of this fork, those `/api/*`
calls will 404 and the corresponding layers will silently stay empty/off —
the globe still loads, but flights/ships/satellites/cameras/voice-AI and the
rest of the live intel picture won't populate.

### Making the live-data layers work on Vercel

Each `/api/*` route in `vite.config.js` would need to be ported to a
[Vercel Serverless (Node.js) Function](https://vercel.com/docs/functions)
under an `/api` directory at the repo root (e.g. `api/opensky.js`,
`api/celestrak.js`, ...), since Vercel auto-deploys any file under `/api` as
its own function with a compatible `(req, res)` signature. A few routes need
extra care beyond a mechanical port:

- **`/api/ais-live`** streams over a WebSocket connection to AISStream.io and
  keeps state across requests — plain Vercel Serverless Functions are
  request/response and don't hold a persistent outbound connection between
  invocations. This one needs a different approach (e.g. a small always-on
  relay service, or Vercel's WebSocket-capable primitives) rather than a
  drop-in function.
- **`/api/opensky`** and **`/api/openai/realtime/token`** handle OAuth /
  API-key exchanges server-side — keep the same secret-handling discipline
  (env vars, never returning raw upstream credentials to the client) when
  porting them.
- Routes with on-disk caching (FIRMS, radio stations, terrain heights,
  TomTom, regional brief, weather effects, military installations) use a
  local `.gev-cache/` directory for TTL'd caching; Vercel Functions are
  stateless/ephemeral between cold starts, so that caching would need to
  move to a KV store (e.g. Vercel KV / Upstash Redis) or be dropped in favor
  of shorter in-memory caching per invocation.

This is intentionally left as follow-up work rather than done here: it's a
large refactor of a ~7,800-line config file touching several
security-sensitive code paths (OAuth, API-key handling), and deserves its
own careful review rather than a rushed mechanical port.

## Node.js version

`package.json` pins `engines.node` to `>=24.14.0 <25 || >=26 <27`. Vercel
now supports Node.js 24 LTS for Builds and Functions — set your project's
Node.js Version to **24.x** under Project Settings → General (Vercel
defaults new projects to a different LTS, so this needs to be set
explicitly).

## One-click deploy

Once the above is acceptable for your use case (static app, BYO keys for the
two client-side providers, live-data layers off until ported), use:

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/tahirmahm/god-s-eye-view)
