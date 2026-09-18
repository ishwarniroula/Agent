AGENT.md — NagarDrishti AI · Master Execution Prompt (Final, One-Shot)
Autonomous build directive for a coding agent (Claude Code or equivalent). Read fully once, then execute end-to-end without pausing for approval. This is the single source of truth — do not ask to update it later.
---
0. Role & Operating Rules
You are an autonomous full-stack delivery team: architect, backend, mobile, AI/ML, DevOps, security, QA.
Build phase-by-phase (Section 15). Finish each phase's deliverable fully before the next.
Do not narrate intent. Build, then end each phase with a 1–3 line status + file list.
Never re-paste unchanged files; use targeted edits after first version.
Batch related files in one pass (all models together, all screens of a flow together).
Only ask a question if a choice is expensive to reverse; otherwise pick the default stated here and note the assumption in one line.
NO demo/seed/fake data anywhere. Every record is created through the real app flows and real APIs. Provide only an empty-state-friendly first-run experience and an optional `scripts/create-first-admin.ts` bootstrap (one admin user via CLI prompt — not seeded content).
NO emojis anywhere in UI or code comments. All iconography via icon libraries (Section 6).
Governance rule (applies everywhere): AI output is always advisory ("AI-suggested"), never an automatic final decision. Officer/admin verification gates every status change from VERIFICATION_REQUIRED onward.
---
1. Product
NagarDrishti AI — "See the Problem. Understand the City. Take Action."
AI-assisted civic issue reporting, mapping, prioritization, and municipal decision support for Nepal. AI recommends; officers decide.
Credit (About screen footer): Prepared by Shree Saraswati Secondary School, Damak-9, Jhapa, Nepal.
Platform: React Native mobile app only (Android + iOS) + backend + AI engine. No website. All four roles (Citizen, Municipal Officer, Field Staff, Administrator) live inside the one app with role-based navigation.
Flow: Citizen → Report Submission → API → AI Analysis Engine → DB → Live Map → Officer Dashboard → Verification → Field Assignment → Progress Tracking → Resolution → Citizen notified + can rate.
---
2. Tech Stack (pinned to latest stable at build time — verify versions with the registry before scaffolding; do not substitute without a one-line reason)
Mobile
Expo SDK (latest stable) + React Native (New Architecture enabled: Fabric + TurboModules + bridgeless), TypeScript strict.
Expo Router (typed file-based routing) · expo-dev-client for native modules.
State/data: TanStack Query v5 (server state) + Zustand (client state) + MMKV (fast encrypted storage).
Maps: react-native-maps (Google provider Android, Apple Maps iOS) + OSM tile fallback; Supercluster for marker clustering.
Animation/graphics: Reanimated 3 (UI-thread worklets only), Moti, react-native-gesture-handler, @shopify/react-native-skia (charts, shaders, hero visuals — this is the RN-native replacement for the web-only GSAP/Lenis/Shery.js stack), react-three-fiber + expo-gl (one lightweight 3D hero scene on the auth/landing screen only), Lottie for micro-illustrations, expo-haptics.
Media: expo-image (with blurhash placeholders), expo-image-picker, expo-camera, on-device compression before upload, expo-location (foreground; background only for field staff navigation with explicit consent).
Icons: lucide-react-native primary; phosphor-react-native for gaps. Never emoji.
Fonts (expo-font/@expo-google-fonts): Inter (UI), Space Grotesk (display/headlines), Noto Sans Devanagari (Nepali), JetBrains Mono (IDs/coordinates).
Forms/validation: react-hook-form + Zod (shared schemas with backend).
i18n: i18next + react-i18next (EN + NE), `nepali-date-converter` for Bikram Sambat display.
Backend (self-hosted — the Firebase alternative)
Node.js LTS + Fastify (typed, fast) + Zod request/response validation.
PostgreSQL 16 + PostGIS via Prisma ORM (raw SQL for PostGIS queries where Prisma lacks support).
Supabase self-hosted OR plain Postgres + MinIO — default: plain Postgres + MinIO (S3-compatible) for image/object storage with presigned upload URLs. Redis for cache, rate-limit counters, queues.
BullMQ for background jobs (AI analysis, notifications, hotspot recomputation, image processing with sharp).
Realtime: Socket.io (map updates, status changes, notification push channel).
Push: Expo Push Notifications service (free, no Firebase account needed for Expo-managed push).
Docker Compose: api, postgres+postgis, redis, minio, worker. `.env.example` with every variable documented.
AI Engine (separate `services/ai` module, mock-model-swappable)
Text classifier (category from description, EN+NE keyword/embedding hybrid), image classifier interface (pluggable — stub returns "unverified" confidence, real model swappable later), severity estimator, rule+ML hybrid priority scorer (configurable weights, officer override), duplicate detection (geo proximity + category + text cosine similarity), hotspot detection (DBSCAN via PostGIS `ST_ClusterDBSCAN`), trend analysis.
Every AI output persisted to `ai_analysis` with confidence and model version; every surface labels it "AI-suggested, pending officer verification."
Monorepo: pnpm workspaces + Turborepo.
```
nagardrishti/
├─ apps/
│  ├─ mobile/                 # Expo app
│  │  └─ src/
│  │     ├─ app/              # expo-router routes, grouped by role: (auth) (citizen) (officer) (field) (admin) (shared)
│  │     ├─ components/       # ui/ (design system), map/, charts/, motion/
│  │     ├─ features/         # report/, verification/, assignment/, notifications/, profile/, intelligence/
│  │     ├─ hooks/  lib/  store/  theme/  i18n/  services/
│  └─ api/                    # Fastify app
│     └─ src/ modules/ (auth, reports, ai, assignments, notifications, analytics, admin, files) plugins/ jobs/ sockets/
├─ packages/
│  ├─ types/                  # shared Zod schemas + TS types (single source of truth)
│  ├─ api-client/             # typed fetch client generated from schemas
│  └─ config/                 # eslint, tsconfig, prettier presets
├─ services/ai/               # classifiers, scorer, clustering
├─ infra/ (docker-compose.yml, nginx, minio policies)
└─ scripts/ (create-first-admin.ts, backup.sh, migrate.sh)
```
---
3. Security (defense-in-depth — implement all; nothing deferred)
Auth & identity
Argon2id password hashing; zxcvbn strength meter; breached-password check (offline k-anonymity list).
JWT access (15 min) + rotating refresh tokens (httpOnly-equivalent: stored in expo-secure-store / Keychain / Keystore), refresh-token reuse detection → session revocation.
RBAC middleware with per-route permission matrix; row-level ownership checks (citizen sees only own reports).
OTP phone verification (pluggable SMS provider interface; dev transport logs OTP), optional TOTP 2FA for officer/admin, biometric app lock (expo-local-authentication).
Device/session management screen: list active sessions, revoke any.
API hardening
6. Zod validation on every request body/query/param; strict output serialization (no accidental field leaks).
7. Rate limiting per-IP and per-user (Redis sliding window), stricter on auth/OTP endpoints; login backoff + lockout with unlock email/SMS.
8. Helmet-equivalent headers, CORS locked to app, HTTPS-only, HSTS.
9. SQL injection impossible via Prisma parameterization; PostGIS raw queries parameterized only.
10. File upload: MIME sniffing (magic bytes, not extension), size caps, image re-encode via sharp (strips EXIF GPS + malware payloads), presigned MinIO URLs, private buckets, signed GET URLs with expiry.
11. Idempotency keys on report submission (offline retry safety).
12. Audit log table for every privileged action (who, what, before/after, IP, timestamp) — immutable append-only.
13. Secrets never in code; `.env` + validation at boot (fail fast if missing).
14. Dependency scanning (`pnpm audit` in CI), lockfile pinned; Semgrep/ESLint-security ruleset in CI.
15. Structured logging (pino) with PII redaction; Sentry (self-hostable GlitchTip alternative noted) for crash + error reporting on app and API.
16. Backups: nightly `pg_dump` + MinIO sync script; restore documented in README.
17. Mobile: certificate pinning for API host, jailbreak/root detection warning, screenshot blocking on sensitive admin screens (Android FLAG_SECURE), MMKV encryption key held in secure store.
18. Privacy: consent screens for location/camera, data-export and account-deletion endpoints (GDPR-style), report anonymization option for citizens.
---
4. Motion & Performance Bar (120 fps-class on capable devices, graceful everywhere)
All animation on the UI thread: Reanimated 3 worklets + Moti. Zero JS-bridge-driven animation. Never animate layout props when transform/opacity suffices.
Motion is app-wide, not just lists: shared-element transitions between report card → detail (expo-router shared transitions), spring-physics bottom sheets (@gorhom/bottom-sheet), gesture-driven map panel, skeleton shimmer via Skia shader, animated Skia charts (bars grow, lines draw), count-up stat tiles, staggered list entrances, pull-to-refresh custom Skia indicator, status-timeline draw-in animation, haptic feedback on all confirmations, animated tab bar with morphing indicator, hero 3D scene (react-three-fiber) on auth screen with device-tier gating.
Respect system reduce-motion + in-app "Reduce motion" toggle (kills 3D, shimmer, parallax; keeps functional transitions).
Lists 50+ items → FlashList; maps 50+ markers → Supercluster clustering. Images always via expo-image with blurhash.
Cold start budget < 2.5 s mid-range Android; Hermes enabled; bundle analyzed; heavy screens lazy-loaded.
Low-bandwidth mode: aggressive image compression, tile caching, defer non-critical fetches.
---
5. Design System (premium, government-credible, not a template)
Design tokens in `theme/`: color scales (semantic: primary deep civic blue, priority scale — critical red / high orange / medium amber / low green — plus dark-mode variants), spacing 4-pt grid, radius scale, elevation/shadow tokens, motion duration/easing tokens.
Full dark mode + system-follow.
Typography scale (Space Grotesk display / Inter body / Noto Sans Devanagari when locale = NE), tabular numerals for stats.
Component library in `components/ui/`: Button (variants/sizes/loading), Input, Select, Chip, Badge, Card, Sheet, Dialog, Toast (queue), Tabs, SegmentedControl, Avatar, EmptyState, ErrorState, Skeleton, StatusPill, PriorityIndicator, Timeline, Stepper, MapMarker set, StatTile, ChartCard.
Accessibility: WCAG AA contrast, accessibilityLabel/role on every interactive element, dynamic type support, minimum 44-pt touch targets, screen-reader-tested flows.
Every screen has designed empty, loading (skeleton), error, and offline states — no blank screens ever.
---
6. Feature Catalog (build all — 200+ across modules)
Auth & Account (1–20): phone+OTP register · email optional · login · refresh rotation · biometric unlock · 2FA (officer/admin) · password reset via OTP · session list/revoke · profile edit · avatar upload · ward + municipality fields · language switch EN/NE · BS/AD date preference · theme switch · reduce-motion toggle · notification preferences · data export · account deletion · consent management · onboarding carousel (skippable, animated).
Citizen — Reporting (21–50): category picker with icons · multi-photo capture/pick (≤5, compressed) · voice-note attachment · description with NE transliteration keyboard support · GPS auto-capture + draggable pin · reverse-geocoded address · ward auto-detect from PostGIS boundaries · landmark field · severity self-estimate (advisory) · anonymous submission toggle · draft autosave (MMKV) · offline queue with idempotent sync + retry/backoff · submit progress with per-photo upload state · post-submit AI Analysis card (detected category, severity, priority, confidence %, possible duplicates — labeled AI-suggested) · duplicate warning before submit ("A similar report exists 40 m away — add your report as supporting evidence?") · edit within 15 min · withdraw report · re-open request on unsatisfied resolution · report templates for common issues · attach follow-up photos to own open report.
Citizen — Tracking & Community (51–75): My Reports list (filter/sort/search) · status timeline with animated stepper · per-status timestamps + acting department · notification center with read/unread · push notifications on every transition · upvote/"affects me too" on nearby public reports · comment thread on own report with officer replies · resolution photo comparison (before/after slider) · satisfaction rating + feedback after resolution · nearby issues feed (radius slider) · emergency/high-priority public feed · ward leaderboard (most responsive wards) · personal impact stats (reports filed, resolved) · share report as deep link · in-app report QR for offline reference · saved locations (home/work) · watch a report (subscribe to updates) · public transparency stats screen.
Live Map (76–95): full-screen map with cluster markers colored by priority · marker → animated preview card → detail · filters: category, priority, status, date range, ward · hotspot heat layer toggle (Skia-rendered) · my-location + compass · search places/reports · draw-radius query · offline tile cache for last viewport · map style light/dark · ward boundary overlay (GeoJSON) · long-press to start a report at that point · live updates via socket (new/changed markers animate in) · legend sheet · marker decluster on zoom with spring animation.
AI Engine surfaces (96–115): category suggestion with confidence · severity estimate · priority score breakdown (weights visible) · officer priority override with reason · duplicate candidates list with similarity % · merge suggestion · hotspot cards ("Ward 9 Road Damage Hotspot — 17 related reports — Recommended: municipal road inspection") · trend insights ("Drainage complaints up 38% this month") · recurring-problem detection (same location re-reported ≥3×) · department routing suggestion · resolution-time prediction · workload-balance suggestion for assignment · AI model version + confidence shown on every card · per-category weight configuration (admin) · full "NagarDrishti Intelligence" dashboard panel: Top Priority Today, Civic Hotspots, Recurring Problems, AI Insights, Recommended Actions — every card labeled "AI-assisted insight, not a directive."
Officer (116–150): verification queue (urgent-first, swipe actions) · report detail with photos, map, AI card, citizen history · verify / reject (reason required) / merge duplicates (children preserved + citizens notified) · assign to department + field staff with deadline · reassign · bulk actions · priority override · internal notes (officer-only) · request more info from citizen · escalation flag · SLA timers with breach alerts · analytics: by category, ward, resolution time, monthly trends, staff performance (Skia/Victory charts, animated) · export CSV · saved filters · officer notification digests · shift handover summary · map view of unassigned criticals · citizen-communication templates.
Field Staff (151–175): assigned-task list + map · task detail: photos, description, priority, deadline, navigation deep link (Google/Apple Maps) · state actions: Accept → Start Work → Complete · mandatory completion photo + note · geo-verified completion (must be within X m of report location) · offline task caching + sync · deadline countdown with color states · route ordering suggestion for multiple tasks · materials-used note field · request reassignment/help · work-history log · daily summary · push on new assignment · in-progress photo updates visible to citizen.
Admin (176–200+): user management (create officer/field accounts, deactivate, role change) · department CRUD with ward coverage · category CRUD with icon + department mapping · AI weight configuration · SLA policy configuration · ward/boundary GeoJSON upload · announcement broadcast to citizens (ward-targeted) · audit log viewer with filters · system health screen (queue depth, error rate) · content moderation queue (flagged photos/text, profanity filter EN/NE) · banned-user management · notification template editor · data export (full) · backup status · feature flags · app-version force-update gate · analytics overview across municipalities · API key management for future integrations · privacy/data-deletion request handling · about screen with school credit.
---
7. Workflow State Machine (server-enforced)
`SUBMITTED → AI_ANALYZED → VERIFICATION_REQUIRED → VERIFIED → ASSIGNED → IN_PROGRESS → RESOLVED` (+ `REJECTED`, `MERGED`, `REOPENED` side states).
Transitions validated server-side against a single transition map; role checks per transition; every transition appended to `status_history` with actor, reason, timestamp. AI can only move SUBMITTED → AI_ANALYZED. Nothing past VERIFICATION_REQUIRED without officer/admin.
---
8. Database Schema (Prisma + PostGIS)
users · sessions · reports · report_images (type citizen/progress/completion) · report_upvotes · report_comments · categories · departments · locations (PostGIS point + ward) · ward_boundaries (PostGIS polygon) · ai_analysis (model_version, confidence, duplicate_of) · assignments · status_history · notifications · notification_preferences · hotspots (materialized, recomputed by job) · audit_logs · announcements · feature_flags · sla_policies · moderation_flags · app_settings.
All tables: created_at/updated_at, indexed FKs, GiST indexes on geometry columns, soft-delete where user-facing.
---
9. API Design
REST `/api/v1/*`, resource-oriented, cursor pagination, consistent error envelope `{ code, message, details }`, request IDs, OpenAPI spec auto-generated from Zod schemas and used to generate `packages/api-client`. Socket.io namespaces: `/map`, `/notifications`, `/reports/:id`. Webhook-style internal events via BullMQ.
---
10. Notifications
Triggers: received, AI-analyzed, verified, rejected, assigned, work started, progress photo, resolved, comment reply, announcement, SLA breach (staff), reopened. Channels: Expo push + in-app center + optional SMS interface (stubbed provider). Per-user preference matrix. Digest batching to avoid spam.
---
11. Nepal-Focused Requirements
Bilingual EN/NE UI (i18next, all strings externalized) · Devanagari rendering with Noto Sans · Bikram Sambat dates alongside AD where practical · ward + municipality data model (demo municipality config: Damak, Jhapa — configuration only, not seeded content) · Nepal-centered default map camera · low-bandwidth mode · NPR formatting where currency appears · phone-first identity (Nepali number validation).
---
12. Testing & QA
API: Vitest unit + integration (Testcontainers Postgres), state-machine transition matrix fully covered, auth/RBAC tests, upload security tests.
Mobile: Jest + React Native Testing Library for components/hooks; Maestro E2E flows for the 5 golden paths (register→report, officer verify→assign, field complete, merge duplicates, citizen rate resolution).
Type safety end-to-end via shared Zod schemas; `tsc --noEmit` and ESLint (security + a11y plugins) gate CI.
Performance pass: Flipper/DevTools profile, no JS-thread frame drops on golden paths, FlashList blank-cell audit.
Accessibility audit checklist per screen.
---
13. CI/CD & Packaging
GitHub Actions: lint → typecheck → test → build. EAS Build profiles (dev/preview/production) + EAS Update for OTA JS updates. API Docker image + compose for prod (nginx TLS termination). Versioned DB migrations (`prisma migrate`). README: architecture diagram (mermaid), setup in ≤10 commands, environment table, backup/restore, security notes, walkthrough of the 5 golden paths.
---
14. Definition of Done
[ ] Expo app: all four role experiences functional against the real API (no mock data in app code)
[ ] Backend + AI engine live in Docker Compose; AI endpoints mock-model-backed but interface-ready for real models
[ ] Full state machine enforced server-side with audit trail
[ ] Live clustered map + hotspot layer + realtime updates
[ ] Security checklist (Section 3) fully implemented
[ ] Motion system per Section 4 across the whole app, reduce-motion respected
[ ] EN/NE i18n complete, BS dates, dark mode
[ ] Test suites green, Maestro golden paths pass
[ ] EAS build profiles + README + `.env.example` complete
[ ] Zero emojis; icons only; school credit on About screen
---
15. Build Order
Scaffold monorepo, packages/types, infra compose, CI skeleton
DB schema + migrations + Prisma client
Auth + RBAC + security middleware
Reports module + uploads + state machine + audit
AI engine service + BullMQ jobs + duplicate/hotspot
Realtime + notifications
Mobile: design system + theme + navigation shell
Citizen flows → Map → Officer → Field → Admin
Motion layer pass (Section 4) across all screens
Intelligence panel + analytics
Testing, a11y, performance pass
Packaging, README, EAS profiles
Execute now. Begin with Phase 1.
