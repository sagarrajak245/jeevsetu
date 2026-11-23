# JeevSetu — Build Plan (48-hour sprint, phase per session)

> A person should never have to watch an injured animal suffer simply because they don't know whom to contact.

**Architecture:** modular monolith. Backend = FastAPI with per-module layering (api → service → repository → model). Frontend = Next.js App Router with feature-based separation.
**Rule for every session:** start one phase, finish it with green tests and one commit, update the "Progress" table at the bottom, stop.

---

## 0. Ground rules (apply to every phase)

1. **AI never blocks a report.** Every AI call has a timeout and a fallback; if all providers fail, the report is still saved and routed.
2. **Honest status.** No status is shown as "accepted"/"dispatched" unless an organization explicitly set it.
3. **Listed ≠ available.** Organizations carry `verification_status` and `availability` separately.
4. **Privacy.** Reporter PII is never returned by public endpoints; public coordinates are rounded/blurred (wildlife always).
5. **Module boundaries.** A module may import another module's **service** (public interface) only — never its repository or models. Enforced by `import-linter` in CI.
6. **TDD.** Each feature starts with 1–2 failing tests (service level), then code. Each phase ends with `pytest`, `ruff`, `mypy`, `pnpm lint`, `pnpm typecheck`, `pnpm test` green.
7. **No hard-coded values.** Limits, timeouts, model names, radii → `core/config` (env) or `constants.py`.

---

## 1. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | Next.js (App Router) + TypeScript + Tailwind + shadcn/ui | SSR, PWA, fast UI |
| FE language | **TypeScript (strict)** everywhere, no `any` | |
| FE data | **TanStack Query** (server state, caching, mutations, optimistic updates) + **TanStack Table** (admin/dashboard tables) + Zod + React Hook Form | typed fetching/forms |
| FE auth | **Supabase Auth** via `@supabase/ssr` (cookie session, middleware refresh) | |
| FE extras | Leaflet + OpenStreetMap, next-intl (later), Serwist (PWA, later) | free maps |
| Backend | FastAPI + Pydantic v2 | typed, async |
| ORM | SQLAlchemy 2 (async) + asyncpg + GeoAlchemy2 + Alembic | PostGIS support |
| DB | **Supabase Postgres + PostGIS** — local via **Supabase CLI** (`supabase start`), hosted Supabase in prod | geo queries, same stack dev/prod |
| Auth | **Supabase Auth** (email + password / magic link; Google later). FastAPI verifies the Supabase JWT (JWKS) and loads role from `profiles` | no custom auth code |
| Storage | **Supabase Storage** — private bucket `report-media`, public bucket `org-assets`; backend issues signed upload/read URLs | secure uploads |
| Redis | **Redis** (Docker locally, Upstash/Render Redis in prod) — ARQ job queue, rate limiting, caching, AI circuit-breaker state, idempotency keys | see "Redis usage" below |
| Jobs | ARQ (on Redis) | notifications, escalation timers, AI enrichment, retention purge |
| Email | Resend (driver behind a `notifications` interface) | free tier |
| AI vision (local) | Ultralytics YOLO (nano, COCO pretrained) exported to **ONNX**, run with `onnxruntime` CPU | species detection without GPU/torch in prod |
| AI LLM / multimodal | **Gemini (primary) → Groq (fallback)**, automatic failover | injury hints, text matching, first-aid card selection |
| NGO no-login links | Signed, expiring **case tokens** (itsdangerous/HMAC) — separate from Supabase sessions | NGOs act without an account |
| Tooling | uv, ruff, mypy, pytest, import-linter · pnpm, eslint, vitest, Playwright (smoke) | |
| Hosting (later) | Vercel (web) + Render/Fly (api + worker) + Supabase | ~₹0–500/month |

> ⚠ **License note:** Ultralytics YOLO is AGPL-3.0. Fine if this repo is open source (recommended for a social-impact project). If it ever must stay closed, revisit (e.g. an Apache-licensed detector).
> ⚠ **Model IDs** (Gemini / Groq) change often — they live only in config and are verified at the start of Phase 3.

---

## 2. Repository layout

```
air-vets/
├── api/                          # FastAPI backend (modular monolith)
│   ├── app/
│   │   ├── main.py               # app factory: middleware, routers, exception handlers
│   │   ├── core/
│   │   │   ├── config/
│   │   │   │   ├── settings.py   # pydantic-settings, one Settings class, grouped sub-settings
│   │   │   │   └── __init__.py   # get_settings() (cached)
│   │   │   ├── constants.py      # app-wide constants (pagination limits, API prefix, enums shared by all)
│   │   │   ├── logging.py        # structured JSON logging + request_id
│   │   │   ├── security.py       # Supabase JWT verification (JWKS, cached), case-token signing
│   │   │   ├── supabase.py       # server-side Supabase client (service role — backend only, never sent to browser)
│   │   │   └── middleware.py     # request id, timing, CORS
│   │   ├── shared/               # cross-module building blocks (no business logic)
│   │   │   ├── exceptions/
│   │   │   │   ├── base.py       # AppException(code, message, status, details)
│   │   │   │   ├── http.py       # NotFound, Conflict, Validation, Forbidden, Unauthorized, RateLimited
│   │   │   │   └── handlers.py   # maps exceptions → standard error envelope
│   │   │   ├── pagination/
│   │   │   │   ├── params.py     # PageParams(page, size) dependency, max size from constants
│   │   │   │   ├── schemas.py    # Page[T] {items, total, page, size, pages}
│   │   │   │   └── paginate.py   # paginate(session, select_stmt, params) helper
│   │   │   ├── db/
│   │   │   │   ├── base.py       # DeclarativeBase, naming conventions
│   │   │   │   ├── mixins.py     # UUID pk, timestamps, soft delete
│   │   │   │   ├── session.py    # async engine, get_session dependency, unit of work
│   │   │   │   └── repository.py # BaseRepository[Model] (get, list, create, update, delete)
│   │   │   ├── schemas/          # ApiResponse, ErrorEnvelope, common value objects (GeoPoint)
│   │   │   ├── geo/              # distance helpers, coordinate blurring, PIN-code lookup
│   │   │   ├── storage/          # StorageDriver protocol + SupabaseStorage driver (+ in-memory fake for tests)
│   │   │   ├── notifications/    # Notifier protocol + email (Resend) / console drivers
│   │   │   ├── cache/            # redis client, cache decorator, rate limiter, idempotency, distributed lock
│   │   │   └── utils/            # time, ids, text
│   │   ├── ai/                   # everything AI, isolated from business modules
│   │   │   ├── constants.py      # timeouts, confidence thresholds, label maps
│   │   │   ├── exceptions.py     # AIProviderError, AllProvidersFailed, ModelNotLoaded
│   │   │   ├── providers/
│   │   │   │   ├── base.py       # LLMProvider protocol: generate_json(), analyze_image()
│   │   │   │   ├── gemini.py
│   │   │   │   ├── groq.py
│   │   │   │   └── router.py     # FailoverLLM: Gemini → Groq, circuit breaker, retries, timeouts
│   │   │   ├── vision/
│   │   │   │   ├── yolo_detector.py  # ONNX model load (once), preprocess, NMS, COCO→species map
│   │   │   │   └── weights/      # *.onnx (git-lfs or downloaded at build)
│   │   │   ├── prompts/          # versioned prompt templates (injury_hint_v1.md, match_v1.md)
│   │   │   ├── schemas.py        # SpeciesPrediction, InjuryHints, MatchIntent (structured outputs)
│   │   │   └── service.py        # AIService facade used by modules (classify, injury_hints, parse_intent)
│   │   ├── modules/
│   │   │   ├── auth/             # profiles + roles (admin, org_member, volunteer) on top of Supabase Auth users; current_user / require_role deps
│   │   │   ├── organizations/    # directory, capabilities, coverage, verification, availability
│   │   │   ├── reports/          # citizen incident reports, media, location
│   │   │   ├── matching/         # ranking organizations for a report
│   │   │   ├── cases/            # rescue requests, status state machine, timeline, case tokens
│   │   │   ├── notifications/    # outbound messages, delivery log, retries (uses shared/notifications)
│   │   │   ├── firstaid/         # species × condition guidance cards
│   │   │   ├── volunteers/       # profiles, skills, area, alerts
│   │   │   ├── admin/            # moderation, verification queue, audit log
│   │   │   └── impact/           # public stats
│   │   │   # each module has the SAME shape:
│   │   │   #   api/routes.py      ← controller (HTTP only: parse, call service, return schema)
│   │   │   #   api/dependencies.py
│   │   │   #   schemas.py         ← request/response DTOs (Pydantic)
│   │   │   #   models.py          ← SQLAlchemy models
│   │   │   #   repository.py      ← DB access only (extends BaseRepository)
│   │   │   #   service.py         ← business logic; the module's public interface
│   │   │   #   constants.py       ← module enums/limits
│   │   │   #   exceptions.py      ← module errors (extend shared AppException)
│   │   └── workers/
│   │       ├── main.py           # ARQ WorkerSettings
│   │       └── tasks/            # send_notification, escalate_case, run_ai_enrichment
│   ├── alembic/                  # migrations
│   ├── scripts/                  # seed_organizations.py, load_pincodes.py, export_yolo_onnx.py
│   ├── tests/
│   │   ├── unit/<module>/        # service tests with fake repositories/providers
│   │   ├── integration/          # API + real Postgres/PostGIS (docker)
│   │   └── conftest.py
│   ├── pyproject.toml            # uv, ruff, mypy, pytest, import-linter contracts
│   └── Dockerfile
│
├── web/                          # Next.js frontend
│   ├── src/
│   │   ├── app/                  # ROUTES ONLY — thin pages that compose features
│   │   │   ├── (public)/         # /, /report, /report/[id]/results, /case/[ref], /orgs/[slug], /impact
│   │   │   ├── (org)/dashboard/  # NGO dashboard
│   │   │   ├── (admin)/admin/    # admin panel
│   │   │   ├── r/[token]/        # no-login NGO case action page
│   │   │   └── layout.tsx
│   │   ├── features/             # one folder per domain, mirrors backend modules
│   │   │   ├── report/           # components/, hooks/, api/, schemas/, types.ts, constants.ts
│   │   │   ├── matching/
│   │   │   ├── cases/
│   │   │   ├── organizations/
│   │   │   ├── firstaid/
│   │   │   ├── volunteers/
│   │   │   ├── auth/
│   │   │   ├── admin/
│   │   │   └── impact/
│   │   ├── shared/
│   │   │   ├── components/ui/    # shadcn primitives
│   │   │   ├── components/       # Map, ImageCapture, Pagination, EmptyState, ErrorBoundary, StatusBadge
│   │   │   ├── lib/api-client.ts # fetch wrapper: base URL, attaches Supabase access token, error envelope → ApiError
│   │   │   ├── lib/query-client.ts   # TanStack QueryClient defaults (retry, staleTime)
│   │   │   ├── lib/query-keys.ts     # central query-key factory per feature
│   │   │   ├── lib/supabase/         # client.ts (browser), server.ts (RSC/route handlers), middleware.ts (session refresh)
│   │   │   ├── hooks/            # useGeolocation, usePagination, useDebounce
│   │   │   ├── config/env.ts     # Zod-validated env
│   │   │   ├── constants/        # routes.ts, species.ts, statuses.ts, pagination.ts
│   │   │   ├── types/            # Page<T>, ApiError — mirror backend shared schemas
│   │   │   └── utils/
│   │   └── styles/
│   ├── tests/                    # vitest unit + playwright smoke
│   └── package.json
│
├── supabase/                     # Supabase CLI project: config.toml, storage buckets, auth settings, seed
├── docker-compose.yml            # redis, api, worker (Supabase local stack runs via `supabase start`; web via pnpm dev)
├── .env.example
├── Makefile                      # make dev / test / lint / migrate / seed
├── .github/workflows/ci.yml
└── docs/
    ├── BUILD_PLAN.md             # this file
    └── adr/                      # short architecture decision records
```

### Layer rules (backend)
- `api/routes.py` → only calls its own `service.py`. No SQL, no business rules.
- `service.py` → business logic, transactions, calls own repository + other modules' **services** + `ai.service` + `shared/*`.
- `repository.py` → only SQLAlchemy queries for its own models. Returns models, never DTOs.
- `schemas.py` → DTOs; conversion model→DTO happens in the service or route via `model_validate`.
- `shared/` and `core/` never import from `modules/` or `ai/`. `ai/` never imports from `modules/`.

### Standard API contracts
- Prefix: `/api/v1`.
- Success: resource or `Page[T]` → `{ "items": [...], "total": 120, "page": 1, "size": 20, "pages": 6 }`.
- Query: `?page=1&size=20` (max size in `core/constants.py`, e.g. 100). Same everywhere, via `PageParams` dependency.
- Error envelope (all errors, via `shared/exceptions/handlers.py`):
  `{ "error": { "code": "ORG_NOT_FOUND", "message": "...", "details": {...}, "request_id": "..." } }`
- Frontend `api-client.ts` converts this envelope into a typed `ApiError`.

### Supabase integration
- **Auth flow:** browser signs in with Supabase (`@supabase/ssr`) → `api-client.ts` sends `Authorization: Bearer <access_token>` → FastAPI `core/security.py` verifies it against Supabase JWKS (cached in Redis) → `modules/auth` loads `profiles.role` → `require_role("admin")` etc.
- **Citizens never need an account.** Only admins, NGO members and volunteers sign in.
- **Data access goes through FastAPI only.** Every table has **RLS enabled with no anon policies**, so the public anon key cannot read data via PostgREST. The service-role key lives only in the backend env.
- **Uploads:** client asks API for a signed upload URL → uploads directly to Supabase Storage (`report-media`, private) → API confirms, worker strips EXIF + makes thumbnail. Reads use short-lived signed URLs.
- **Migrations:** Alembic owns the app schema; `supabase/` owns auth/storage config and buckets.

### Redis usage
| Use | Where |
|---|---|
| ARQ job queue (notifications, escalation, AI enrichment, EXIF strip, retention purge) | `workers/` |
| Rate limiting (report creation per IP/device, login, AI calls) | `shared/cache/rate_limit.py` |
| Caching (PIN-code centroids, org lists, matching results for 60 s, JWKS) | `shared/cache/cache.py` |
| AI circuit-breaker state shared across API instances | `ai/providers/router.py` |
| Idempotency keys (duplicate report submission on flaky networks) | `shared/cache/idempotency.py` |
| Distributed lock (escalation job runs once per case) | `shared/cache/lock.py` |

### AI failover design
```
AIService.injury_hints(image)
   └─ FailoverLLM.analyze_image(prompt, image, schema)
        ├─ GeminiProvider   (primary)  timeout 8s, 1 retry
        │     fail / timeout / 429 / 5xx / invalid JSON → mark failure (circuit breaker opens after N fails for M sec)
        └─ GroqProvider     (fallback) timeout 8s
              fail → raise AllProvidersFailed → caller degrades gracefully (no hints)
   Every call logs: provider used, latency, fallback_used, prompt_version.
YOLO runs locally (no network), ~50–150 ms on CPU, result stored on the report as a *suggestion*.
```
Config: `AI_PRIMARY=gemini`, `AI_FALLBACK=groq`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `GEMINI_MODEL`, `GROQ_VISION_MODEL`, `AI_TIMEOUT_SECONDS`, `AI_ENABLED=true`.

---

## 3. Data model (core tables)

| Table | Key columns |
|---|---|
| `profiles` | id (= `auth.users.id` from Supabase), full_name, phone, role (`admin`/`org_member`/`volunteer`), is_active |
| `organizations` | id, slug, name, type (`ngo`/`vet_clinic`/`hospital`/`shelter`/`bird_rescue`/`wildlife_centre`/`forest_dept`/`snake_rescuer`), address, pincode, `location geography(Point)`, `coverage geography(Polygon)` nullable, coverage_radius_km, phone, whatsapp, email, website, is_24x7, verification_status (`unverified`/`pending`/`verified`/`suspended`), last_verified_at, availability (`unknown`/`accepting`/`full`/`closed`), donation_url |
| `organization_species` | org_id, species (`dog`/`cat`/`bird`/`cattle`/`wildlife`/`reptile`/`other`) |
| `organization_services` | org_id, service (`rescue`/`treatment`/`ambulance`/`shelter`/`advice`) |
| `organization_hours` | org_id, weekday, opens, closes |
| `organization_members` | org_id, user_id, role |
| `pincodes` | pincode, district, state, `centroid geography(Point)` |
| `reports` | id, public_ref (e.g. `JS-7F3K2`), species, species_ai_suggestion, ai_confidence, conditions (text[]), description, `location geography(Point)`, location_source (`gps`/`pin`/`manual`), location_accuracy_m, pincode, landmark, reporter_name, reporter_phone, reporter_email, injury_hints (jsonb), status, created_at |
| `report_media` | id, report_id, storage_key, mime, width, height, exif_stripped |
| `cases` | id, report_id, organization_id, channel (`whatsapp`/`email`/`in_app`), status, accepted_at, resolved_at |
| `case_events` | id, case_id, from_status, to_status, actor_type (`system`/`reporter`/`org`/`admin`), actor_id, note, created_at |
| `notifications` | id, case_id, channel, recipient, status (`queued`/`sent`/`failed`), attempts, last_error |
| `firstaid_cards` | id, species, condition, title, steps (jsonb), donts (jsonb), reviewed_by, language |
| `volunteers` | user_id, skills[], `home_area geography(Point)`, radius_km, available |
| `audit_logs` | actor, action, entity, entity_id, diff (jsonb), created_at |

**Case status machine** (in `modules/cases/constants.py` + `service.py`, invalid transitions raise `InvalidStatusTransition`):
`submitted → seen → accepted → en_route → rescued → under_treatment → resolved`
plus `submitted|seen → rejected | needs_info | unable_to_rescue`, `needs_info → seen`.

**Matching score** (in `modules/matching/service.py`, weights in `constants.py`):
`hard filter: species supported AND (inside coverage OR within radius)` →
`score = w_species_specialist + w_service_fit + w_distance_decay + w_open_now + w_verified + w_availability + w_response_rate` → top 5, with a reason string per result ("Bird specialist · 2.1 km · Open now · Verified").

---

## 4. Phases (one session each)

Estimates assume focused work; the 48 h target covers Phases 0–5 as must-have; 6–7 are stretch.

### Phase 0 — Foundation (~5 h)
**Goal:** both apps run, shared building blocks exist, CI is green.
- Backend: `uv` project, `core/config` (Settings), `core/constants.py`, logging + request-id middleware, `shared/exceptions` (+ handlers + envelope), `shared/pagination`, `shared/db` (base, mixins, session, BaseRepository), `shared/cache` (Redis client + rate limiter), Alembic init with PostGIS extension, `/api/v1/health` (db + redis + supabase check).
- import-linter contracts for the layer rules; ruff + mypy config.
- Frontend: Next.js + TypeScript (strict) + Tailwind + shadcn init, `shared/` skeleton, `api-client.ts` with `ApiError`, Zod env, TanStack Query provider + Devtools + query-key factory, Supabase browser/server clients + session middleware, base layout (mobile-first), `Page<T>` type.
- Infra: Supabase CLI init (`supabase/`, local stack with PostGIS enabled, buckets `report-media` + `org-assets`), `docker-compose.yml` (redis, api, worker), `.env.example`, Makefile, GitHub Actions CI (lint, typecheck, test for both).
- `git init`, `main` + `develop` branches.
- **Tests:** exception handler returns envelope; pagination helper (page math, max size); rate limiter; health endpoint.
- **Done when:** `make dev` boots everything, `make test` green, CI green.

### Phase 1 — Organizations & directory (~6 h)
**Goal:** a curated, searchable directory managed by an admin.
- `modules/auth`: Supabase JWT verification dependency, `profiles` table + trigger on `auth.users` insert, `require_role` dependency, seed admin script. RLS enabled (deny anon) on all app tables.
- `modules/organizations`: models (+ species, services, hours), repository, service, CRUD routes (admin), public list/detail (paginated, filters: type, species, pincode), verification + availability fields.
- `scripts/load_pincodes.py` (India Post dataset, Maharashtra first), `scripts/seed_organizations.py` (~30 Mumbai orgs from public info, all `unverified` until phone-checked).
- `modules/admin`: audit log on create/update.
- Frontend: `features/auth` (Supabase sign-in, protected route middleware), `features/organizations` (admin TanStack Table with shared Pagination, create/edit form with map pin picker, TanStack Query mutations + cache invalidation), public org profile page.
- Redis cache on public org list + PIN lookups.
- **Tests:** org service CRUD + validation; pagination on list; role guard (valid / expired / wrong-role token); open-now calculation.
- **Done when:** admin can add/edit orgs with location; public directory pages render.

### Phase 2 — Citizen report, location & matching (~6 h)
**Goal:** the core citizen flow works end to end without AI.
- `shared/storage`: Supabase Storage signed upload URL, image validation (type/size), EXIF strip + thumbnail (ARQ job), signed read URLs.
- `modules/reports`: create report (no login, Redis rate limit + idempotency key), media attach, public_ref generation, PII separation (public vs private schema).
- `shared/geo`: PIN → centroid, coordinate blurring for public view.
- `modules/matching`: PostGIS hard filter + weighted score + reason strings, top 5.
- `modules/firstaid`: seed static cards (dog/cat/bird/wildlife/snake × bleeding/can't-walk/trapped/general), lookup by species + conditions.
- Frontend `features/report`: stepper — photo (camera + upload, client-side compression ~200 KB) → species chips → condition checkboxes → location (GPS with accuracy, draggable Leaflet pin, PIN fallback) → submit. Results page: 5 org cards (distance, reason, Call / Directions / WhatsApp / Email buttons, verified/listed badge) + first-aid card.
- **Tests:** matching ranks bird specialist over dog shelter for `bird`; excludes orgs outside coverage; PIN fallback; report create rejects bad image; public schema hides PII.
- **Done when:** report a pigeon in Mumbai on a phone → see 5 sensible orgs + first-aid.

### Phase 3 — AI module: YOLO + Gemini/Groq failover (~6 h)
**Goal:** helpful AI suggestions that never block the flow.
- Verify current Gemini and Groq model IDs (vision-capable) and SDKs; record in `.env.example`.
- `ai/vision/yolo_detector.py`: `scripts/export_yolo_onnx.py` (YOLO nano COCO → ONNX), load once at startup, map COCO classes → our species (`dog`, `cat`, `bird`, `cow/horse/sheep → cattle/other`, `elephant/bear/zebra/giraffe → wildlife`), confidence threshold from `ai/constants.py`.
- `ai/providers`: `LLMProvider` protocol, `GeminiProvider`, `GroqProvider`, `FailoverLLM` (timeouts, retry, circuit breaker, structured JSON output validated with Pydantic, logs provider/fallback/latency).
- `ai/prompts/injury_hint_v1.md`: conservative visible-sign hints (`possible_bleeding`, `possible_wound`, `lying_down`, `possible_limb_injury`) + `urgency_hint` — explicitly "not a diagnosis".
- `ai/service.py` facade: `suggest_species(image)`, `injury_hints(image)`, `parse_description(text)` (species + conditions from free text, feeds matching).
- Integration: on photo upload → YOLO runs synchronously (fast) to pre-select species chip; injury hints run as ARQ background task and attach to report; matching uses parsed description when species is "other"/unknown.
- Frontend: "Looks like a **dog**? ✓ / change" chip; injury hints shown as "Possible signs (AI, not a diagnosis)"; graceful when absent.
- **Tests:** FailoverLLM uses Groq when Gemini raises/times out; raises `AllProvidersFailed` when both fail; report still created when AI disabled; YOLO class mapping; invalid JSON from provider → fallback.
- **Done when:** upload dog photo → species pre-selected; kill Gemini key → Groq answers; kill both → flow still works.

### Phase 4 — Rescue cases, notifications & NGO no-login link (~6 h)
**Goal:** a citizen can send a structured rescue request and track honest status.
- `modules/cases`: create case from report + org, status state machine, `case_events` timeline, signed expiring case token (`core/security.py`).
- `modules/notifications`: email via Resend driver (console driver in dev), WhatsApp share deep-link builder (pre-filled structured message + case link), delivery log, ARQ retry with backoff.
- Escalation worker: no `seen/accepted` within N min (config) → mark case "no response yet" and suggest next org to the reporter.
- Public case page `/case/[ref]`: timeline with attributed events ("Sent by you via WhatsApp 22:14", "Accepted by RESQ 22:31").
- NGO action page `/r/[token]`: view full report (photo, exact location, reporter contact), buttons Accept / Reject / Need info / En route / Rescued / Resolved + note.
- **Tests:** invalid transition rejected; expired/forged token rejected; email failure retried and logged; escalation fires after timeout; public case page never leaks reporter phone.
- **Done when:** full loop — report → send to NGO (email + WhatsApp link) → NGO accepts via link → citizen sees it.

### Phase 5 — NGO dashboard & org onboarding (~6 h)
**Goal:** organizations can self-register and manage cases.
- Org self-registration (pending verification), member invites, `org_member` role.
- Dashboard: incoming queue (paginated, filter by status), case detail, map of active cases, availability toggle, service-area editor (radius / draw polygon), basic stats (response time, accepted, resolved).
- Admin: verification queue (approve/reject with notes), suspend org, moderation of flagged reports.
- Report abuse/spam flag from NGO side; NGO member invites via Supabase Auth invite emails.
- Dashboard live-ish updates via TanStack Query polling (Supabase Realtime later if needed).
- **Tests:** org member sees only own org's cases; unverified org not shown "verified"; rate limit returns 429 envelope.
- **Done when:** an NGO registers → admin verifies → NGO handles cases from the dashboard.

### Phase 6 — Volunteers, push & impact (stretch, ~5 h)
- `modules/volunteers`: sign-up, skills, home area + radius, availability.
- Web push (VAPID) for volunteers near unanswered cases; nearby active case feed (radius + species filters, blurred locations).
- Duplicate report hint (same species, < 200 m, < 2 h) shown to reporter and NGO.
- `modules/impact`: public stats page (reports, rescues, median response time, active verified orgs).
- **Tests:** volunteer alert only within radius; duplicate detection window; impact aggregates.

### Phase 7 — Hardening & launch (stretch, ~5 h)
- PWA (Serwist): installable, offline queue for report submission.
- i18n: English + Marathi + Hindi for the citizen flow (next-intl).
- PII retention job (purge reporter contact after N days), security review, CORS/headers, upload limits.
- Deploy: hosted Supabase (DB + Auth + Storage), Render/Fly (api + worker), Upstash/Render Redis, Vercel (web); Sentry/free logging.
- Playwright smoke: report → match → case → NGO accept.
- **Done when:** deployed URL works on a real phone over 3G throttling.

---

## 5. Session protocol

Start of each session, paste:
> "Continue JeevSetu. Read `docs/BUILD_PLAN.md`. Start **Phase N**. Follow the layer rules and TDD."

End of each session:
1. All checks green, one commit on a `feature/phase-N-<name>` branch, PR to `develop`.
2. Update the Progress table below (status + notes + anything deferred).

## 6. Progress

| Phase | Status | Notes |
|---|---|---|
| 0 Foundation | TODO | |
| 1 Organizations & directory | TODO | |
| 2 Report, location & matching | TODO | |
| 3 AI (YOLO + Gemini/Groq) | TODO | |
| 4 Cases, notifications, NGO link | TODO | |
| 5 NGO dashboard & onboarding | TODO | |
| 6 Volunteers, push, impact | TODO | stretch |
| 7 Hardening & launch | TODO | stretch |

## 7. Decisions taken by default (change any time)
- Supabase Auth + Supabase Storage + Supabase Postgres (user decision 2026-11-23); local dev via Supabase CLI, Redis via Docker.
- All data access through FastAPI; RLS on with no anon policies.
- Offset pagination (`page`/`size`) everywhere; cursor pagination only if the volunteer feed needs it.
- YOLO via ONNX Runtime on CPU (no torch in production image).
- Repo open source (keeps YOLO AGPL licence simple) — confirm before first public push.
