# moviemode-service

**Inner Animal Media product worker** — MovieMode encode lane, studio UI, and the scroll-driven **3D globe landing** (`Code = Communication`).

Deploy target: Cloudflare Worker `moviemode-service` · Git: `SamPrimeaux/moviemode-service` · Invoked from **meauxcloud** via service binding.

## Layout

```
moviemode-service/
├── public/                 # Globe landing (served at /)
│   ├── index.html          # "Code = Communication" hero
│   ├── globe.js            # Procedural Three.js globe (window.GlobeScene)
│   ├── scroll.js           # Scroll choreography → globe + cards
│   └── charts.js           # SVG mini-charts in cards
├── studio/                 # MovieMode React + Remotion editor (served at /studio/)
├── worker/src/             # Cloudflare Worker — API + webhooks
│   ├── index.js
│   ├── api/                # moviemode, cloudconvert, stream webhooks
│   └── core/               # conversions, templates, stream, cloudconvert
├── migrations/             # D1 schema (synced from inneranimalmedia)
├── docs/MOVIEMODE.md
└── scripts/sync-from-iam.sh
```

## Routes

| URL | What |
|-----|------|
| `/` | Globe landing — scroll-driven frosted-earth scene |
| `/studio/` | MovieMode editor (after `npm run build:studio`) |
| `/api/moviemode/*` | Projects, media, conversions, templates, timelines |
| `/api/cloudconvert/*` | Presets, jobs, operations |
| `/api/stream/*` | Live inputs, Stream library |
| `/api/webhooks/cloudconvert` | CloudConvert job lifecycle |
| `/api/webhooks/stream/vod` · `/live` | Stream VOD + live |
| `/health` | Liveness |

## Globe scene

The landing is **vanilla Three.js** (no `.glb`):

- `globe.js` — procedural shader earth + atmosphere; exposes `window.GlobeScene.setProgress/tint/motion`
- `scroll.js` — maps scroll → globe rotation, card choreography, background wash
- `charts.js` — monochrome SVG sparklines in feature cards

Open `https://<moviemode-service>.workers.dev/` after deploy, or route a custom domain in wrangler.

## Secrets (Cloudflare dashboard)

| Secret | Purpose |
|--------|---------|
| `CLOUDCONVERT_API_KEY` | Encode jobs |
| `CLOUDCONVERT_WEBHOOK_SECRET` | Webhook HMAC |
| `CLOUDFLARE_API_TOKEN` | Stream / account API |
| `MESHYAI_API_KEY` | 3D mesh generation (CAD lane) |
| `OPEN_AI_KEY` | Optional Whisper / tooling |
| `OIDC_PRIVATE_KEY` | Service-to-service auth |
| `IAM_SERVICE_KEY` | Header `X-IAM-Service-Key` from meauxcloud |

Bindings: **D1** `inneranimalmedia-business`, **R2** `inneranimalmedia` + `artifacts`, **AI**, **KV** `IAM_SESSION` (session cookies from main site).

## meauxcloud integration

On **meauxcloud** worker, add service binding:

```toml
[[services]]
binding = "MOVIEMODE"
service = "moviemode-service"
environment = "production"
```

Forward media routes:

```js
if (url.pathname.startsWith('/api/moviemode') || url.pathname.startsWith('/api/cloudconvert')) {
  return env.MOVIEMODE.fetch(request);
}
```

Or serve landing at `meauxcloud.com` root via `env.MOVIEMODE.fetch(new Request('https://moviemode/'))`.

## Develop

```bash
# Refresh from monorepo
IAM_ROOT=../inneranimalmedia npm run sync

npm install
npm run check
npm run dev          # wrangler dev — globe at http://localhost:8787/

cd studio && npm install && npm run build
npm run build        # copies studio → public/studio
npx wrangler deploy
```

## Relationship to inneranimalmedia

| Concern | Where |
|---------|--------|
| Production dashboard route | `inneranimalmedia.com/dashboard/moviemode` |
| Canonical API (today) | Main worker `inneranimalmedia` |
| Product iteration | **This repo** — sync script keeps parity |
| PTY Remotion render | Main worker / terminal (not this worker) |
| Vectorize / pgvector embed | Main worker `POST /api/moviemode/embed` |

Run `npm run sync` after MovieMode changes in `inneranimalmedia`, then commit here.
