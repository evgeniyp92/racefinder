# CLAUDE.md — Racefinder

## Current TODOs
- [ ] Fix the dashboard password being leaked in traefik/values.yaml
- [ ] Create a landing page for boysoft.dev as a smoke test and disambiguation
- [ ] Create a racefinder namespace
- [ ] Deploy and secure PostgreSQL
- [ ] Add PostGIS extension
- [ ] Set up routing (Gateway API HTTPRoutes via Traefik)

---

## What is Racefinder?

Racefinder is a cross-platform sim racing schedule aggregator. It pulls race
schedules from multiple sim racing platforms (iRacing, Le Mans Ultimate, Low
Fuel Motorsport) and presents them through a **map-based interface** rather than
traditional tables/lists. Users visually navigate to tracks on a world map and
see what races are available across platforms — tap on Spa and see GT3 on
iRacing in 45 min, Hypercar on LMU tomorrow, LFM GT3 split in 2 hours.

The map is the killer feature. Every sim racing platform presents races as
sorted tables — functional but boring. Racefinder's differentiator is spatial
browsing: a world map where circuits are pins, and filtering by
class/platform/time highlights only relevant tracks.

---

## Tech Stack

| Layer         | Technology                                |
|---------------|-------------------------------------------|
| Monorepo      | Turborepo                                 |
| Frontend      | Next.js (App Router) + React + TypeScript |
| Map renderer  | MapLibre GL JS + deck.gl                  |
| Basemap tiles | Hosted free tier (MapTiler / Protomaps)   |
| API           | tRPC (via `@trpc/next`)                   |
| ORM           | Prisma                                    |
| Database      | PostgreSQL + PostGIS                      |
| Auth          | Discord OAuth (via `arctic` or `lucia`)   |
| Testing       | Vitest                                    |
| Deployment    | Kubernetes (K3s) behind Traefik ingress   |
| Runtime       | Node (with Bun for package installs)      |

### Runtime note

The project uses Node as the primary runtime. Bun is used for `bun install` (
faster package installs). Dockerfiles should use Node base images unless a
specific service has been explicitly migrated to Bun. For production, Next.js
should use `standalone` output mode to minimize the deployment footprint (~30MB
Node server).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Frontend (Next.js)                    │
│    App Router · deck.gl + MapLibre GL · ISR SEO pages   │
│    tRPC API routes · Auth (Discord OAuth) · Caching     │
│              Admin: /admin route (role-gated)           │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│              PostgreSQL + PostGIS                        │
│                                                         │
│  Canonical: Track, Car, ClassDefinition                 │
│  Platform-specific: TrackAlias, PlatformClass, etc.     │
│  Events: FetcherRawEvent → RaceEvent (normalized)       │
│  Users: Discord auth, favorites, platform prefs         │
└──────────────────────▲──────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼───┐ ┌─────▼────┐ ┌────▼─────┐
   │ iRacing  │ │   LMU    │ │   LFM    │
   │ Fetcher  │ │ Fetcher  │ │ Fetcher  │
   └──────────┘ └──────────┘ └──────────┘
          │            │            │
          └────────────┼────────────┘
                       │
              ┌────────▼────────┐
              │   Normalizer    │
              │ raw → canonical │
              └─────────────────┘
```

### Data flow

1. **Fetcher services** (one per platform) run as K8s CronJobs on configurable
   schedules. Each fetcher pulls raw schedule data from its platform's API and
   writes it to the `fetcher_raw_events` staging table as JSON.
2. **Normalizer service** reads raw events, maps them to canonical
   tracks/cars/classes using alias tables, and writes clean `RaceEvent` records.
   This decoupling means improved alias mappings can be applied retroactively by
   reprocessing raw data.
3. **API layer** is tRPC, integrated via `@trpc/next`. tRPC routers can run as
   Next.js API routes or as a standalone server — choose based on deployment
   needs. Since race schedules are relatively static, use aggressive caching (
   HTTP cache headers, optional Redis layer).
4. **Frontend** is Next.js App Router. The map page (`/`) is a client-rendered
   route (MapLibre GL is WebGL, cannot be SSR'd). SEO pages (
   `/races/:platform/:track`) use ISR to serve cached static HTML with
   structured data. See the SEO Pages section below.

### Fetcher scheduling guidelines

- **iRacing**: Seasons are 12 weeks with published track rotations. Fetch the
  full season schedule once, then poll for session times/participation closer to
  race time.
- **LFM**: Schedule rotates weekly (track changes on Mondays typically). Daily
  fetch plus more frequent checks for session status.
- **LMU**: Least structured — may require scraping if no formal API. Design the
  fetcher interface to be pluggable for adding future platforms.

Each fetcher's cron expression is stored in the `platforms` table, so frequency
can be tuned per-platform without redeployment.

### Admin interface

The admin UI is a protected `/admin` route in the Next.js app, gated by a role
check. Eugene is the only admin initially. Admin features include: curating
track alias mappings, managing car classifications, monitoring fetcher health.

---

## Database Schema

The Prisma schema is the source of truth. Key design decisions:

### Track normalization (critical for the map)

- `Track` — canonical entry with lat/lng coordinates, country. This is what
  becomes a map pin.
- `TrackConfiguration` — layouts like "Daytona Oval" vs "Daytona Road Course".
  Races happen on configurations, not tracks.
- `TrackAlias` — maps platform-specific names to canonical tracks (e.g.,
  iRacing's "Autodromo Internazionale Enzo e Dino Ferrari" → canonical "Imola").

Track location data is the most valuable static dataset. Invest in accurate
lat/lng coordinates.

### Car class hierarchy (three-tier)

- `ClassDefinition` — abstract concept: "GT3", "Hypercar", "LMDh". Has a
  `colorHex` field for map pin / badge coloring and a `sortOrder` for UI
  ordering.
- `PlatformClass` — a class as it exists on a specific platform ("Hypercar on
  iRacing" has different cars than "Hypercar on LMU").
- `PlatformClassCar` — join table: which specific cars are in which class on
  which platform.

When a user filters by "Hypercar", query `ClassDefinition` and cascade down to
show the right cars per platform.

### Car availability

- `Car` — canonical car entry (e.g., "Porsche 963") with manufacturer, model,
  year.
- `CarPlatformAvailability` — which platforms a car is available on, with
  platform-specific metadata.

### Race events

- `FetcherRawEvent` — staging table. Fetchers dump raw JSON here. Has a `status`
  enum: `PENDING`, `PROCESSED`, `FAILED`, `SKIPPED`.
- `RaceEvent` — normalized event, references `PlatformClass` and
  `TrackConfiguration`. Has a `sourceType` enum (`PLATFORM_OFFICIAL`, `LEAGUE`,
  `COMMUNITY`) baked in from day one so leagues won't require a schema migration
  later. Includes a `rawData` JSON field preserving original fetcher data.

### Users

- Discord OAuth as primary auth. Store Discord user ID.
- Optional linked accounts: `iracingCustomerId`, `lfmProfileId`.
- Favorites via join tables: `UserFavoriteTrack`, `UserFavoriteClass`.
- `UserPlatformPreference` — toggle visibility of platforms.
- `timezone` field for displaying race times correctly.

### Series

- `Series` — groups race events into their parent series (e.g., "IMSA Endurance
  Series", "LFM GT3 Sprint"). A race event without series context is harder to
  make sense of on the frontend.

---

## Map Implementation

### Stack

- **MapLibre GL JS** — basemap rendering (FLOSS Mapbox GL fork)
- **deck.gl** — visualization layer on top of MapLibre. Handles track pins (
  `IconLayer`), clustering (`ClusterLayer`), interactions, and GPU-accelerated
  rendering. Use `MapboxOverlay` adapter for MapLibre integration.
- **Basemap tiles** — use a hosted free tier to start (MapTiler, Protomaps, or
  Stadia Maps). No self-hosted tile infrastructure for V1.

### Why this approach

Track locations are just a few hundred lat/lng points — not custom geometry that
needs a tile server. deck.gl renders them as an overlay on the basemap. This
avoids standing up Martin + PMTiles infrastructure in V1 while still giving you
GPU-accelerated clustering and smooth interactions.

### Future: self-hosted tiles

If/when you need custom basemap styling deeply integrated with the
Asphalt/Circuit Dawn palette, or want to serve your own geospatial data as
vector tiles, add **Martin** (Rust tile server from PostGIS) and/or **PMTiles
** (static basemap in a flat file). This is an additive change — MapLibre GL
swaps tile sources without affecting the deck.gl overlay layer.

### Map UX requirements

- **Clustering**: Circuits cluster heavily in Western Europe. Use marker
  clustering at low zoom levels that expand as users zoom in. At zoomed-out
  level, show region badges like "Europe — 47 races today".
- **Filter-first interaction**: Users should be able to say "show me GT3 races
  in the next 3 hours" and the map highlights only relevant tracks. The map is a
  visual filter, not just a display.
- **Sidebar/bottom sheet**: Filtered list alongside the map so users can switch
  between spatial and temporal browsing. Consider draggable floating panels over
  the map layer.
- **Color-coded pins**: Use `ClassDefinition.colorHex` to color map pins by
  class.

---

## SEO Pages (ISR)

Racefinder uses Next.js Incremental Static Regeneration to generate crawlable,
bookmarkable pages for high-intent search queries like "iRacing races at Spa
this week." These are standalone destinations, not thin doorway pages.

### Route structure

- `/` — the main map experience (client-rendered, MapLibre GL)
- `/races/:platform/:track` — full schedule page for a track on a platform (ISR)
- `/races/this-week/:track` — cross-platform schedule for a track this week (
  ISR)
- `/tracks/:track` — track info page with all platforms and upcoming races (ISR)

### ISR revalidation

Set `revalidate` based on data freshness needs. Race schedules are relatively
static — a revalidation period of 1 hour is likely sufficient. When fetchers
update the database, you can optionally call `res.revalidate()` via an on-demand
revalidation webhook for immediate freshness.

### SEO requirements

- Proper `<title>` tags: "iRacing GT3 Races at Spa-Francorchamps — This Week |
  Racefinder"
- Open Graph and Twitter Card meta tags for social sharing
- JSON-LD structured data using [Event schema](https://schema.org/Event) for
  each race
- Canonical URLs to avoid duplicate content across route variants
- Each page should include a prominent "Open in Racefinder" CTA that deep-links
  into the map with appropriate filters applied

### Implementation notes

- tRPC queries for these pages run server-side in React Server Components — the
  HTML ships fully rendered.
- The map page (`/`) should use `'use client'` at the page or layout level since
  MapLibre GL is entirely client-side.
- ISR pages are cached to disk after first render. The Pi serves cached HTML on
  subsequent requests — minimal CPU cost.

---

## Design System

### Color palette — unified light/dark

The palette uses **Circuit Dawn** (light) and **Asphalt** (dark) as a unified
pair. Both modes share Racing Orange as the primary accent.

#### Shared accents (both modes)

| Token           | Hex       | Role                                         |
|-----------------|-----------|----------------------------------------------|
| `primary`       | `#e8590c` | Racing Orange — brand color, primary actions |
| `primary-hover` | `#d14e08` | Hover state                                  |
| `primary-muted` | `#c2571d` | Desaturated for subtle use                   |
| `success`       | `#16a34a` | Green flag, available, success               |
| `warning`       | `#ca8a04` | Yellow flag, caution                         |
| `error`         | `#dc2626` | Red flag, destructive                        |

#### Circuit Dawn (light mode)

| Token            | Hex       | Role                            |
|------------------|-----------|---------------------------------|
| `page-bg`        | `#faf8f5` | Page background (warm white)    |
| `card-bg`        | `#f0ece6` | Cards, panels (sand)            |
| `elevated`       | `#e6e0d8` | Hover, pressed states (clay)    |
| `border`         | `#d4cdc3` | Borders, dividers (gravel)      |
| `border-subtle`  | `#e0dbd4` | Subtle borders                  |
| `secondary`      | `#1e3a5f` | Deep navy — headers, navigation |
| `text-primary`   | `#1a1a1a` | Headings, body text             |
| `text-secondary` | `#5c5c5c` | Metadata, timestamps            |
| `text-muted`     | `#9c9c9c` | Disabled, placeholders          |

#### Asphalt (dark mode)

| Token            | Hex       | Role                                           |
|------------------|-----------|------------------------------------------------|
| `page-bg`        | `#121210` | Page background (tarmac)                       |
| `card-bg`        | `#1c1c18` | Cards, panels (warm-tinted black)              |
| `elevated`       | `#272720` | Elevated surfaces (curb)                       |
| `border`         | `#38382e` | Borders, dividers                              |
| `border-subtle`  | `#2e2e26` | Subtle borders                                 |
| `secondary`      | `#c8b89a` | Warm stone — replaces navy (invisible on dark) |
| `text-primary`   | `#e8e8e0` | Headings, body text                            |
| `text-secondary` | `#8a8a7a` | Metadata, timestamps                           |
| `text-muted`     | `#5a5a4e` | Disabled, placeholders                         |

Dark mode surfaces use **warm/olive-tinted blacks** (not blue-gray). This is
intentional — it evokes tarmac under floodlights. Do not replace with cold
neutral grays.

### Typography

- **Primary font**: Satoshi (via Fontshare:
  `https://api.fontshare.com/v2/css?f[]=satoshi@400,500,700,900&display=swap`)
- **Monospace** (for data values): system monospace — use for countdown timers,
  lap times, registration counts, hex codes. Everything else uses Satoshi.
- Fallback stack: `'Satoshi', system-ui, sans-serif`

### Design principles

- Avoid the generic motorsport trap: don't overuse red+black together, don't use
  too many colors at once.
- Let structure and motion carry motorsport energy, not just color. Subtle
  gradients hinting at metallic/carbon surfaces, snappy micro-interactions.
- The app should feel approachable to a casual sim racer who just bought ACC on
  sale, while still standing out from generic SaaS apps.

---

## Project Structure (Turborepo)

```
racefinder/
├── apps/
│   └── web/                 # Next.js app (App Router, tRPC client + API routes)
├── packages/
│   ├── db/                  # Prisma schema, client, migrations
│   ├── api/                 # tRPC router definitions (imported by web + services)
│   ├── shared/              # Shared types, constants, utilities
│   └── ui/                  # Shared UI components (if needed)
├── services/
│   ├── fetcher-iracing/     # iRacing fetcher (K8s CronJob)
│   ├── fetcher-lmu/         # LMU fetcher (K8s CronJob)
│   ├── fetcher-lfm/         # LFM fetcher (K8s CronJob)
│   └── normalizer/          # Raw → canonical normalization service
├── turbo.json
├── package.json
└── CLAUDE.md
```

The `packages/db` package owns the Prisma schema and is imported by `apps/web`
and all services. tRPC router definitions live in `packages/api` so they can be
imported by `apps/web` for both API route handlers and client-side type
inference. The Next.js app handles both frontend rendering and API routes —
there is no separate API server.

---

## Deployment

### Target environment

- **K3s** (lightweight Kubernetes) on a Raspberry Pi 4B with 8GB RAM and
  external NVMe SSD.
- **Traefik** as ingress controller (bundled with K3s), using Gateway API
  HTTPRoutes.
- Pi-hole runs on the same hardware but **outside K3s** as a standalone service
  to avoid DNS dependency on the cluster.

### Service containerization

- All services are containerized with Docker.
- Fetchers and normalizer run as K8s CronJobs (short-lived, bursty workload).
- Next.js runs as a Deployment (using `standalone` output mode — ~30MB Node
  server). Serves both the frontend and tRPC API routes.
- PostgreSQL runs as a StatefulSet (or external if preferred).

### Resource budget (approximate)

| Service                   | RAM        |
|---------------------------|------------|
| K3s control plane         | ~500-700MB |
| PostgreSQL                | ~512MB     |
| Next.js (standalone)      | ~150-200MB |
| Traefik                   | ~128MB     |
| Fetcher CronJobs (peak)   | ~384MB     |
| Normalizer CronJob (peak) | ~256MB     |

Steady-state usage is ~1.5GB, leaving plenty of headroom on the 8GB Pi. ISR
pages are cached to disk after first render, so the Next.js process isn't doing
continuous SSR work. Basemap tiles are served externally (hosted free tier), so
there's no tile server to run locally.

---

## Auth Flow

1. User clicks "Sign in with Discord"
2. Frontend redirects to Discord OAuth2 authorization URL
3. Discord redirects back with auth code
4. API exchanges code for tokens, fetches Discord user profile
5. API creates or updates user in `users` table, issues JWT or session cookie
6. Optional: user can later link iRacing customer ID or LFM profile for
   personalization

No Keycloak — it's overkill for one or two OAuth providers. Use `arctic` (OAuth
library) or `lucia` (session library) in the API server.

---

## Coding Conventions

- **TypeScript** everywhere — frontend, API, fetchers, normalizer.
- **Prisma** as the single source of truth for DB schema. Run `prisma generate`
  after schema changes.
- **tRPC** routers provide end-to-end type safety from API to frontend via
  `@trpc/next`. Never manually duplicate types between API and web — they should
  flow from tRPC inference.
- **Zod** for runtime validation on tRPC inputs.
- **Vitest** for all tests across the monorepo. Configured at the workspace
  level via Turborepo.
- **Next.js App Router** conventions: use React Server Components by default.
  Mark components/pages with `'use client'` only when they need browser APIs (
  MapLibre GL, user interactions, hooks). ISR pages use `revalidate` exports.
- Use `bun install` for package management (faster than npm).

---

## Key Domain Concepts

- **Platform**: A sim racing application (iRacing, LMU, LFM). New platforms
  should be easy to add via a new fetcher service.
- **Canonical vs platform-specific**: Cars, tracks, and classes have canonical
  entries (the real-world identity) and platform-specific mappings. Always
  normalize to canonical before storing in user-facing tables.
- **NormalizedRaceEvent**: The shared contract all fetchers must produce after
  normalization. Design fetcher interfaces to be pluggable.
- **Source type**: Race events have a `sourceType` (`PLATFORM_OFFICIAL`,
  `LEAGUE`, `COMMUNITY`). V1 focuses on `PLATFORM_OFFICIAL` only. Leagues will
  come later via a separate tab with user-driven data entry.

---

## Future Considerations (not for V1, but design for them)

- **Leagues tab**: User-submitted league schedules. The `sourceType` enum and
  `Series` model already anticipate this.
- **Notifications**: If users heavily adopt favorites, push notifications for
  upcoming races on favorited tracks/classes.
- **Multiclass events**: A `MulticlassEvent` join table for races with multiple
  classes running simultaneously (like IMSA GTP + GTD).
- **Fetcher health monitoring**: A `FetcherLog` table tracking last success,
  last failure, error counts per platform.
- **OpenSearch**: If search becomes a bottleneck, add as a secondary index (
  Postgres stays source of truth, sync a subset of fields to OpenSearch for
  full-text/faceted search).
- **On-demand ISR revalidation**: Wire fetcher completion events to Next.js
  revalidation webhooks so SEO pages update immediately when schedules change,
  rather than waiting for the `revalidate` interval.
- **Self-hosted tile server**: Add Martin (Rust) to serve vector tiles from
  PostGIS and/or PMTiles for full basemap customization matching the
  Asphalt/Circuit Dawn palette. Replaces the hosted free tier basemap.