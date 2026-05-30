# AI Meeting Scheduler — Phased Development Plan

> Project: 313-ai-meeting-scheduler · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files into a concrete, phased build. The product is an open-source, self-hostable, AI-native meeting scheduler that combines link-based booking (Calendly-class), preference-learning AI (beyond Motion/Reclaim), focus-block protection, and an MCP server for agentic scheduling. It targets self-hosted and cloud deployment with full data ownership and is built on open calendar standards (RFC 5545 iCalendar, RFC 4791 CalDAV, RFC 5546 iTIP, RFC 6047 iMIP, OAuth 2.0/PKCE).

---

## Core Requirements Summary

- **What it does**: Connects a user's Google/Outlook/CalDAV calendars, exposes shareable booking links with configurable availability, computes conflict-free slots in real time, ranks slots with a transparent preference-learning engine, emits RFC 5545 invites, and exposes both a REST API (OpenAPI 3.1) and an MCP server for AI agents.
- **Primary users**: Executives/consultants protecting deep work; sales teams running high-volume prospect calls; distributed engineering teams across time zones; HR/recruiting coordinators running interview loops.
- **Key differentiators vs incumbents**: open-source + self-hostable (vs Motion/Reclaim/Calendly), transparent and auditable AI slot scoring with natural-language override (vs opaque heuristics), and a first-class MCP server for agent-to-agent scheduling (a defined market gap).
- **Deployment model**: Self-hosted (Docker Compose) and cloud (same image). Full data ownership for self-hosters.
- **Integration surface**: Google Calendar API v3, Microsoft Graph Calendar, generic CalDAV (RFC 4791/6638), Zoom/Google Meet/Teams conferencing links, SMTP for iMIP delivery, outbound webhooks, REST API, MCP server, LLM provider (for preference explanations and NL override parsing).
- **Standards to implement**: RFC 5545 (iCalendar), RFC 4791 + RFC 6638 + RFC 6578 (CalDAV + scheduling + sync), RFC 5546 (iTIP), RFC 6047 (iMIP), RFC 7953 (VAVAILABILITY semantics), OAuth 2.0/2.1 + PKCE (RFC 7636), OpenAPI 3.1, JSON Schema 2020-12, MCP, ISO 8601, GDPR data-export/erasure.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **TypeScript (Node.js 22 LTS)** | The project is API/integration/frontend-heavy, not ML-training-heavy. Google (`googleapis`), Microsoft (`@microsoft/microsoft-graph-client`), and the MCP TypeScript SDK are all first-class in Node. A single language across API, MCP server, workers, and web UI minimises context-switching and lets the booking widget share types with the backend. |
| Runtime/package manager | **pnpm** workspaces (monorepo) | Multiple deployables (API, worker, web, MCP server) share `packages/core` and `packages/db`. pnpm's content-addressed store keeps the Docker image small and installs deterministic. |
| API framework | **Fastify 5** + `@fastify/swagger` | Fastify's JSON-Schema-first design produces an OpenAPI 3.1 document for free (a hard requirement from standards.md). Faster than Express; first-class TypeScript types. |
| Validation / schema | **Zod** + `zod-to-openapi` | One Zod schema per payload validates input at runtime and generates JSON Schema 2020-12 for the OpenAPI spec — single source of truth. |
| Database | **PostgreSQL 16** | The hybrid relational+JSONB data model (suggestion 3) needs JSONB with GIN indexes, `TIMESTAMPTZ`, range types for overlap checks, and `EXCLUDE` constraints to prevent double-booking. SQLite cannot do these. Postgres is the only target. |
| ORM / migrations | **Drizzle ORM** + `drizzle-kit` | Type-safe SQL-first ORM that supports raw Postgres features (JSONB, `tstzrange`, partial indexes, generated columns) without fighting the abstraction. Migrations are plain SQL files, reviewable in CI. |
| Task queue | **BullMQ** (Redis-backed) | Calendar sync, webhook delivery, reminder/follow-up scheduling, and LLM calls are async and must survive restarts with retries and backoff. BullMQ provides delayed jobs (perfect for "remind 15 min before") and repeatable jobs (periodic incremental sync). |
| Cache / locks | **Redis 7** | Backs BullMQ, free/busy cache, OAuth-state nonces, and Redlock distributed locks used when committing a booking (prevents race-condition double-booking). |
| LLM access | **Vercel AI SDK** (`ai`) with provider adapters | Preference explanations, natural-language override parsing, and (later) the email agent need an LLM. The AI SDK abstracts providers (Anthropic, OpenAI, local) so self-hosters can point at any model. All AI is *advisory*: scoring math is deterministic; the LLM only explains and parses. |
| Frontend | **Next.js 16 (App Router) + React 19 + Tailwind + shadcn/ui** | Needs an admin dashboard (event types, availability, analytics) and public booking pages (SSR for SEO + fast first paint). Server Components reduce client JS on the public booking page. |
| Embeddable widget | **Vanilla TS web component** (`packages/widget`) bundled to a single `<script>` | Mirrors Calendly/Cal.com embed. Must run framework-agnostically on any third-party site; a web component avoids shipping React to embedders. |
| Calendar libraries | **`ical-generator`** (emit), **`node-ical`** / **`ical.js`** (parse), **`tsdav`** (CalDAV client) | `ical-generator` emits RFC 5545 VEVENT/VFREEBUSY; `tsdav` speaks CalDAV (RFC 4791/6638) for Apple/Nextcloud/Fastmail. |
| Calendar provider SDKs | **`googleapis`**, **`@microsoft/microsoft-graph-client`** + **`@azure/msal-node`** | Official SDKs for the two dominant providers; MSAL handles Microsoft OAuth + token refresh. |
| MCP server | **`@modelcontextprotocol/sdk`** (TypeScript) | standards.md flags a first-mover MCP advantage. The server exposes scheduling tools (`list_event_types`, `find_slots`, `create_booking`, `get_freebusy`) over stdio + streamable HTTP. |
| Auth (app users) | **Lucia-style sessions** via `@fastify/secure-session` + OAuth login (Google/Microsoft) | App login reuses the same OAuth providers we already integrate; sessions are cookie-based and CSRF-protected. API clients use API keys / OAuth 2.1. |
| Email delivery | **Nodemailer** (SMTP) | Self-hosters configure any SMTP server; iMIP (RFC 6047) requires sending `.ics` as a `text/calendar; method=REQUEST` MIME part, which Nodemailer supports via `icalEvent`. |
| Testing | **Vitest** (unit/integration) + **Playwright** (E2E) + **Testcontainers** (real Postgres/Redis) | Vitest is fast and TS-native. Testcontainers spins ephemeral Postgres/Redis for integration tests. Playwright drives the booking flow and dashboard E2E. |
| Code quality | **Biome** (lint + format) + **tsc** (strict) | Biome is a single fast tool replacing ESLint+Prettier. `tsc --noEmit` in CI enforces strict types. |
| Containerisation | **Docker** + **docker-compose.yml** | One image runs `api`, `worker`, `mcp`, or `web` selected by command; compose wires Postgres + Redis + the four services for one-command self-hosting. |
| Secrets / token encryption | **AES-256-GCM** via Node `crypto`, key from env | OAuth refresh tokens are stored encrypted at rest (GDPR / standards.md TLS+AES baseline). A single `ENCRYPTION_KEY` env var drives an envelope-encryption helper. |
| CI | **GitHub Actions** | Lint, typecheck, unit+integration (Testcontainers), `.ics` RFC validation, OpenAPI diff, Docker build. |

### Project Structure

```
ai-meeting-scheduler/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── biome.json
├── tsconfig.base.json
├── Dockerfile                        # multi-stage; ENTRYPOINT selects service via $SERVICE
├── docker-compose.yml                # postgres, redis, api, worker, web, mcp
├── docker-compose.dev.yml
├── .env.example
├── packages/
│   ├── core/                         # framework-agnostic domain logic (no HTTP, no DB driver)
│   │   ├── src/
│   │   │   ├── availability/         # free-slot computation, interval math
│   │   │   │   ├── intervals.ts       # Interval type, merge/subtract/intersect
│   │   │   │   ├── slot-generator.ts  # availability rules → candidate slots
│   │   │   │   └── conflict.ts        # apply busy times, buffers, min-notice
│   │   │   ├── scoring/              # preference-learning slot scorer (deterministic)
│   │   │   │   ├── features.ts        # extract slot features
│   │   │   │   ├── scorer.ts          # weighted score + explanation factors
│   │   │   │   └── learner.ts         # update weights from signals
│   │   │   ├── ical/                 # RFC 5545 emit/parse wrappers
│   │   │   │   ├── vevent.ts
│   │   │   │   ├── vfreebusy.ts
│   │   │   │   └── itip.ts            # REQUEST/REPLY/CANCEL builders (RFC 5546)
│   │   │   ├── timezone/              # IANA tz math (Luxon)
│   │   │   ├── routing/               # routing-form rule evaluation
│   │   │   └── types.ts               # shared domain types (mirrors DB)
│   │   └── test/
│   ├── db/                           # Drizzle schema, migrations, query helpers
│   │   ├── src/
│   │   │   ├── schema/                # one file per table group
│   │   │   ├── migrations/            # generated SQL
│   │   │   ├── client.ts              # pool + drizzle instance
│   │   │   └── repositories/          # typed data-access functions
│   │   └── drizzle.config.ts
│   ├── integrations/                 # external-system adapters behind one interface
│   │   ├── src/
│   │   │   ├── calendar/
│   │   │   │   ├── provider.ts        # CalendarProvider interface
│   │   │   │   ├── google.ts
│   │   │   │   ├── microsoft.ts
│   │   │   │   └── caldav.ts
│   │   │   ├── conferencing/          # zoom.ts, meet.ts, teams.ts
│   │   │   ├── llm/                   # AI SDK wrapper, prompt templates
│   │   │   └── email/                 # nodemailer + iMIP MIME assembly
│   │   └── test/
│   └── widget/                       # embeddable booking web component
│       └── src/
├── apps/
│   ├── api/                          # Fastify REST API + OpenAPI + webhooks
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/               # auth, db, queue, swagger, error handler
│   │   │   ├── routes/                # event-types, availability, bookings, public, webhooks, oauth
│   │   │   └── schemas/               # Zod request/response schemas
│   │   └── test/
│   ├── worker/                       # BullMQ processors
│   │   └── src/
│   │       ├── queues.ts
│   │       └── processors/            # sync, webhook-delivery, reminders, llm-explain
│   ├── mcp/                          # MCP server (stdio + HTTP)
│   │   └── src/
│   │       ├── server.ts
│   │       └── tools/
│   └── web/                          # Next.js dashboard + public booking pages
│       └── src/app/
└── docs/
    ├── openapi.json                  # generated, committed for diffing
    └── self-hosting.md
```

The structure is grouped by concern (domain core, data, integrations, deployables), so each phase adds files without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Domain Types

### Purpose
Establish the repository skeleton, tooling, the PostgreSQL schema, and the framework-agnostic domain types that every later phase depends on. After this phase, the database migrates cleanly, the test harness runs against a real Postgres via Testcontainers, and CI is green on an empty-but-wired codebase.

### Tasks

#### 1.1 — Monorepo & tooling bootstrap

**What**: A pnpm workspace with Biome, strict TypeScript, Vitest, and a CI workflow.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `tsconfig.base.json`: `"strict": true`, `"moduleResolution": "bundler"`, `"target": "ES2023"`, `"verbatimModuleSyntax": true`. Each package extends it.
- `biome.json`: formatter (2-space, single quotes, trailing commas), linter recommended + `noUnusedImports: error`.
- Root scripts: `build`, `lint`, `typecheck`, `test`, `test:int`, `db:generate`, `db:migrate`.
- `.env.example` enumerates every env var with defaults:
  ```
  DATABASE_URL=postgres://scheduler:scheduler@localhost:5432/scheduler
  REDIS_URL=redis://localhost:6379
  ENCRYPTION_KEY=            # 32-byte base64; required
  APP_BASE_URL=http://localhost:3000
  API_BASE_URL=http://localhost:4000
  GOOGLE_CLIENT_ID=
  GOOGLE_CLIENT_SECRET=
  MS_CLIENT_ID=
  MS_CLIENT_SECRET=
  SMTP_URL=
  LLM_PROVIDER=anthropic     # anthropic|openai|none
  LLM_API_KEY=
  LLM_MODEL=claude-sonnet-4-6
  ```
- `.github/workflows/ci.yml`: matrix job running `lint`, `typecheck`, `test`, `test:int` (with Postgres + Redis service containers), Docker build.

**Testing**:
- `Unit: config loader with all required env present → typed Config object`
- `Unit: config loader missing ENCRYPTION_KEY → throws with "ENCRYPTION_KEY is required"`
- `Unit: config loader with non-base64 ENCRYPTION_KEY → throws "must be 32-byte base64"`
- `CI smoke: pnpm install && pnpm build && pnpm lint && pnpm typecheck all exit 0 on empty scaffold`

#### 1.2 — Database schema (hybrid relational + JSONB)

**What**: Drizzle schema and migrations implementing the data model (suggestion 3) with double-booking prevention.

**Design**:
Adopt **Data Model 3 (Hybrid Relational + JSONB)** as the canonical schema: typed columns for entities queried in hot paths (bookings, time ranges, foreign keys) and JSONB for variable-shape config (availability rules, routing conditions, provider metadata, AI feature vectors). Core tables (DDL excerpt; full DDL lives in `packages/db/src/schema/`):

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- for EXCLUDE constraint

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  name TEXT,
  timezone TEXT NOT NULL DEFAULT 'UTC',       -- IANA id
  locale TEXT NOT NULL DEFAULT 'en',
  settings JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE calendar_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider TEXT NOT NULL CHECK (provider IN ('google','microsoft','apple','caldav')),
  email_address TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','syncing','error','disconnected')),
  oauth_tokens_enc TEXT,            -- AES-256-GCM envelope (access+refresh+expiry)
  caldav_url TEXT,
  sync_state JSONB NOT NULL DEFAULT '{}',  -- {syncToken, deltaLink, lastSyncedAt}
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id, provider, email_address)
);

CREATE TABLE event_types (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  team_id UUID REFERENCES teams(id),
  slug TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  duration_minutes INT NOT NULL DEFAULT 30,
  scheduling_type TEXT NOT NULL DEFAULT 'individual'
    CHECK (scheduling_type IN ('individual','round_robin','collective')),
  config JSONB NOT NULL DEFAULT '{}',  -- buffers, min_notice, max_future, location, questions
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id, slug)
);

CREATE TABLE availability_schedules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name TEXT NOT NULL DEFAULT 'Working Hours',
  timezone TEXT NOT NULL DEFAULT 'UTC',
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
  rules JSONB NOT NULL DEFAULT '[]',     -- [{dayOfWeek,start,end}], maps to VAVAILABILITY
  overrides JSONB NOT NULL DEFAULT '[]', -- [{date,start,end,unavailable}]
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type_id UUID NOT NULL REFERENCES event_types(id),
  host_id UUID NOT NULL REFERENCES users(id),
  uid TEXT NOT NULL UNIQUE,              -- RFC 5545 UID
  external_event_ids JSONB NOT NULL DEFAULT '{}', -- {provider: providerEventId}
  title TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'confirmed'
    CHECK (status IN ('pending','confirmed','cancelled','rescheduled','no_show','completed')),
  time_range TSTZRANGE NOT NULL,        -- [start,end)
  timezone TEXT NOT NULL,
  location JSONB NOT NULL DEFAULT '{}',  -- {type, value, meetingUrl}
  answers JSONB NOT NULL DEFAULT '[]',
  ai_meta JSONB NOT NULL DEFAULT '{}',   -- {suggestedRank, score, explanation}
  rescheduled_from_id UUID REFERENCES bookings(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- prevent double-booking the same host for overlapping confirmed/pending meetings
  EXCLUDE USING gist (
    host_id WITH =,
    time_range WITH &&
  ) WHERE (status IN ('confirmed','pending'))
);

CREATE INDEX idx_bookings_host_time ON bookings USING gist (host_id, time_range);
CREATE INDEX idx_bookings_status ON bookings (host_id, status);
CREATE INDEX idx_event_types_config ON event_types USING gin (config);

CREATE TABLE invitees (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  name TEXT,
  timezone TEXT,
  rsvp_status TEXT NOT NULL DEFAULT 'accepted'
    CHECK (rsvp_status IN ('accepted','declined','tentative','needs_action'))
);
```

AI, teams, routing, and workflow tables (`scheduling_preferences`, `preference_signals`, `focus_blocks`, `teams`, `team_members`, `routing_forms`, `workflows`, `webhooks`, `api_keys`) are created in the phases that introduce them — migrations are additive. The `EXCLUDE USING gist` constraint is the load-bearing double-booking guard and must exist from Phase 1 so booking logic can rely on it.

**Testing**:
- `Integration (Testcontainers): run all migrations on fresh Postgres → no error, schema_migrations populated`
- `Integration: insert two confirmed bookings, same host, overlapping time_range → second insert raises exclusion_violation`
- `Integration: insert overlapping bookings where one is 'cancelled' → both succeed (partial WHERE clause respected)`
- `Integration: insert booking with empty time_range start>=end → CHECK/range error`
- `Integration: delete user → cascades to calendar_accounts, event_types, bookings' FK behavior verified`

#### 1.3 — Domain types & encryption helper

**What**: Shared TypeScript domain types in `packages/core/types.ts` and an AES-256-GCM token-encryption helper in `packages/db`.

**Design**:
```ts
// packages/core/src/types.ts
export type Interval = { start: Date; end: Date }; // [start, end)
export type Provider = 'google' | 'microsoft' | 'apple' | 'caldav';

export interface AvailabilityRule { dayOfWeek: 0|1|2|3|4|5|6; start: string; end: string } // "09:00"
export interface EventTypeConfig {
  bufferBeforeMin: number; bufferAfterMin: number;
  minNoticeHours: number; maxFutureDays: number;
  location: { type: 'video'|'phone'|'in_person'|'custom'; value?: string };
  questions: Array<{ id: string; label: string; type: string; required: boolean; options?: string[] }>;
}

// packages/db/src/crypto.ts
export function encryptTokens(plaintext: string, key: Buffer): string; // returns base64(iv|tag|ct)
export function decryptTokens(payload: string, key: Buffer): string;   // throws on tag mismatch
```
The encryption helper uses `aes-256-gcm` with a random 12-byte IV per call; output is `base64(iv ‖ authTag ‖ ciphertext)`.

**Testing**:
- `Unit: encrypt then decrypt round-trips arbitrary UTF-8 string`
- `Unit: decrypt with wrong key → throws (auth tag mismatch)`
- `Unit: decrypt with tampered ciphertext byte → throws`
- `Unit: two encryptions of same plaintext produce different output (random IV)`

---

## Phase 2: Availability Engine — Free-Slot Computation

### Purpose
Implement the deterministic heart of any scheduler: given availability rules, existing busy intervals, and event-type constraints, compute the exact set of bookable slots in the invitee's timezone. This is pure domain logic with no I/O, so it is fully unit-testable and forms the foundation the booking and AI phases build on.

### Tasks

#### 2.1 — Interval algebra

**What**: Functions to merge, subtract, and intersect half-open time intervals.

**Design**:
```ts
// packages/core/src/availability/intervals.ts
export function merge(intervals: Interval[]): Interval[];       // sort + coalesce overlaps
export function subtract(base: Interval[], remove: Interval[]): Interval[]; // base minus busy
export function intersect(a: Interval[], b: Interval[]): Interval[];
export function clamp(i: Interval, window: Interval): Interval | null;
```
All intervals are half-open `[start, end)`; comparisons use millisecond epoch. No timezone logic here — inputs are absolute `Date`s.

**Testing**:
- `Unit: merge [9-10, 9:30-11] → [9-11]`
- `Unit: merge adjacent [9-10, 10-11] → [9-11]` (touching boundaries coalesce)
- `Unit: subtract [9-17] minus [12-13] → [9-12, 13-17]`
- `Unit: subtract [9-17] minus [8-18] → []`
- `Unit: intersect [9-12, 14-16] with [11-15] → [11-12, 14-15]`
- `Unit: empty inputs → empty output for all four functions`

#### 2.2 — Slot generator from availability rules

**What**: Expand weekly availability rules + date overrides into concrete candidate intervals across a date window, in the schedule's timezone, honoring DST.

**Design**:
```ts
// packages/core/src/availability/slot-generator.ts
export function expandAvailability(
  schedule: { timezone: string; rules: AvailabilityRule[]; overrides: DateOverride[] },
  window: Interval,            // absolute UTC window to expand into
): Interval[];
```
Algorithm:
1. For each day in `window` (computed in `schedule.timezone` via Luxon), find matching `rules` by `dayOfWeek`.
2. Build day-local `start`/`end` `DateTime`s in the schedule tz, convert to UTC `Date` (Luxon handles DST gaps/overlaps; a 09:00 rule on a spring-forward day shifts correctly).
3. Apply `overrides`: a date with `unavailable=true` removes that day's intervals; an override with times replaces them.
4. Clamp results to `window`, merge, return sorted.

**Testing**:
- `Unit: Mon-Fri 09:00-17:00 in 'America/New_York' over one week → 5 intervals, each 8h in UTC`
- `Unit: DST spring-forward day (2026-03-08 US) → 09:00-17:00 local still maps to correct UTC offset`
- `Unit: override date unavailable=true → that day produces no intervals`
- `Unit: override with 10:00-12:00 replaces the default rule for that date`
- `Unit: window narrower than rules → intervals clamped to window`

#### 2.3 — Bookable-slot computation

**What**: Combine availability, busy times, and event-type constraints into final bookable start times.

**Design**:
```ts
// packages/core/src/availability/conflict.ts
export function computeSlots(input: {
  availability: Interval[];   // from expandAvailability
  busy: Interval[];           // from calendar free/busy + focus blocks + existing bookings
  durationMin: number;
  bufferBeforeMin: number; bufferAfterMin: number;
  minNoticeHours: number; maxFutureDays: number;
  granularityMin: number;     // slot step, default 15
  now: Date;
}): Interval[];               // each interval is exactly durationMin long
```
Algorithm:
1. Inflate each `busy` interval by `bufferBefore`/`bufferAfter`, then `merge`.
2. `free = subtract(availability, inflatedBusy)`.
3. Enforce earliest = `now + minNotice`, latest = `now + maxFutureDays`; clamp `free`.
4. Within each free interval, step by `granularityMin` emitting `[t, t+duration)` slots that fit entirely inside the interval.

**Testing**:
- `Unit: 8h availability, 1 busy hour midday, 30-min duration, 15-min granularity → expected count of slots, none overlapping busy+buffer`
- `Unit: 15-min buffer before/after a 30-min meeting consumes 60 min of availability per booked slot`
- `Unit: minNotice=24h → no slots in next 24h`
- `Unit: maxFutureDays=7 → no slots beyond 7 days`
- `Unit: duration longer than any free interval → []`
- `Property test: emitted slots never intersect any inflated busy interval`

---

## Phase 3: REST API & Public Booking Flow (MVP Core)

### Purpose
Ship the user-facing MVP backbone: a Fastify API with auto-generated OpenAPI 3.1, CRUD for event types and availability, a public endpoint that returns slots (using Phase 2), and a booking endpoint that atomically creates a booking guarded by the Phase 1 exclusion constraint. After this phase, a meeting can be booked end-to-end via API even before calendar sync exists (using only internal bookings as busy times).

### Tasks

#### 3.1 — Fastify server, plugins, OpenAPI

**What**: The API process with DB pool, error handling, auth scaffolding, and Swagger/OpenAPI generation.

**Design**:
- `server.ts` registers plugins: `@fastify/swagger` (+ `@fastify/swagger-ui` at `/docs`), a `db` plugin (Drizzle pool), a `queue` plugin (BullMQ connections, no-op if Redis absent in tests), an `auth` plugin (session + API-key strategies), and a global error handler that maps domain errors → RFC 7807 problem+json.
- Every route declares Zod schemas; a `buildOpenApi()` step writes `docs/openapi.json` and CI fails if it drifts from committed.
- Error contract:
  ```json
  { "type": "https://errors.scheduler/slot-unavailable",
    "title": "Slot no longer available", "status": 409, "detail": "..." }
  ```

**Testing**:
- `Integration: GET /healthz → 200 {status:"ok", db:"up"}`
- `Integration: GET /docs/json → valid OpenAPI 3.1 document (validated with @readme/openapi-parser)`
- `Integration: unknown route → 404 problem+json`
- `Integration: handler throwing DomainError(409) → 409 problem+json with type/title`

#### 3.2 — Event-type & availability CRUD

**What**: Authenticated CRUD endpoints for event types and availability schedules.

**Design**:
Endpoints (all under `/v1`, require session or API key for the owning user):
| Method | Path | Body / Result |
|--------|------|---------------|
| POST | `/event-types` | `EventTypeCreate` → `EventType` |
| GET | `/event-types` | → `EventType[]` |
| PATCH | `/event-types/:id` | partial → `EventType` |
| DELETE | `/event-types/:id` | → 204 |
| POST | `/availability` | `AvailabilityCreate` → `AvailabilitySchedule` |
| GET/PATCH/DELETE | `/availability/:id` | … |

`EventTypeCreate` (Zod): `{ slug, title, durationMinutes, schedulingType, config: EventTypeConfig }`. `slug` validated `^[a-z0-9-]+$`; uniqueness per user enforced by DB, surfaced as 409.

**Testing**:
- `Integration: POST valid event type → 201, persisted, slug normalized`
- `Integration: POST duplicate slug for same user → 409 problem+json`
- `Integration: POST with durationMinutes=0 → 422 validation error naming the field`
- `Integration: PATCH event type owned by another user → 403`
- `Integration: availability rules with end<=start → 422`

#### 3.3 — Public slots endpoint

**What**: Unauthenticated endpoint returning bookable slots for a public event type.

**Design**:
```
GET /public/:userSlug/:eventTypeSlug/slots?from=ISO&to=ISO&timezone=IANA
→ 200 { timezone, slots: [{ start: ISO, end: ISO }] }
```
Pipeline: load event type → load default availability → gather busy times (Phase 3 = internal `bookings` only; Phase 4 adds external calendars) → `expandAvailability` → `computeSlots` → format in requested `timezone`. Results cached in Redis 60s keyed by `(eventTypeId, from, to)`.

**Testing**:
- `Integration: slots for event type with one existing booking → booked slot excluded`
- `Integration: from/to spanning 2 weeks but maxFutureDays=7 → slots capped at 7 days`
- `Integration: invalid timezone param → 422`
- `Integration: inactive event type → 404`
- `Integration: identical request twice → second served from cache (assert single DB query via spy)`

#### 3.4 — Booking creation (atomic)

**What**: Public endpoint that creates a booking, re-validating availability under a lock to prevent races.

**Design**:
```
POST /public/:userSlug/:eventTypeSlug/book
body: { start: ISO, name, email, timezone, answers? }
→ 201 { booking } | 409 slot-unavailable
```
Algorithm:
1. Acquire Redlock `lock:host:{hostId}` (TTL 5s).
2. Recompute slots for the requested window; assert `start` is still bookable → else 409.
3. Generate `uid` (`{ulid}@{APP_HOST}`), build `tstzrange [start, start+duration)`.
4. `INSERT` booking + invitee in one transaction. The `EXCLUDE` constraint is the final backstop: on `exclusion_violation` → 409.
5. Enqueue jobs: `calendar.create` (Phase 4), `notify.confirmation` (Phase 6), `webhook.dispatch` (Phase 8). Enqueue is best-effort; queue absence does not fail the booking.
6. Release lock; return booking.

**Testing**:
- `Integration: book a free slot → 201, booking persisted, uid present`
- `Integration: book the same slot twice concurrently (Promise.all) → exactly one 201, one 409`
- `Integration: book a slot outside availability → 409`
- `Integration: book required-question event without answer → 422`
- `Integration (mocked queue): successful booking enqueues calendar.create + notify.confirmation jobs`

---

## Phase 4: Calendar Provider Integration & Sync

### Purpose
Connect real Google, Microsoft, and CalDAV calendars so free/busy reflects the user's actual schedule and confirmed bookings are written back to their calendar. This turns the scheduler from a standalone booking DB into a true calendar coordinator. Built behind a single `CalendarProvider` interface so providers are pluggable.

### Tasks

#### 4.1 — CalendarProvider interface & OAuth connect flow

**What**: A provider abstraction plus OAuth 2.0 + PKCE connect/callback endpoints that store encrypted tokens.

**Design**:
```ts
// packages/integrations/src/calendar/provider.ts
export interface CalendarProvider {
  getFreeBusy(account: Account, window: Interval): Promise<Interval[]>;
  createEvent(account: Account, e: CalEvent): Promise<{ externalId: string; meetingUrl?: string }>;
  updateEvent(account: Account, externalId: string, e: CalEvent): Promise<void>;
  deleteEvent(account: Account, externalId: string): Promise<void>;
  incrementalSync(account: Account): Promise<{ changes: BusyChange[]; nextState: SyncState }>;
}
```
OAuth endpoints:
```
GET  /v1/calendars/connect/:provider   → 302 to provider auth URL (PKCE verifier+state in Redis, 10-min TTL)
GET  /v1/calendars/callback/:provider  → exchange code, store calendar_accounts.oauth_tokens_enc, 302 to dashboard
DELETE /v1/calendars/:accountId        → revoke + mark disconnected
```
Scopes minimised per standards.md: Google `calendar.events` + `calendar.readonly`; Microsoft `Calendars.ReadWrite` + `offline_access`. A `TokenManager` decrypts tokens, refreshes when `expiry < now+60s`, re-encrypts on refresh.

**Testing**:
- `Integration (mocked provider): connect → callback exchanges code, tokens stored encrypted (assert ciphertext != plaintext)`
- `Unit: TokenManager refreshes when expiring within 60s, persists new token`
- `Integration: callback with mismatched state → 400, no account created`
- `Integration: DELETE account → status 'disconnected', tokens nulled`

#### 4.2 — Google & Microsoft providers

**What**: Concrete implementations using `googleapis` and Microsoft Graph.

**Design**:
- **Google**: `getFreeBusy` → `freebusy.query`; `createEvent` → `events.insert` with `conferenceData` for Meet links; `incrementalSync` uses `syncToken` (RFC 6578-style delta); expired token → full resync.
- **Microsoft**: `getFreeBusy` → `/me/calendar/getSchedule`; `createEvent` → `POST /me/events` with `onlineMeeting` for Teams; `incrementalSync` uses Graph `deltaLink`.
- Both map provider events into the internal `CalEvent` (DTSTART/DTEND/SUMMARY/UID) and write `bookings.external_event_ids[provider]`.

**Testing**:
- `Integration (nock/recorded fixtures): Google getFreeBusy parses busy array → Interval[]`
- `Integration (fixtures): Google createEvent returns Meet URL → stored on booking.location.meetingUrl`
- `Integration (fixtures): Microsoft getSchedule maps scheduleItems → Interval[]`
- `Unit: provider error 401 → throws TokenExpiredError (triggers refresh-and-retry)`
- `Contract test: both providers satisfy the CalendarProvider interface (shared test suite run against each)`

#### 4.3 — CalDAV provider

**What**: Generic CalDAV support (Apple iCloud, Nextcloud, Fastmail) via `tsdav`.

**Design**:
- `getFreeBusy`: CalDAV `REPORT` free-busy-query (RFC 6638) where supported; fallback to fetching VEVENTs in the window and computing busy locally.
- `createEvent`: `PUT` an `.ics` (RFC 5545) to the calendar collection; `incrementalSync` uses CalDAV `sync-collection` (RFC 6578) with `sync_state.syncToken`.
- Auth via app-specific password (iCloud) or basic/OAuth depending on server; stored in `oauth_tokens_enc`.

**Testing**:
- `Integration (Radicale test container): PUT event then REPORT free-busy → event appears as busy`
- `Integration: sync-collection returns sync-token; second call with token returns only changes`
- `Unit: VEVENT with RRULE expands to correct busy occurrences in window`

#### 4.4 — Sync worker & free/busy aggregation

**What**: BullMQ jobs that periodically sync calendars and a busy-time aggregator feeding the slots endpoint.

**Design**:
- Repeatable job `sync.account` per active account every 5 min calls `incrementalSync`, caching busy intervals in Redis (`busy:{accountId}`, TTL 10 min).
- `gatherBusy(userId, window)` unions cached busy from all the user's conflict-checked calendars + `focus_blocks` (Phase 5) + internal pending/confirmed bookings, then `merge`s. Phase 3's slots endpoint switches to call `gatherBusy`.
- `calendar.create` job (enqueued by booking) calls `provider.createEvent` and writes back `external_event_ids` + `meetingUrl`; on failure, retries with backoff and marks the booking `ai_meta.syncError` after max attempts (booking still valid locally).

**Testing**:
- `Integration (mocked providers): gatherBusy unions two accounts + bookings → merged intervals`
- `Integration: sync.account with stale Redis cache → refreshes from provider`
- `Integration: calendar.create job success → booking.external_event_ids populated`
- `Integration: calendar.create job fails 3x → booking flagged syncError, not deleted`

---

## Phase 5: AI Preference-Learning Engine & Focus Blocks

### Purpose
Deliver the project's core differentiator: a transparent, auditable preference-learning engine that re-ranks bookable slots by inferred user preferences, plus AI-detected focus-block protection. The scoring is deterministic math (explainable, no LLM in the hot path); the LLM only renders natural-language explanations and parses NL overrides. This is the "AI-native advantage" from the README.

### Tasks

#### 5.1 — Preference schema & signal capture

**What**: Migrations for `scheduling_preferences`, `preference_signals`, `focus_blocks`; signal-logging hooks on booking actions.

**Design**:
Tables per data-model-suggestion-1 §"AI Preferences & Focus Blocks" (typed-row preferences + signal log + focus blocks). Signals are logged on: booking created (`slot_chosen` for the chosen slot, `slot_skipped` for higher-ranked slots the invitee passed over), booking declined/cancelled by host, focus-block override.
```ts
interface PreferenceSignal {
  userId: string; signalType: 'slot_chosen'|'slot_skipped'|'booking_declined'|'reschedule_initiated'|'focus_block_overridden';
  contextTimeOfDay: string; contextDayOfWeek: number;
  contextMeetingType?: string; contextDurationMin?: number; createdAt: Date;
}
```

**Testing**:
- `Integration: booking creation logs one slot_chosen + slot_skipped for each higher-ranked unbooked slot`
- `Integration: host cancels booking → booking_declined signal with correct context`
- `Unit: signal context derived from booking start in host timezone (time-of-day bucketed)`

#### 5.2 — Deterministic slot scorer

**What**: A feature-based linear scorer that ranks Phase 2 slots by learned preference weights, producing a score and explanation factors.

**Design**:
```ts
// packages/core/src/scoring/scorer.ts
interface SlotFeatures {
  hourOfDay: number; dayOfWeek: number; isAfterLunch: boolean;
  meetingsAlreadyThatDay: number; adjacentToExistingMeeting: boolean;
  minutesFromPreferredWindow: number;
}
interface ScoredSlot { slot: Interval; score: number; factors: Array<{ name: string; contribution: number }>; }

export function scoreSlots(slots: Interval[], ctx: ScoringContext): ScoredSlot[];
```
Score = Σ(weightᵢ × featureᵢ), weights drawn from `scheduling_preferences` (default neutral weights when none learned). `factors` exposes each weighted term so the UI can show "Ranked higher: matches your preferred morning window (+0.4); fewer same-day meetings (+0.2)". Output sorted by score desc; ties broken by earliest start.

**Testing**:
- `Unit: with preferred-morning weight, a 9am slot outranks a 4pm slot`
- `Unit: max-meetings-per-day weight penalizes slots on already-busy days`
- `Unit: factors sum equals total score`
- `Unit: zero learned preferences → neutral ranking == chronological order`
- `Unit: explanation factors are sorted by absolute contribution desc`

#### 5.3 — Online preference learner

**What**: Update preference weights from accumulated signals (lightweight logistic-regression-style online update).

**Design**:
```ts
// packages/core/src/scoring/learner.ts
export function updateWeights(current: WeightVector, signals: PreferenceSignal[]): WeightVector;
```
Treat `slot_chosen` as positive, `slot_skipped`/`booking_declined` as negative examples; perform gradient steps with L2 regularization and a learning rate, clamping weights to `[-1,1]`. A nightly `learn.preferences` BullMQ job recomputes per user from the trailing 90 days and upserts `scheduling_preferences` with `confidence = f(signalCount)` and `source='ai_inferred'`. User-stated preferences (`source='user_stated'`) are never overwritten.

**Testing**:
- `Unit: repeated slot_chosen at 9am increases preferred-morning weight monotonically`
- `Unit: conflicting signals → weights stay bounded in [-1,1]`
- `Unit: user_stated preference not modified by learner`
- `Unit: confidence rises with signal count, capped at 1.0`
- `Integration: nightly job upserts preferences from seeded signals`

#### 5.4 — Focus-block detection & protection

**What**: AI-suggested focus blocks treated as busy time, plus an endpoint to accept/reject suggestions.

**Design**:
- Detector analyses trailing 4 weeks of bookings to find recurring unbooked deep-work windows and proposes `focus_blocks` with `is_ai_generated=true`, `is_active=false` (pending acceptance).
- `gatherBusy` (Phase 4) includes active focus blocks as busy. Overriding one (booking over it) logs a `focus_block_overridden` signal feeding the learner.
- `GET/POST/PATCH /v1/focus-blocks` to list suggestions and accept/reject/edit.

**Testing**:
- `Unit: detector finds a consistently-free Tue/Thu morning → proposes focus block there`
- `Integration: accepted focus block excluded from public slots`
- `Integration: booking over an active focus block → allowed but logs focus_block_overridden`
- `Integration: reject suggestion → not persisted as active`

#### 5.5 — LLM explanation & NL override

**What**: Natural-language rendering of scoring factors and parsing of NL preference overrides (LLM optional; degrades gracefully).

**Design**:
- `packages/integrations/src/llm` wraps the Vercel AI SDK. `explainRanking(scoredSlot)` → one-sentence rationale. `parseOverride(text)` → structured preference change.
- Override prompt (system): *"You convert a user's natural-language scheduling preference into a JSON preference object with fields {preferenceType, dimension, value:-1..1}. Output only JSON."* Example user: *"Stop scheduling me before 10am"* → `{preferenceType:'avoided_time', dimension:'hour<10', value:-0.8}` stored with `source='user_stated'`.
- When `LLM_PROVIDER=none`, `explainRanking` falls back to a template string from `factors`, and NL override returns a validation error pointing users to the structured UI.

**Testing**:
- `Integration (mocked LLM): explainRanking returns sentence referencing top factor`
- `Integration (mocked LLM): parseOverride('no meetings before 10am') → expected preference JSON, stored as user_stated`
- `Unit: LLM_PROVIDER=none → explainRanking uses template fallback, no network call`
- `Integration (mocked LLM): malformed LLM JSON → 422, nothing stored`

#### 5.6 — AI-ranked public slots

**What**: Extend the slots endpoint to return AI-ranked order with explanations when requested.

**Design**: `GET /public/.../slots?ranked=true` returns slots ordered by `scoreSlots`, each with `score`, `factors`, and optional `explanation`. Default (`ranked=false`) stays chronological for invitee-facing pages; the host dashboard uses `ranked=true`.

**Testing**:
- `Integration: ranked=true reorders slots per learned weights and includes factors`
- `Integration: ranked=false → chronological, no scoring fields`
- `Integration: ranked slots still exclude busy/focus times (scoring never resurrects unavailable slots)`

---

## Phase 6: Notifications, iCalendar Delivery & iTIP/iMIP

### Purpose
Make bookings real to invitees: send RFC 6047 (iMIP) email invitations carrying RFC 5545 `.ics` with the correct RFC 5546 (iTIP) method, plus reminders, confirmations, reschedule/cancel notices, and follow-ups. This completes the MVP communication loop.

### Tasks

#### 6.1 — iCalendar & iTIP builders

**What**: Generate spec-compliant VEVENT/VFREEBUSY and iTIP REQUEST/REPLY/CANCEL objects.

**Design**:
```ts
// packages/core/src/ical/itip.ts
export function buildRequest(booking, host, invitees): string; // VCALENDAR METHOD:REQUEST
export function buildCancel(booking, host, invitees): string;  // METHOD:CANCEL, SEQUENCE++
export function buildReply(booking, invitee, partstat): string;// METHOD:REPLY
```
Uses `ical-generator`. UID is the booking `uid`; `SEQUENCE` increments on each modification (RFC 5546). ORGANIZER = host; ATTENDEE entries carry `PARTSTAT`. CI validates output against the iCalendar.org validator rules (offline ruleset).

**Testing**:
- `Unit: buildRequest → contains METHOD:REQUEST, UID, DTSTART/DTEND in UTC, ORGANIZER, ATTENDEE`
- `Unit: buildCancel increments SEQUENCE over the prior request`
- `Fixture: generated .ics validated against committed RFC 5545 ruleset → 0 errors`
- `Unit: timezone-aware event emits VTIMEZONE or UTC times consistently`

#### 6.2 — Email delivery via SMTP/iMIP

**What**: Send invitation/reschedule/cancel emails with the `.ics` attached as the correct MIME calendar part.

**Design**:
Nodemailer `icalEvent: { method: 'REQUEST', content }` produces `text/calendar; method=REQUEST` per RFC 6047 so Gmail/Outlook render an RSVP-able invite. Templates (subject/body) come from `workflows` rows or built-in defaults. A `notify.send` BullMQ job renders + sends; failures retry with backoff.

**Testing**:
- `Integration (mailpit container): confirmation email arrives with text/calendar; method=REQUEST part`
- `Integration: cancel email carries METHOD:CANCEL ics`
- `Unit: template variables ({{inviteeName}}, {{startLocal}}) interpolated correctly`
- `Integration: SMTP failure → job retried, marked failed after max attempts`

#### 6.3 — Reminders, follow-ups & workflows

**What**: Time-based reminders and post-meeting follow-ups driven by `workflows`.

**Design**:
- `workflows` table (from data-model-suggestion-1): `trigger` (`booking_created`|`reminder_before`|`follow_up_after`|`booking_cancelled`), `action_type` (`email`|`webhook`|`slack`), `offset_minutes`, templates.
- On booking creation, the API enqueues delayed BullMQ jobs at `start - offset` (reminder) and `end + offset` (follow-up). Cancellation removes pending delayed jobs by jobId `reminder:{bookingId}`.

**Testing**:
- `Integration: booking_created with reminder_before=15 → delayed job scheduled at start-15m (assert job delay)`
- `Integration: cancelling booking removes its pending reminder job`
- `Integration: follow_up_after job fires after end → follow-up email sent (fake timers)`
- `Integration: rescheduled booking reschedules its reminder to the new time`

---

## Phase 7: Web Dashboard & Embeddable Booking Widget

### Purpose
Provide the human interface: a Next.js dashboard for hosts (connect calendars, manage event types/availability, view bookings, see AI rankings and analytics) and a clean public booking page plus an embeddable widget mirroring Calendly/Cal.com. Can be built in parallel with Phases 4–6 once Phase 3's API contract is stable.

### Tasks

#### 7.1 — Host dashboard

**What**: Authenticated Next.js app for managing the account.

**Design**:
- Auth via Google/Microsoft OAuth login (reuses provider apps); session cookie shared with API origin or via API token.
- Pages: `/dashboard` (upcoming bookings), `/event-types`, `/availability`, `/calendars` (connect/disconnect), `/preferences` (view learned prefs + NL override box), `/analytics`. Server Components fetch from the API; mutations via Server Actions hitting the API.
- shadcn/ui components; AI rankings shown with the `factors` explanation chips from Phase 5.

**Testing**:
- `E2E (Playwright, mocked API): create event type via form → appears in list`
- `E2E: connect-calendar button initiates OAuth redirect (mocked)`
- `E2E: preferences page submits NL override → confirmation shown`
- `Component: booking list renders empty state when no bookings`

#### 7.2 — Public booking page

**What**: SSR booking page at `/{userSlug}/{eventTypeSlug}`.

**Design**: Server Component fetches slots (`ranked=false`), renders a month/day picker with timezone selector (auto-detected via `Intl`), collects answers, posts to `/public/.../book`, shows confirmation with add-to-calendar links. Handles 409 (slot taken) by refreshing slots.

**Testing**:
- `E2E: pick slot → fill form → submit → confirmation page with booking details`
- `E2E: timezone selector changes displayed slot times`
- `E2E: booking a now-taken slot → friendly "slot no longer available", slots refresh`
- `E2E: required custom question blocks submit until answered`

#### 7.3 — Embeddable widget

**What**: A framework-agnostic web component embedders drop onto any site.

**Design**: `packages/widget` builds to `embed.js` exposing `<meeting-scheduler user="..." event-type="..." />` (Shadow DOM, scoped styles). Inline and popup modes; posts to the same public API; emits `meeting:booked` CustomEvent for host-site analytics. Documented snippet in `docs/`.

**Testing**:
- `E2E (Playwright, static host page): widget loads in iframe-free shadow DOM, lists slots, books successfully`
- `Unit: widget emits 'meeting:booked' CustomEvent with booking payload`
- `Unit: popup mode opens/closes overlay`

---

## Phase 8: REST API Hardening — Auth, Webhooks, Teams & Routing

### Purpose
Round out the integration surface to match Calendly/Cal.com: API keys + OAuth 2.1 for third parties, signed outbound webhooks, round-robin/collective team scheduling, and routing forms. Depends on Phase 3 (API) and Phase 4 (multi-host availability).

### Tasks

#### 8.1 — API keys & OAuth 2.1

**What**: Machine auth for third-party API consumers.

**Design**:
- `api_keys` table (hashed key, prefix, scopes). `Authorization: Bearer sk_...`; key looked up by prefix, verified by hash. Scopes gate endpoints (`bookings:read`, `bookings:write`, `event-types:write`).
- OAuth 2.1 authorization-code + PKCE flow for multi-user integrations (`/oauth/authorize`, `/oauth/token`), issuing scoped access/refresh tokens.

**Testing**:
- `Integration: valid API key with bookings:read → GET /v1/bookings 200`
- `Integration: key lacking scope → 403`
- `Integration: revoked key → 401`
- `Integration: OAuth 2.1 code+PKCE exchange → access token; reused code → invalid_grant`

#### 8.2 — Outbound webhooks

**What**: Signed webhook subscriptions for booking lifecycle events (Calendly-parity events).

**Design**:
- `webhooks` table (url, secret_hash, events[]). Events: `booking.created`, `booking.cancelled`, `booking.rescheduled`, `routing_form.submitted`.
- `webhook.dispatch` BullMQ job POSTs JSON with header `X-Signature: sha256=hmac(secret, body)` and `X-Webhook-Id`; retries with exponential backoff up to 24h, then marks failed. Idempotency via stable event id.

**Testing**:
- `Integration: booking.created → webhook POST with valid HMAC signature (verify with secret)`
- `Integration: endpoint returns 500 → retried with backoff; 410 → disabled`
- `Integration: subscription filters → only subscribed event types delivered`
- `Unit: signature recomputation matches across identical payloads`

#### 8.3 — Team scheduling (round-robin & collective)

**What**: Multi-host event types.

**Design**:
- `teams`, `team_members` (with `weight`). For `round_robin`: pick the least-recently-assigned eligible member with capacity (query from data-model-suggestion-1) whose `gatherBusy` shows the slot free. For `collective`: slot must be free for *all* members (intersect availabilities, union busy).
- Booking endpoint resolves the host at commit time under the lock.

**Testing**:
- `Integration: round-robin distributes 4 sequential bookings across 2 members ~evenly by weight`
- `Integration: collective slots = intersection of all members' free time`
- `Integration: round-robin skips a member who became busy between slot-list and booking`

#### 8.4 — Routing forms

**What**: Conditional pre-booking forms that route invitees to event types/owners.

**Design**: `routing_forms` + fields + rules (JSONB conditions in the hybrid model). `evaluateRouting(answers, rules)` returns target event type or URL (first matching rule by `sort_order`). Submission fires `routing_form.submitted` webhook.

**Testing**:
- `Unit: answers matching rule 'department=Sales' → routes to sales event type`
- `Unit: no rule matches → default target`
- `Integration: submission → webhook fired, redirect to resolved booking page`

---

## Phase 9: MCP Server — Agentic Scheduling

### Purpose
Capture the first-mover MCP advantage from standards.md: expose the scheduler's capabilities to any MCP-compatible AI assistant (Claude Desktop, agents) over stdio and streamable HTTP. This is the foundation for the agent-to-agent scheduling frontier the research identifies as an unserved gap.

### Tasks

#### 9.1 — MCP server with scheduling tools

**What**: An MCP server exposing read/write scheduling tools, authenticated by API key.

**Design**:
```
apps/mcp — @modelcontextprotocol/sdk server
Tools:
  list_event_types()            → EventType[]
  get_freebusy(userSlug, from, to) → Interval[]
  find_slots(eventTypeSlug, from, to, ranked?) → ScoredSlot[]
  create_booking(eventTypeSlug, start, invitee) → Booking
  cancel_booking(uid, reason)   → ok
```
Each tool calls the same `packages/core` + repositories the REST API uses (shared logic, no duplication). Transport: stdio for desktop clients, streamable HTTP (`/mcp`) for remote agents, gated by API key + scopes from Phase 8.

**Testing**:
- `Integration (MCP test client): find_slots returns ranked slots identical to REST ranked=true`
- `Integration: create_booking via MCP creates a booking with the same exclusion guarantees`
- `Integration: tool call without valid API key → MCP error`
- `Integration: tool schemas validate against MCP spec (list_tools)`

#### 9.2 — Agent-to-agent negotiation primitive (experimental)

**What**: A `propose_meeting` / `respond_proposal` tool pair enabling two agents to converge on a slot across organisations.

**Design**: `propose_meeting(targetUserSlug, durationMin, windowPrefs)` returns up to N mutually-rankable candidate slots computed against *both* parties' free/busy (when the target is local) or via a returned VFREEBUSY the counterpart agent can evaluate. `respond_proposal(proposalId, chosenSlot)` commits. State held in a short-lived `proposals` table. Marked experimental; documented as the A2A frontier from standards.md.

**Testing**:
- `Integration: propose_meeting between two local users → candidate slots free for both`
- `Integration: respond_proposal commits one slot, books both calendars`
- `Integration: expired proposal → respond returns gone`

---

## Phase 10: Analytics, GDPR, Hardening & Self-Hosting

### Purpose
Production-readiness: team analytics (meeting load, focus-time ratio), GDPR data export/erasure, rate limiting, observability, and one-command self-hosting. Depends on bookings (Phase 3), calendars (Phase 4), and AI (Phase 5).

### Tasks

#### 10.1 — Analytics

**What**: Meeting-load and focus-time-ratio metrics per user/team.

**Design**: Read-only aggregation endpoints (`GET /v1/analytics/load?from&to`) computing meetings/day, total meeting hours, focus-time ratio (focus minutes ÷ working minutes), reschedule rate. Surface burnout signal when meeting density exceeds a configurable threshold (the research's burnout-monitoring gap).

**Testing**:
- `Integration: seeded bookings → correct meeting count and total hours`
- `Unit: focus-time ratio = focus minutes / available working minutes`
- `Integration: density above threshold → burnout flag true`

#### 10.2 — GDPR export & erasure

**What**: Self-service data export (JSON/ICS/CSV) and right-to-erasure.

**Design**: `GET /v1/me/export` streams a zip (profile JSON, bookings ICS, signals CSV). `DELETE /v1/me` revokes provider tokens, deletes user (cascades), and tombstones invitee PII per GDPR.

**Testing**:
- `Integration: export contains user's bookings as valid .ics and signals as CSV`
- `Integration: account deletion cascades and revokes tokens (assert provider revoke called)`
- `Integration: after erasure, invitee PII tombstoned/nulled`

#### 10.3 — Rate limiting, observability, security headers

**What**: Production safeguards.

**Design**: `@fastify/rate-limit` (Redis store) per API key/IP; structured JSON logging (pino) with request ids; OpenTelemetry traces; TLS 1.3 termination documented; security headers (HSTS, CSP for booking pages). Webhook and OAuth endpoints rate-limited separately.

**Testing**:
- `Integration: exceeding rate limit → 429 with Retry-After`
- `Integration: response carries security headers`
- `Unit: log redaction removes tokens/secrets`

#### 10.4 — Docker & self-hosting

**What**: One-command self-host via Docker Compose.

**Design**: Multi-stage `Dockerfile` building one image; `$SERVICE` env selects `api|worker|mcp|web`. `docker-compose.yml` wires Postgres 16, Redis 7, and the four services; an `init` step runs migrations. `docs/self-hosting.md` documents env setup, OAuth app registration, and backup.

**Testing**:
- `E2E (compose in CI): docker compose up → /healthz green, book a meeting end-to-end against the stack`
- `Integration: migrations run idempotently on container start`
- `Smoke: image runs each $SERVICE without crashing`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, types)        ─── required by everything
    │
Phase 2: Availability Engine (pure logic)        ─── requires Phase 1
    │
Phase 3: REST API & Booking Flow (MVP core)      ─── requires Phase 1, 2
    │
    ├── Phase 4: Calendar Integration & Sync      ─── requires Phase 3
    │       │
    │       ├── Phase 5: AI Preference Engine      ─── requires Phase 3, 4
    │       │       │
    │       │       └── Phase 6: Notifications/iTIP/iMIP  ─── requires Phase 3 (uses 5 for content)
    │       │
    │       └── Phase 8: API Hardening/Teams/Routing ─── requires Phase 3, 4
    │
    └── Phase 7: Web Dashboard & Widget           ─── requires Phase 3 contract; parallel with 4–6
            │
Phase 9: MCP Server                               ─── requires Phase 3, 4, 5
    │
Phase 10: Analytics, GDPR, Hardening, Self-host   ─── requires Phase 3, 4, 5
```

**Parallelism opportunities:**
- **Phase 7 (Web/Widget)** can be built concurrently with Phases 4–6 once the Phase 3 API contract (OpenAPI) is frozen.
- **Phase 8 (API hardening/teams/routing)** can proceed in parallel with **Phase 5 (AI)** after Phase 4, since they touch different subsystems.
- Within Phase 4, the three providers (4.2 Google/MS, 4.3 CalDAV) can be implemented in parallel once the 4.1 interface exists.

**Scope estimate:** Large (10 phases, ~38 tasks, four deployables).

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. **All tasks implemented** as specified in the phase's Design sections.
2. **All unit tests pass** (`pnpm test`).
3. **All integration tests pass** against real Postgres + Redis via Testcontainers (`pnpm test:int`).
4. **E2E tests pass** where the phase introduces user-facing flows (`pnpm test:e2e`).
5. **Linting and formatting pass** (`pnpm lint` / Biome — zero errors).
6. **Type checking passes** (`pnpm typecheck` — `tsc --noEmit`, strict).
7. **Docker image builds** and the affected service(s) start cleanly.
8. **Feature works end-to-end** — demonstrated by at least one integration or E2E test exercising the full path.
9. **New config options documented** in `.env.example` and `docs/self-hosting.md`.
10. **New/changed API endpoints appear in `docs/openapi.json`**, and the committed spec matches generated output (CI diff is clean).
11. **Database migrations created** via `drizzle-kit`, additive (no destructive changes to prior phases), and run idempotently.
12. **Generated `.ics` output validates** against the RFC 5545 ruleset (for phases touching iCalendar).
13. **Standards honored** where the phase touches them (RFC 5545/5546/6047/4791/6638/6578, OAuth 2.0/2.1 + PKCE, OpenAPI 3.1, MCP, GDPR), with a note in the PR describing which.
