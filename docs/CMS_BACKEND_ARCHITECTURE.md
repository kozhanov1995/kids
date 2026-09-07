# CMS_BACKEND_ARCHITECTURE.md — Content Platform, Admin CMS & Backend

**Status: PLANNING DOCUMENT — not yet approved for implementation.**

This document is the architecture audit and implementation plan for turning
Kids Growth ("Orle Kids") from a mock-data application into a manageable
content platform with a private Admin CMS and a real backend. It is based on
a full read of the actual repository (`app/`) as it exists today, not on an
idealized greenfield design. No code changes were made while producing this
document.

Naming note: the product is referred to as **"Orle Kids"** in the brief that
requested this document, but the repository, `package.json`, and every
existing doc (`CLAUDE.md`, `README.md`, `docs/*.md`) call it **"Kids
Growth."** This document uses "Kids Growth" when referring to the existing
codebase/docs and "Orle Kids" only when quoting the brief. Resolve this
before shipping the CMS (rename is cheap now, expensive once admin UI text,
Supabase project name, and buckets exist).

The UI/UX design and release QA phases are treated as frozen per the
instruction that triggered this document. Nothing below proposes a redesign
of any existing screen; every UI-facing change (Admin CMS itself, the
service-status banners it implies) is new surface area, not a rework of
frontend V1.5.

---

## 0. Executive summary

The codebase is materially **further along than its own docs admit**:

- `docs/DATABASE.md` / `docs/DATABASE_AND_API.md` describe Supabase
  integration as "V2.0, not started." In reality, `app/src` already has:
  - A real Supabase client/server/middleware setup (`@supabase/ssr`).
  - Email+password **and** phone+OTP auth, wired to real Supabase Auth calls.
  - `Hybrid*Repository` classes for diary entries and completed activities
    that transparently route to Supabase when a session exists and to
    LocalStorage otherwise, with a working local→remote sync-on-login path
    (`syncLocalDataToRemote`).
  - A `SupabaseChildrenRepository` with get-or-create-default-child logic.
  - A hand-written `Database` type (`shared/types/database.ts`) matching the
    one applied migration.
- What is **not** built yet, and is the actual subject of this brief: any
  CMS content table, any admin surface, real LLM wiring for the assistant,
  a media/asset model, RAG/pgvector, assistant observability, and an
  importer from the 430+ mock content records into Postgres.
- The single Supabase migration
  (`app/supabase/migrations/20260717000000_init_schema.sql`) has **not been
  applied to a live Supabase project** — there is no `.env.local` value
  present beyond a placeholder, no `supabase/config.toml`, and no Supabase
  project is referenced anywhere. This is a decision point, not a blocker:
  Phase V2.0 below assumes a Supabase project needs to be provisioned from
  scratch.
- Content today is **not just hardcoded — it is hand-authored as
  TypeScript/TSX**, not JSON: cover art for articles/games/facts is not URLs
  to images at all, it's ~30 bespoke inline-SVG React illustration
  components selected by category enum
  (`shared/illustrations/*-illustration.tsx`). There is no `MediaAsset` of
  any kind in the app today. `next.config.ts` only allowlists
  `images.unsplash.com`, which is unused (`grep` for `unsplash`/`coverUrl`
  in the actual mock data returns nothing) — a leftover from scaffolding,
  not a real image pipeline.
- Repo hygiene: the actual project lives at
  `C:\Users\tuf\Desktop\kids\app` (git repo, one commit — "Initial commit
  from Create Next App" — with essentially the entire app sitting as
  uncommitted changes on top of it). There is no remote configured, no CI,
  no `supabase/config.toml`. This affects the "deployment assumptions"
  inventory item below: there are effectively none yet beyond a Dockerfile
  (`output: "standalone"`) and aspirational mentions of Vercel in the docs.

This matters for planning: **V2.0 (backend foundation for user data) is
~70% done.** The real, unstarted work is the CMS/content/media/AI-studio/RAG
stack this brief is actually asking for. The phased plan in §15 reflects
this — it does not re-propose work that already exists.

---

## 1. Current architecture inventory

| Area | Finding |
|---|---|
| Framework | Next.js 16.2.10 (App Router), React 19.2.4, TypeScript strict. `next dev/build --webpack` (Turbopack explicitly disabled — Serwist doesn't support it yet, see [`ARCHITECTURE.md`](../docs/ARCHITECTURE.md) §7). |
| Architecture style | Feature-Sliced-ish: `app/ → widgets/ → features/ → entities/ → shared/`, one-way import rule enforced by convention (not lint-enforced today — no import-boundary ESLint rule found). |
| Routing | App Router with `[locale]` segment (`ru`\|`kk`), route groups `(main)` (bottom-nav shell) and `(auth)` (login/register). No route groups yet for a private admin area. |
| Auth | Real Supabase Auth: email+password (`signInWithPassword`, `signUp`) and phone+OTP (`signInWithOtp` / `verifyOtp`) in `src/features/auth/api/actions.ts`. Session exposed via React Context (`SessionProvider`, client-side `getSession`/`onAuthStateChange`). `src/proxy.ts` (Next.js 16 renamed `middleware.ts` → `proxy.ts`; easy to miss on a first pass) refreshes the session server-side on every request via `updateSupabaseSession()`, alongside the next-intl locale routing — **but it only refreshes, it doesn't gate**: no route anywhere redirects an unauthenticated visitor away from a `(main)` page. That's deliberate, not a gap — `docs/REQUIREMENTS.md` §1.1 specifies a guest browsing mode for mock content — but it means `/admin` needs its own explicit gate (added in `src/app/admin/(protected)/layout.tsx`, checked server-side on every request via `requireAdminSession()`) rather than inheriting anything from the parent app's proxy. `isSupabaseConfigured` gates everything else — the app works with zero backend if env vars are absent. |
| User/profile architecture | `profiles` (1:1 with `auth.users`), `children` (1:N per profile). Client-side `Child` type still uses `gender: "unspecified"` vs the DB's `"other"` — mapped at the repository boundary (`DB_TO_APP_GENDER`/`APP_TO_DB_GENDER` in `entities/child/model/supabase-repository.ts`). Only ever resolves/creates **one** default child (`getOrCreateChildren` returns `children[0]` everywhere it's consumed) — multi-child UI is not actually wired despite the data model supporting N children. |
| LocalStorage stores | `entities/diary-entry/model/local-storage.ts` (key `kids-growth:diary:<childId>`), `entities/progress/model/local-storage.ts` (key `kids-growth:progress:<childId>` — inferred, same pattern), `features/checklists/model/use-checklist-progress.ts` (key `kids-growth:checklist-progress:<checklistId>`, **not yet backed by any Supabase table or repository interface** — pure LocalStorage, no Hybrid/Supabase counterpart exists). |
| `useSyncExternalStore` stores | Same three: diary, progress, checklist-progress — plus `shared/hooks/use-is-client.ts`/`use-time-of-day.ts`/`use-today-label.ts` (client/time detection, not persisted state) and `widgets/continue-section/continue-section.tsx` (reads across the above to build a "continue where you left off" card). |
| Content mock files | 8,533 lines across `entities/*/mock/*.ts`: articles (1,161 lines / ~16 articles), games (split by goal, ~100 games), facts (split by category, ~100), tips (split by category, ~100), calm-moments (split by kind, ~50), milestones (299 lines / ~29), checklists (113 lines / ~6 checklists, 31 items), daily-recommendations (split by age band). Every localizable field is `{ ru: string, kk: string }` inline (`LocalizedText`), not a separate translation table. |
| Games model | `entities/game/model/types.ts`: `id, slug, title, description, ageRangeMonths, goals[], durationMinutes, difficulty, materialsNeeded[], steps[], commonMistake, expectedResult, parentTip?, youtubeId?`. `youtubeId` field exists but is unused in every mock record found — video is architecturally anticipated, not implemented. |
| Daily Games model | Separate mock set (`entities/game/mock/daily-games.ts`), same `Game` type, extra goals (`attention`/`gestures`/`imitation`/`calm`) layered onto `GameGoal`. Selected via `pickOfTheDay()` (day-of-year modulo list length, `features/daily-recommendations/lib/daily-index.ts`) — fully deterministic, no backend, no personalization beyond age filtering. |
| Knowledge/Articles model | `entities/article/model/types.ts`: rich — `content` (localized paragraph arrays), `tips[]`, `faq[]`, `todayAction`, `whenToSeeSpecialist`, `usefulLinks[]` (in-app routes only, by design — "we never fabricate external medical links"). No `sources[]` field yet despite `docs/CONTENT.md` §6 flagging it as a "Stage 2" need. |
| Facts model | `entities/fact/model/types.ts` — short cards, reuses `ArticleCategory` taxonomy (no separate category enum) — confirmed by `docs/CONTENT.md` §9. |
| Calm Moments model | `entities/calm-moment/model/types.ts` — `title, description, durationMinutes, whenToUse, parentTip`; **no audio field at all** — "Listen later" is a non-functional placeholder button today (confirmed in `docs/REQUIREMENTS.md` §5.5 and `docs/CONTENT.md` §9). |
| Checklists model | `Checklist { id, slug, category, title, description, ageRangeMonths, items: ChecklistItem[] }`; progress tracked per-checklist in LocalStorage only, not per-item ID stability guaranteed across content edits (a CMS edit that removes/reorders items would silently desync existing users' progress — see §21). |
| Milestones model | `Milestone { id, domain, ageRangeMonths, title, description }` — flat list, no checklist/game cross-links. |
| Diary model | User-generated, not content: `DiaryEntry { id, childId, date, category, text }`. Real Supabase table exists (`diary_entries`) and a working `HybridDiaryRepository` already syncs it. |
| Progress/completion models | `CompletedActivity { id, childId, activityType, activityId, completedAt }` — generic across game/fact/article/checklist/tip, keyed by content **slug** (`activityId`), unique per `(child_id, activity_type, activity_id)`. This is the load-bearing reason content IDs/slugs must be preserved during migration (§12, §21). Distinct from `checklist_progress` (per-item, still LocalStorage-only, not yet in this table or any Supabase table). |
| AI Assistant architecture | Fully deterministic today, **zero LLM calls, zero AI SDK dependency** (`grep` for `ai`/`@ai-sdk`/`OPENAI`/`ANTHROPIC` in `package.json` and env files: nothing). `POST /api/ai-chat` (`src/app/api/ai-chat/route.ts`) validates with Zod, detects ru/kk by Kazakh-only-letter regex, checks an `EMERGENCY_KEYWORDS` list first (hard-coded phrases, bypasses everything else), otherwise calls `getAssistantReply()` (`features/ai-chat/api/mock-assistant.ts`) — a pure function returning one of 16 canned answers (8 quick-question IDs × 2 locales) built from 4-block templates (empathy → explanation → 5-minute game → "when to see a specialist"). `buildSystemPrompt()` exists and is fully written (matches the V2 prompt spec in `docs/AI_COMPANION.md` §7) but is called with `void` — built for parity, never sent anywhere. Response is streamed client-side via a fake `setTimeout` chunker, not a real model stream. No conversation persistence (`ai_chat_conversations`/`ai_chat_messages` tables are documented in `DATABASE.md` §3 as planned but **do not exist in the applied/pending migration** — only `profiles`, `children`, `diary_entries`, `completed_activities` are in the actual SQL file). |
| Current API routes | Exactly one: `POST /api/ai-chat`. No REST/RPC surface for content, no admin API, no webhook endpoints. |
| Media/video handling | None. `youtubeId?` field on `Game` is unused. `next.config.ts` allowlists `images.unsplash.com` for `next/image` but no mock record references an external image URL — all visual content is hand-authored inline SVG illustration components (`shared/illustrations/`, ~30 files) selected by category/domain enum, not by a media reference. This is a first-class fact for the Media Library design (§9): illustrations-by-category must remain the fallback/default even after a real `MediaAsset` model exists. |
| Localization architecture | `next-intl`, locale as first path segment (`/ru/...`, `/kk/...`), `middleware.ts`-driven (cookie → `Accept-Language` → default `ru`). UI strings: `shared/i18n/messages/{ru,kk}.json`, namespaced by feature. Content strings: inline `{ ru, kk }` objects on every mock record (**not** the same mechanism as UI strings — two parallel localization systems). `en` is reserved in the DB check constraint (`profiles.preferred_locale`) and in docs but **not** in `shared/i18n/routing.ts`'s `locales` array — adding `en` today would under-serve the DB, which already allows it, and over-promise relative to the actual `next-intl` config. |
| Deployment assumptions | Docker (`Dockerfile`, `output: "standalone"` in `next.config.ts`) is the only deployment path actually configured. Docs mention Vercel as the target, but nothing Vercel-specific exists (no `vercel.json`, no Edge Function, no `.vercel/`). No git remote configured. No CI config found anywhere in the repo. |
| Environment configuration | `.env.example` has exactly two keys: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`. No service-role key, no LLM provider key, no storage/bucket config, no analytics key. `isSupabaseConfigured` is computed from the two public keys only — meaning **no server-only Supabase client with elevated privileges exists yet** (relevant: Admin CMS writes will need one, via RLS-bypassing service-role calls from server actions/route handlers only, never shipped to the browser). |

---

## 2. Problems with the current data architecture

1. **Content changes require a code deploy.** Every article/game/fact/tip/
   checklist/milestone/calm-moment edit is a TypeScript file edit + PR +
   build + deploy. There is no way to fix a typo in production without
   shipping a new app version — this is the entire premise the brief is
   asking to fix.
2. **No draft/review/publish lifecycle exists for content.** A mock file
   edit ships directly. There is no concept of "draft" content at all today.
3. **No versioning/audit trail for content.** A bad edit to
   `articles.ts` is only recoverable via git history, which is not always a
   safe assumption on this repo (single squashed initial commit, rest
   uncommitted at time of audit).
4. **Two disconnected localization systems.** UI strings go through
   `next-intl`; content strings are ad hoc `{ ru, kk }` objects with no
   validation beyond the `I18N.md` §9 manual parity script and no
   translation-status tracking (is a `kk` string a real translation or a
   stale copy of `ru`?).
5. **`checklist_progress` is the one piece of user state with no backend
   story at all** — not even a documented target table matches what's
   actually needed once checklist items become admin-editable (item IDs need
   to survive content edits; today they're arbitrary strings assigned once
   in a mock file and never revisited).
6. **The assistant has no memory and no observability.** Nothing is
   persisted about what users ask, whether an answer was any good, or which
   questions the deterministic engine's `generic fallback` catches (the
   literal signal the brief calls "knowledge gaps"). You cannot build the
   feedback loop in §"KNOWLEDGE FEEDBACK LOOP" of the brief on top of the
   current `/api/ai-chat` — it is stateless by construction.
7. **No media model, so no reuse.** Every visual is a component, not a
   data-referenced asset — there is currently no way for an admin to "upload
   a photo and attach it to three articles" because there is no seam for a
   photo to attach to at all.
8. **Content IDs are informally stable, not contractually stable.**
   `activityId`/`activity_type` in `completed_activities` are content
   slugs. Nothing enforces that a slug, once shipped, can never change —
   this is tribal knowledge in `docs/DATABASE_AND_API.md`, not a constraint
   anywhere in code. A CMS that lets an editor rename a slug will silently
   orphan every user's progress on that item unless this is made explicit
   and enforced (§21).
9. **No server-side authorization boundary exists yet.** All Supabase calls
   today go through the anon key from the browser, protected only by RLS.
   That's correct and suftrue for user data, but an Admin CMS needs
   privileged writes (publish content, moderate media, read all users'
   aggregate stats) that must **never** be anon-key + RLS-only — this is a
   net-new trust boundary the app has not needed before.
10. **`isSupabaseConfigured` as a global on/off switch is a good instinct
    for offline-first user data, but it must not leak into the CMS.** The
    Admin app must hard-require a configured backend — there is no
    legitimate "admin mode with no database" state.

---

## 3. Target architecture

**Recommendation: keep one Next.js application, add `/admin` as a route
group — do not split into an `apps/web` + `apps/admin` monorepo.**

Reasoning, grounded in what's actually in this repo:

- The codebase is small (single Next.js app, ~180 source files under
  `src/`), has no existing monorepo tooling (no `pnpm-workspace.yaml`, no
  Turborepo/Nx config, no shared-package boundary already in place).
- The FSD-ish layering (`app/widgets/features/entities/shared`) already
  gives the admin surface a clean seam: a new `features/admin-*` and
  `entities/*` content types slot in exactly like existing features, and
  `shared/api/supabase` is already the one Supabase access point both sides
  would use.
- The admin surface and the parent-facing app share almost everything that
  matters for a monorepo split to pay for itself: the same content types
  (`Game`, `Article`, ...), the same Supabase project, the same i18n
  message format conventions, the same design tokens if the admin UI reuses
  `shared/ui`. Splitting now would mean solving cross-package type-sharing
  (content Zod schemas, `Database` types) for zero present benefit.
- A monorepo becomes worth it when: the admin app needs a different deploy
  cadence/target than the parent app, a different team owns it, or bundle
  size of the parent PWA starts being polluted by admin-only code. None of
  these are true yet. If they become true post-V2.x, the FSD boundaries
  already in place make extracting `/admin` into its own app a mechanical
  refactor, not a redesign.
- Concretely: `src/app/[locale]/admin/(admin)/...` **cannot** reuse the
  `[locale]` parent layout (bottom nav, parent-facing chrome) — it needs its
  own route group with its own layout, likely **outside** the `[locale]`
  segment entirely (`src/app/admin/...`, English-only UI, no next-intl
  routing needed for an internal tool used by a small Kazakhstani ops team
  who can read Russian regardless of what locale segment convention the
  parent app uses). This avoids admin URLs being subject to locale
  redirects meant for parents.

**Supabase for**: Postgres (content + user data, one project, two logical
domains via schemas or just table-prefix/RLS separation — a single `public`
schema is fine at this scale), Auth (parents via phone/email, admins via a
separate flow, see §6.1), Storage (media library), RLS (both domains),
pgvector (assistant knowledge retrieval — `pgvector` extension on the same
Postgres instance, no separate vector DB service). Realtime is **not**
recommended for V2.x: nothing in the current product needs live
multi-viewer sync (no collaborative editing requirement stated, no
multi-device live-progress requirement). Revisit only if admin content
editing needs live "someone else is editing this" presence — not a stated
need today.

---

## 4. What remains local vs. what moves to backend

Classification per the brief's A–E scheme, applied to every concrete entity
found in the repo (not a generic list):

| Data | Class | Target |
|---|---|---|
| UI copy (`shared/i18n/messages/*.json`), brand illustrations (`shared/illustrations/*`), design tokens, icons, PWA manifest | **A** — static brand/UI | Stays bundled. Illustrations remain the *default* visual for content that has no uploaded media (§9). |
| Articles, Milestones, Checklists (+ items), Games (incl. daily), Facts, Tips, Calm Moments, Daily Recommendations pool | **B** — CMS content | Move to Postgres (`content_*` tables, §5). Currently mock TS. |
| `profiles`, `children` | **C** — user data | Already in Postgres (existing migration), already has a working repository. No change needed beyond the profile-creation trigger noted in §21. |
| `diary_entries`, `completed_activities` | **C** — user data | Already in Postgres with a working Hybrid repository + offline fallback. No change needed. |
| `checklist_progress` (per-item completion) | **C** — user data | **Net new table needed** (documented as a target in `DATABASE.md` §3 but never migrated). Must key on a stable `checklist_item_id`, not array position (§21). |
| AI chat messages (once real) | **C** — user data (with **D** derived caching, see below) | New `ai_conversations` / `ai_messages` tables (§10). |
| Theme (light/dark/system), locale cookie, "which onboarding tooltip has been dismissed" | **D** — local device state | Stays `localStorage`/cookie — no product reason to sync across devices today (`DATABASE.md` §3 explicitly defers `profiles.theme` until a "family cabinet" feature exists — no evidence that's imminent). |
| Sync queue for offline mutations (the FIFO queue documented in `DATABASE_AND_API.md` §4, **not yet implemented** in code — only documented) | **D** — local device state | Build as designed in that doc once real content-driven UI depends on write-through (§11). |
| Child's computed age in months, "today's recommendation" index, "is this checklist complete" (derived from item completion count) | **E** — derived | Never store; compute at read time as today (`getAgeInMonths()`, `getDailyIndex()`, `useChecklistProgress().completedCount`). No change. |
| Retrieval chunks/embeddings for RAG | **B/E** hybrid — derived from B, stored because computing embeddings per-query is wasteful | `content_chunks` + `content_embeddings` tables (§10), regenerated on publish, not user-facing "content" in their own right. |
| Media files (images/video/audio) | **B** — CMS content (the asset) / **A** for bundled brand assets | Supabase Storage, referenced by `media_assets` table (§9). |
| Admin audit log, AI generation drafts | **B**-adjacent — operational content | Postgres, admin-only RLS. |

---

## 5. Proposed database schema

All new tables live in `public` alongside the existing four. Naming
convention matches what's already there (`snake_case`, `timestamptz`,
explicit `created_at`/`updated_at` triggers via the existing
`set_updated_at()` function — reuse it, don't reinvent).

### 5.1 Enums

```sql
create type content_type as enum (
  'game', 'daily_game', 'article', 'fact', 'tip',
  'checklist', 'milestone', 'calm_moment'
);
create type content_status as enum ('draft', 'review', 'published', 'archived');
create type media_type as enum ('image', 'video', 'audio', 'music');
create type admin_role as enum ('super_admin', 'editor', 'viewer', 'expert_reviewer');
create type ai_generation_status as enum ('generated', 'edited', 'discarded');
create type recommendation_mode as enum ('auto', 'curated');
create type message_role as enum ('user', 'assistant', 'system');
create type assistant_feedback as enum ('helpful', 'not_helpful');
```

Reuses the existing `check (...)` style would also work (matches current
migration's convention more closely than enums do — the existing migration
uses `text ... check (x in (...))`, not Postgres enum types). **Decision
needed from the team**: stay consistent with the existing `text + check`
style for uniformity, or introduce enums now for the much larger content
domain where a `check` list would get unwieldy (games alone have 10
`GameGoal` values). Recommendation: use enums for the new content domain
(cheaper to `ALTER TYPE ... ADD VALUE` than rewrite a `check` constraint,
and this domain will grow), keep the existing four tables' `text + check`
columns untouched (no reason to churn a working migration).

### 5.2 Content domain

```sql
-- One row per piece of content, regardless of type. Type-specific fields
-- live in a jsonb `data` column validated by a Zod schema per content_type
-- at the application boundary (not by Postgres CHECK — the shapes are too
-- different across 8 content types and change more often than the DB
-- should need a migration for).
create table public.content_items (
  id uuid primary key default gen_random_uuid(),
  content_type content_type not null,
  slug text not null,                    -- STABLE, see §21. Never reused.
  status content_status not null default 'draft',
  data jsonb not null,                   -- type-specific fields (title, ageRangeMonths, steps, etc.), keyed by locale where localized (see §13)
  cover_media_id uuid references public.media_assets(id) on delete set null,
  source_refs jsonb,                     -- optional [{label, url}] for professional/sourced content (brief's "sources/references")
  ai_generated boolean not null default false,
  ai_generation_id uuid references public.ai_generations(id) on delete set null,
  created_by uuid references public.admin_users(id),
  updated_by uuid references public.admin_users(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  published_at timestamptz,
  archived_at timestamptz,
  legacy_id text                          -- original id/slug from the mock-data import, for traceability (§12)
);

create unique index content_items_type_slug_idx on public.content_items (content_type, slug);
create index content_items_status_idx on public.content_items (status);
create index content_items_type_status_idx on public.content_items (content_type, status);

-- Checklist items live as their own rows (not nested in `data`) precisely
-- because completed_activities/checklist_progress need a stable FK target
-- that survives independent of the parent checklist's edit history.
create table public.checklist_items (
  id uuid primary key default gen_random_uuid(),
  checklist_id uuid not null references public.content_items(id) on delete cascade,
  order_index int not null,
  data jsonb not null,                    -- { title: {ru, kk, ...} }
  legacy_id text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index checklist_items_checklist_id_idx on public.checklist_items (checklist_id, order_index);

-- Simple snapshot-based versioning (brief explicitly allows this instead of
-- a full event-sourced history): one row per save, newest first.
create table public.content_versions (
  id uuid primary key default gen_random_uuid(),
  content_item_id uuid not null references public.content_items(id) on delete cascade,
  data jsonb not null,                    -- full snapshot of content_items.data at save time
  status content_status not null,
  created_by uuid references public.admin_users(id),
  created_at timestamptz not null default now()
);
create index content_versions_item_id_idx on public.content_versions (content_item_id, created_at desc);
```

Why one polymorphic `content_items` table instead of 8 typed tables
(`games`, `articles`, ...): the brief's content types share the entire
lifecycle (draft/review/publish/archive/version/AI-generate/media-attach/
translate) and admin UI (one list view, one workflow, one audit trail). A
typed-table-per-type design would duplicate all of that machinery 8×. The
cost of `jsonb data` — no DB-level schema enforcement per type — is paid
back by enforcing each type's shape with a **Zod schema at the API/server-
action boundary** (mirrors exactly what the app already does for content:
every mock file's shape *is* a hand-maintained TS type today, this just
moves the enforcement point from "TypeScript compiler at build time" to
"Zod at write time," which is strictly more correct for content an admin
edits at runtime). `checklist_items` is the one deliberate exception,
justified above by the FK-stability requirement.

### 5.3 Media domain

```sql
create table public.media_assets (
  id uuid primary key default gen_random_uuid(),
  media_type media_type not null,
  title text,
  storage_path text not null,             -- Supabase Storage object path
  mime_type text not null,
  file_size_bytes bigint not null,
  duration_seconds numeric,               -- video/audio/music only
  width int,                              -- image/video only
  height int,                             -- image/video only
  thumbnail_storage_path text,
  alt_text jsonb,                         -- { ru: "...", kk: "..." } — a11y, required for images at publish time (enforced at API layer)
  tags text[] not null default '{}',
  external_video_provider text,           -- 'youtube' | null — see §9.3, kept nullable/optional by design
  external_video_id text,
  created_by uuid references public.admin_users(id),
  created_at timestamptz not null default now()
);
create index media_assets_type_idx on public.media_assets (media_type);
create index media_assets_tags_idx on public.media_assets using gin (tags);

-- Many-to-many: content references media without duplicating it. Covers
-- cover images AND in-body media (e.g. an article with 3 inline images).
create table public.content_media (
  content_item_id uuid not null references public.content_items(id) on delete cascade,
  media_asset_id uuid not null references public.media_assets(id) on delete restrict,
  role text not null default 'inline',    -- 'cover' | 'inline' | 'audio' | 'video'
  order_index int not null default 0,
  primary key (content_item_id, media_asset_id, role)
);
```

`on delete restrict` on `content_media.media_asset_id` is deliberate —
prevents deleting a media asset that's still referenced anywhere, forcing
the "orphaned asset" question (§9.4) to be answered explicitly by the admin
UI (unlink first, then delete) rather than silently breaking published
content.

### 5.4 Admin domain

```sql
create table public.admin_users (
  id uuid primary key references auth.users(id) on delete cascade,
  role admin_role not null default 'viewer',
  display_name text,
  created_at timestamptz not null default now()
);

create table public.audit_log (
  id uuid primary key default gen_random_uuid(),
  actor_id uuid references public.admin_users(id),
  action text not null,                   -- 'content.published', 'media.deleted', 'user.status_changed', ...
  entity_type text not null,
  entity_id text not null,
  metadata jsonb,
  created_at timestamptz not null default now()
);
create index audit_log_entity_idx on public.audit_log (entity_type, entity_id);
create index audit_log_created_at_idx on public.audit_log (created_at desc);

create table public.ai_generations (
  id uuid primary key default gen_random_uuid(),
  content_type content_type not null,
  input_params jsonb not null,            -- age range, goal, tone, language, etc. from the AI Content Studio form
  raw_output jsonb not null,              -- what the model returned, structured
  status ai_generation_status not null default 'generated',
  model text not null,                    -- e.g. 'claude-sonnet-5' — never omit, for auditability
  requested_by uuid references public.admin_users(id),
  created_at timestamptz not null default now()
);
```

### 5.5 Assistant / RAG domain

```sql
create table public.ai_conversations (
  id uuid primary key default gen_random_uuid(),
  profile_id uuid references public.profiles(id) on delete set null,
  child_id uuid references public.children(id) on delete set null,
  locale text not null,
  started_at timestamptz not null default now()
);

create table public.ai_messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references public.ai_conversations(id) on delete cascade,
  role message_role not null,
  content text not null,
  -- Retrieval/confidence metadata, NOT chain-of-thought (brief explicitly excludes internal reasoning):
  retrieved_content_ids uuid[],           -- which content_items were retrieved for this answer
  confidence numeric,                     -- retrieval similarity score or model self-reported confidence, 0-1
  is_fallback boolean not null default false,  -- true = generic/"don't know" fallback fired
  is_emergency boolean not null default false, -- true = emergency-keyword path fired
  feedback assistant_feedback,
  feedback_note text,
  created_at timestamptz not null default now()
);
create index ai_messages_conversation_idx on public.ai_messages (conversation_id, created_at);
create index ai_messages_fallback_idx on public.ai_messages (is_fallback) where is_fallback;
create index ai_messages_feedback_idx on public.ai_messages (feedback) where feedback = 'not_helpful';

-- Knowledge gap queue: derived view, not a separate table — a message is
-- "in the queue" iff is_fallback OR confidence < threshold OR feedback =
-- 'not_helpful', AND it hasn't been resolved yet. "Resolved" is tracked
-- explicitly because the same underlying question can recur many times
-- before someone gets to it.
create table public.knowledge_gap_resolutions (
  id uuid primary key default gen_random_uuid(),
  ai_message_id uuid not null references public.ai_messages(id),
  resolved_by uuid references public.admin_users(id),
  resolution_content_id uuid references public.content_items(id), -- the article/fact created to close the gap
  resolved_at timestamptz not null default now()
);

-- RAG: chunks + embeddings of PUBLISHED content only. Re-generated on publish/unpublish.
create extension if not exists vector;

create table public.content_chunks (
  id uuid primary key default gen_random_uuid(),
  content_item_id uuid not null references public.content_items(id) on delete cascade,
  locale text not null,
  chunk_index int not null,
  chunk_text text not null,
  embedding vector(1536),                 -- dimension per chosen embedding model; pick once, document it
  created_at timestamptz not null default now()
);
create index content_chunks_item_idx on public.content_chunks (content_item_id);
create index content_chunks_embedding_idx on public.content_chunks
  using hnsw (embedding vector_cosine_ops);
```

`content_chunks` rows are deleted and regenerated wholesale on every publish
(simplicity over incremental diffing — content items are short enough that
re-chunking the whole item is cheap) and deleted entirely on
unpublish/archive (enforces "archived/unpublished content must not continue
appearing in retrieval," per the brief, at the data layer rather than
trusting a query-time filter).

### 5.6 Daily recommendations domain

```sql
create table public.daily_recommendation_overrides (
  id uuid primary key default gen_random_uuid(),
  target_date date not null,
  audience_min_age_months int,            -- null = all ages
  audience_max_age_months int,
  content_item_id uuid not null references public.content_items(id),
  created_by uuid references public.admin_users(id),
  created_at timestamptz not null default now()
);
create unique index daily_rec_overrides_date_audience_idx
  on public.daily_recommendation_overrides (target_date, coalesce(audience_min_age_months, -1), coalesce(audience_max_age_months, -1));
```

`recommendation_mode` enum is a **read-time** concept, not a column: for a
given date/audience, if a row exists in `daily_recommendation_overrides`,
mode is `curated` and that row wins; otherwise mode is `auto` and the
existing deterministic `pickOfTheDay()` logic runs unchanged, just reading
from `content_items` instead of the mock arrays. This is the fallback
behavior the brief asks to define — no new state needed to express it.

### 5.7 checklist_progress (the one missing V1.5→V2.0 user-data table)

```sql
create table public.checklist_progress (
  id uuid primary key default gen_random_uuid(),
  child_id uuid not null references public.children(id) on delete cascade,
  checklist_item_id uuid not null references public.checklist_items(id) on delete cascade,
  is_completed boolean not null default true,
  completed_at timestamptz not null default now()
);
create unique index checklist_progress_child_item_idx
  on public.checklist_progress (child_id, checklist_item_id);
alter table public.checklist_progress enable row level security;
create policy "checklist_progress_all_own_children" on public.checklist_progress for all
  using (child_id in (select id from public.children where profile_id = auth.uid()))
  with check (child_id in (select id from public.children where profile_id = auth.uid()));
```

### 5.8 RLS summary for new tables

| Table | Policy |
|---|---|
| `content_items`, `checklist_items`, `content_versions`, `content_chunks` | `select` on `status = 'published'` for `anon`/`authenticated` (public read of published content); full CRUD for `admin_users` with role in `('super_admin','editor')`; `viewer` role gets `select` on everything including drafts, no writes. |
| `media_assets`, `content_media` | Same shape — published-content-linked media is public-readable; unlinked/draft-only media is admin-only. |
| `admin_users`, `audit_log`, `ai_generations` | Admin-only, no public access at all. `admin_users` self-row readable by the admin themself (so the admin app can show "logged in as ..."). |
| `ai_conversations`, `ai_messages` | Owner (`profile_id = auth.uid()`) can `select`/`insert` their own; **no `update`/`delete` from the client** (feedback is submitted via a dedicated RPC/server action that validates ownership, not a raw `update`, to keep `retrieved_content_ids`/`confidence` from being client-writable). Admins (`editor`/`super_admin`) get `select` on all rows for observability — this is explicitly sensitive (§"Users" below) and must be its own policy, not "admin can do anything," so it's auditable which admin role sees conversation content. |
| `knowledge_gap_resolutions`, `daily_recommendation_overrides`, `checklist_progress` | Former two admin-only; `checklist_progress` follows the existing "own children" pattern identically to `diary_entries`. |

---

## 6. Supabase architecture

### 6.1 Auth

Two populations, one Supabase Auth instance, distinguished by
`admin_users` membership (not a separate Supabase project — no product
reason for that overhead at this scale):

- **Parents**: unchanged — email/password + phone/OTP, exactly as built.
- **Admins**: email/password only (an internal ops tool for a small team —
  phone OTP adds SMS cost/complexity for zero benefit here). A user is an
  admin **iff** a row exists for them in `admin_users`; sign-up for the
  admin app is **not self-service** — rows are inserted by a `super_admin`
  via the Admin CMS's own Users→Admins screen or a one-time seed script,
  never via a public register form. This directly satisfies "Admin
  functionality must never be exposed merely by hiding UI routes" — the
  `/admin` route group's layout does a **server-side** check
  (`admin_users` row exists + role) before rendering anything, and every
  admin server action re-checks role server-side (never trust a client-
  sent role).

### 6.2 Database

Single Postgres instance/project, `public` schema, tables as in §5 plus the
existing four. No schema-per-domain split — RLS already gives the
isolation that matters, and a schema split would only add friction to
Supabase's auto-generated PostgREST API for no isolation benefit at this
table count.

### 6.3 Storage

One bucket per media type is simplest to reason about for policies and
lifecycle rules, and matches the brief's IMAGE/VIDEO/AUDIO/MUSIC framing:

```
media-images/   (public read via signed-URL-free public bucket; admin write)
media-video/    (public read; admin write)
media-audio/    (public read; admin write)  -- calm-moment sounds, lullabies
media-thumbnails/ (public read; generated server-side on upload)
```

All four are **public-read** buckets (content is meant for every app user,
not gated per-user) with **admin-only write** Storage policies (`insert`/
`update`/`delete` restricted to `auth.uid()` present in `admin_users`).
Public-read means no signed-URL machinery is needed for delivery — `next/
image` and `<audio>`/`<video>` tags hit the Supabase Storage public URL
directly, which also plays well with the Serwist cache strategy (§ Offline/
Cache below can `CacheFirst` these URLs like any other static asset).

Upload limits (Supabase Storage default is configurable per-bucket):
images 10 MB, video 200 MB, audio/music 50 MB — generous enough for the
described content (short instructional clips, lullabies) without inviting
users... except there are no user uploads here, only admin uploads, which
somewhat relaxes the abuse-surface concern but not the "don't let someone
fat-finger a 4K ProRes file into a mobile app's bundle-adjacent CDN"
concern. Enforce both client-side (before upload starts) and via a Storage
bucket-level size limit.

Accepted MIME types: `image/{jpeg,png,webp}`, `video/mp4`, `audio/
{mpeg,mp4,ogg}` — reject everything else at the Admin CMS upload widget and
again via Storage bucket MIME allowlist (defense in depth, since the CMS is
the only writer but "the CMS" includes whatever the admin's browser
extensions do to a file input).

Thumbnails: generated server-side (a Next.js server action calling `sharp`,
already a dependency) on image/video upload, stored in `media-thumbnails/`,
referenced by `media_assets.thumbnail_storage_path`. Video thumbnails need a
frame-extraction step — out of scope for `sharp` alone; either accept a
manually-uploaded poster frame at upload time (simplest, recommended for
V2.2) or add `ffmpeg` server-side later if volume justifies it.

### 6.4 RLS

Covered per-table in §5.8. General principle carried over from the existing
migration's own doc comment: "a single point of truth for ownership" — here
that's `admin_users` for anything admin-gated, and the existing
`children.profile_id = auth.uid()` subquery pattern for anything user-owned.

### 6.5 pgvector

`vector` extension, one `content_chunks.embedding` column, HNSW index (see
§5.5). Embedding model choice is an implementation detail deferred to V2.8,
but the dimension is a schema decision made once — document it in the
migration comment when written, since changing it later means rebuilding
the index and re-embedding everything.

---

## 7. Admin CMS information architecture

```
/admin                          Dashboard (KPIs from §"Analytics")
/admin/content                  Content list, filterable by type/status
/admin/content/games
/admin/content/daily
/admin/content/knowledge        (articles)
/admin/content/facts
/admin/content/calm-moments
/admin/content/checklists
/admin/content/milestones
/admin/content/:type/:id        Editor (form + preview + version history)
/admin/content/:type/new
/admin/ai-studio                 AI Content Studio (§8)
/admin/media                      Media Library
/admin/media/images
/admin/media/video
/admin/media/audio
/admin/users                       Registered users list + detail
/admin/users/:id
/admin/assistant                    AI Assistant Observability (§10)
/admin/assistant/conversations
/admin/assistant/questions
/admin/assistant/feedback
/admin/assistant/knowledge-gaps
/admin/recommendations              Daily Recommendations calendar
/admin/settings                     Admin users/roles, feature flags
/admin/settings/admins
```

Left nav groups: Dashboard · Content (submenu per type) · AI Content Studio
· Media Library (submenu per type) · Users · AI Assistant (submenu) · Daily
Recommendations · Settings. Notifications gets a `Settings →
Notifications (coming soon)` stub entry — architecturally acknowledged, not
built (per brief).

Each content-type list view shares one component (`AdminContentTable`)
parameterized by a per-type Zod schema + column config — this is the
payoff of the polymorphic `content_items` table from §5.2: one admin table/
editor shell, 8 form configs, not 8 admin sub-apps.

---

## 8. AI Content Studio architecture

Not a chat box. A structured, multi-step form → structured-output flow:

1. **Type picker** — one of the 8 `content_type` values, each with its own
   parameter form (age range, developmental goal, category, duration,
   difficulty, materials, tone, language) matching the brief's example
   almost verbatim for `Game`.
2. **Server action** calls the model (Claude, via the Anthropic API —
   `claude-sonnet-5` by default, per the project's own stack guidance) with:
   - A **fixed system prompt per content type**, not a freeform prompt box.
     Each system prompt: (a) states the content type's exact target schema
     (the same Zod schema the editor form and DB write path use — single
     source of truth, generated into the prompt, not hand-duplicated), (b)
     embeds the safety rules already written and battle-tested in
     `docs/CONTENT_RULES.md` §1–2 verbatim (this project already has a
     precise, enumerated banned/allowed-phrasing list — reuse it, don't
     re-derive it), (c) forbids inventing medical claims/sources — if the
     content type is one where `source_refs` matters (articles, milestones),
     the prompt requires the model either cite a real, checkable
     WHO/CDC/AAP-class reference or leave `source_refs` empty, never
     fabricate one.
   - **Structured output** via the API's native structured-output/tool-use
     mode constrained to the type's Zod schema — not "ask nicely for JSON
     and hope." A response that fails schema validation is retried once,
     then surfaced to the admin as a generation failure, never partially
     saved.
3. Result is written as one `ai_generations` row (`status = 'generated'`)
   and one `content_items` row (`status = 'draft'`, `ai_generated = true`,
   `ai_generation_id` set). **Never `published` — enforced at the DB/API
   layer, not just the UI**, i.e. the publish server action refuses to
   transition `ai_generated = true, status = 'draft'` straight to
   `published`; it must pass through `review` and a human-triggered publish
   action, and that action logs which admin user did it in `audit_log`.
4. Admin can: **Regenerate** (new `ai_generations` row, same params,
   replaces the draft's `data`, keeps the same `content_items.id` — doesn't
   fork a new content item), **Edit** (plain form edit, snapshots to
   `content_versions` on save), **Preview** (renders using the *actual*
   parent-app content components in an isolated preview route — not a
   separate admin-only renderer that can drift from production, see §"Files
   to change" for why this is feasible given current component structure),
   **Save draft**, **Submit for review** (`status → review`), **Publish**
   (`status → published`, sets `published_at`, triggers RAG re-chunking if
   `content_type` is retrieval-eligible).

### 8.1 Content safety guardrails (concrete, not aspirational)

- A **second pass, non-generative check** runs on every AI draft before it's
  even shown to the admin as ready: a regex/keyword scan reusing the exact
  banned-phrase list from `docs/CONTENT_RULES.md` §1 (this is the same
  category of defense-in-depth the app already applies to the assistant's
  canned answers — extend it to cover AI-drafted content too, don't build a
  parallel mechanism). A hit doesn't block saving the draft (it's still a
  draft, a human will read it) but **flags it visibly** in the editor
  ("⚠ contains a phrase from the banned list — review before publishing")
  and **blocks the publish action** until the admin acknowledges or edits it
  out.
- Content types with real developmental/medical adjacency (`article`,
  `milestone`, `fact`) require a non-empty `source_refs` **or** an explicit
  "no external source, editorial content" flag before they can leave
  `draft` — makes the "don't allow AI to silently invent authoritative
  medical recommendations" requirement a workflow gate, not a policy
  statement.
- `expert_reviewer` role (§"Admin roles") exists specifically so a
  `super_admin`/`editor` can route a medically-adjacent draft to someone
  qualified before publish, without that person needing full editorial
  rights.

---

## 9. Media architecture

Covered schema-wise in §5.3/6.3. Additional points:

### 9.1 Reuse, not duplication

`content_media` join table means one uploaded lullaby MP3 can back N calm-
moment entries, one instructional photo can be a game's cover **and**
appear inline in a related article. This is the brief's explicit
requirement and is the reason `media_assets` isn't a child table of
`content_items`.

### 9.2 Illustration fallback (project-specific, not generic)

Every content type that currently renders one of the ~30
`shared/illustrations/*-illustration.tsx` components keyed by category
**keeps doing so as the default** when `content_items.cover_media_id` is
null. The Admin CMS's editor shows the illustration-that-would-be-used as a
live placeholder in the cover-image field, with an explicit "Upload real
photo/art" action — this preserves the current visual identity (frozen
per this brief's instruction) for the ~430 existing records that will be
imported with no cover media at all (§12), and makes photo upload additive,
never a forced migration.

### 9.3 Video

`content_media.role = 'video'` covers uploaded video (Storage-backed,
`media_assets.media_type = 'video'`). For "external video where explicitly
allowed" (the brief's phrasing, matching the existing but unused
`Game.youtubeId` field): model it as `media_assets.external_video_provider
+ external_video_id`, **nullable and orthogonal to `storage_path`** — a
media asset is either Storage-backed (`storage_path` set,
`external_video_*` null) or external (`external_video_*` set, `storage_path`
a placeholder/empty). The content model (`content_media`) doesn't know or
care which — it just references a `media_asset_id`, exactly satisfying "do
not tightly couple the content model to YouTube." `Game.youtubeId` gets
migrated to this on import (§12) rather than kept as a bespoke field.

### 9.4 Deletion safety / orphaned assets

`content_media.media_asset_id` is `on delete restrict` (§5.3) — deleting a
referenced asset is a two-step admin action (unlink everywhere, then
delete), never a one-click accidental content break. A scheduled/manual
"orphaned media" report (`media_assets` with zero `content_media` rows,
older than N days) surfaces cleanup candidates in the Media Library UI —
simple SQL `left join ... where content_media.media_asset_id is null`, no
background job needed at this scale.

---

## 10. Assistant / RAG architecture

### 10.1 What changes in `/api/ai-chat`

The route's **contract stays identical** (`{ messages, childContext,
questionId? } → text stream`) — this was explicitly designed for this swap
(`docs/AI_COMPANION.md` §7, `ARCHITECTURE.md` §6). What changes internally:

1. Emergency-keyword check stays **first, unchanged, pre-model** — the
   existing docs are explicit this must never depend on model behavior, and
   that's correct; carry it forward verbatim.
2. On every message, persist `ai_conversations`/`ai_messages` rows (user
   message first, assistant message after generation) — this is the only
   way any of the observability/feedback-loop asks become possible.
3. Retrieval step: embed the user's message (same embedding model as
   `content_chunks`), pgvector cosine-similarity search over
   `content_chunks` filtered to the conversation's locale, top-K (start
   with K=5) chunks, joined back to their parent `content_items` for
   attribution.
4. Call the model with: the existing system prompt spirit from
   `docs/AI_COMPANION.md` §7 (identity, no-diagnosis rules, 4-block answer
   structure, red-flag deferral, locale lock) **plus** the retrieved chunks
   inserted as grounding context, explicitly instructed to answer only from
   the provided context plus general supportive tone — **not** the entire
   knowledge base (brief's explicit constraint).
5. Compute/record `confidence` — the simplest honest signal available is
   the top retrieved chunk's cosine similarity score; if it's below a
   threshold (tune empirically, start around 0.75 depending on embedding
   model), set `is_fallback = true`-equivalent behavior: the model is told
   explicitly "no confident source found" and instructed to give a general
   supportive answer *and* the response is flagged for the knowledge-gap
   queue regardless of what the model says (never rely on the model to
   self-report "I don't know" — that's a UX nicety, not a data-pipeline
   trigger).
6. Post-generation guardrail pass (already speced in `AI_COMPANION.md` §7
   closing paragraph) stays as the last step before the response is
   streamed to the client.

### 10.2 What's stored vs. not

Per the brief: conversation, message, role, question(=user message),
answer(=assistant message), retrieved sources (`retrieved_content_ids`),
confidence/retrieval metadata, feedback — all in `ai_messages` (§5.5). No
chain-of-thought, no raw model reasoning traces, no system-prompt text
stored per-message (it's derivable from `created_at` + a version-controlled
prompt template, not data).

### 10.3 Knowledge feedback loop

Exactly the brief's diagram, made concrete against the schema:

```
User asks → ai_messages row (is_fallback / low confidence / feedback = not_helpful)
    → surfaces in /admin/assistant/knowledge-gaps (query: is_fallback OR confidence < threshold OR feedback = 'not_helpful', minus ones with a knowledge_gap_resolutions row)
    → admin clicks "Create knowledge content" → pre-fills AI Content Studio with the question as context
    → AI drafts an article/fact → DRAFT → human review → PUBLISH
    → publish triggers re-chunk/embed (§10.4) → INDEX
    → next matching question retrieves it → knowledge_gap_resolutions row links the original ai_message to the new content_item (closes the loop, and lets the dashboard show "N gaps closed this month")
```

### 10.4 Re-indexing triggers

A Postgres trigger on `content_items` (`after update of status`) that, when
`status` transitions to `published`, enqueues a re-chunk job; when it
transitions **away from** `published` (archived/unpublished), deletes the
item's `content_chunks` rows immediately (synchronous delete, not queued —
cheap, and correctness here matters more than throughput: an archived
article must stop being retrievable the moment it's archived). "Enqueue a
job" at this scale can be a Supabase Edge Function invoked directly from the
trigger via `pg_net`/`supabase_functions.http_request`, or — simpler, no
extra moving part — the admin server action that performs the publish
directly calls the chunk/embed function synchronously after the DB write
succeeds, and the trigger is just a safety net for publishes that don't go
through the admin UI (there shouldn't be any, but the trigger costs nothing
to keep as a backstop). Recommend the synchronous-in-the-server-action
approach for V2.8 (simpler, no queue infra to build) with the trigger-based
backstop deferred unless direct-SQL publishes actually happen in practice.

---

## 11. User-data synchronization strategy

Most of this **already exists and works** — see §0/§1. What's left:

1. **`checklist_progress` needs the same Hybrid-repository treatment** the
   diary/progress entities already got: `LocalStorageChecklistProgress
   Repository` (refactor `use-checklist-progress.ts`'s module-level
   cache/localStorage functions into a class implementing a new
   `IChecklistProgressRepository`), `SupabaseChecklistProgressRepository`,
   `HybridChecklistProgressRepository` — mechanically identical to the
   existing two, once `checklist_items` have stable UUIDs to key against
   (§5.7, §21).
2. **The documented sync queue (`DATABASE_AND_API.md` §4) is not yet
   implemented** — today, "hybrid" means "prefer remote if a session is
   confirmed *right now*, else write local," not "queue local writes made
   while offline and replay them on reconnect." For a signed-in user who
   goes offline mid-session, a write currently silently lands in
   LocalStorage under their (still-local) child bucket and **does not
   automatically migrate to their Supabase child on reconnect** — only the
   one-time `syncLocalDataToRemote()` call at sign-in does that migration,
   and it's not re-invoked on `online` events. Building the documented FIFO
   queue (or, more simply, re-running the equivalent of
   `syncLocalDataToRemote` reconciliation on every `online` event for
   signed-in users, not just at sign-in) closes this gap. Recommend the
   simpler "re-run reconciliation on reconnect" approach over a full FIFO
   task queue unless usage data later shows conflicts that need per-
   mutation ordering — avoids building infrastructure ahead of evidence.
3. **Conflict resolution rule**: last-write-wins at the row level
   (Postgres `updated_at`), which is already implicit in every `upsert(...,
   { onConflict: "id" })` call in `sync-local-data.ts`. This is sufficient
   because the only place true concurrent edits could occur (same child,
   two devices, both offline, both write the same diary entry ID) can't
   actually happen — diary/progress entry IDs are generated client-side
   per-write, so two devices never generate the same ID for different
   content; the only "conflict" is an ID appearing twice from the same
   device's own retry, which upsert already handles correctly.
4. Anonymous→authenticated merge is already built (`syncLocalDataToRemote`)
   and correctly scoped to "remap onto the signed-in parent's first/only
   child" — matches the product's current single-child-in-practice UX.
   Flag for future revisit only if/when multi-child UI actually ships (the
   merge target would then need to ask "which child does this offline data
   belong to" rather than assuming child #1).

---

## 12. Existing-content migration strategy

1. **Write a Zod schema per content type** matching the target `data` jsonb
   shape (§5.2) — largely a direct port of the existing `entities/*/model/
   types.ts` types, since those are already well-formed and this is
   explicitly not a redesign.
2. **Importer script** (`scripts/import-content.mjs` or a one-off Node
   script, not a permanent app feature): reads every `entities/*/mock/*.ts`
   module (they're plain ES modules exporting arrays — importable directly
   in a Node/tsx script, no parsing needed), validates each record against
   its Zod schema, and for each:
   - Sets `content_items.slug = <existing mock record's slug>` — **never
     regenerate slugs**. This is the single most important migration rule:
     `completed_activities.activity_id` in the live (once-populated) DB is a
     slug, and any parent-app user who's marked a game "done" is keyed to
     that exact string forever.
   - Sets `legacy_id = <existing mock record's `id`>` for traceability
     (mock records use both an `id` and a `slug` today — keep both,
     `slug` becomes the stable public key, `legacy_id` is an audit trail).
   - For `Checklist`, also inserts each `ChecklistItem` as a
     `checklist_items` row, setting `legacy_id` = the item's current `id`
     string (these are the IDs `checklist_progress`, once built, must key
     against — see §21, this is why the importer must run **before**
     `checklist_progress` goes live with real users, not after).
   - `status = 'published'`, `published_at = now()` for everything imported
     (it's already live content, not new drafts) — except records the team
     flags for review during the audit (there is no evidence any mock
     content is currently non-compliant with `CONTENT_RULES.md`, since it
     was presumably written against those exact rules — spot-check rather
     than blanket-`review` all 430 records).
   - `Game.youtubeId`, where present (currently: nowhere, per §1, but the
     importer should handle it for forward-compatibility if it's populated
     before migration happens) → creates a `media_assets` row with
     `external_video_provider = 'youtube'` and a `content_media` link with
     `role = 'video'`.
3. **Validation pass**: after import, assert `count(content_items) ===
   count(mock records)` per type, assert every `activity_type +
   activity_id` combination that could plausibly exist in any beta user's
   `completed_activities` (there likely are none yet, pre-launch — confirm
   with the team before treating this as low-risk) resolves to a real
   `content_items` row, assert i18n parity (`ru`/`kk` both present) for
   every localized field, matching the existing manual check from
   `docs/I18N.md` §9 but automated against the DB instead of the JSON
   files.
4. **Idempotency**: importer is safe to re-run (upsert on `(content_type,
   slug)`), so it can run in CI against a staging Supabase project before
   ever touching production, and re-run after mock-data fixes discovered
   during review.
5. **Cutover**: the parent app's content-reading hooks/repositories switch
   from importing mock arrays to querying `content_items` (new
   `Supabase<Type>Repository` per content type, same Repository Pattern
   already proven for diary/progress) **only after** the importer's
   validation pass is clean on production data — feature-flag this per
   content type if a staged rollout is preferred (ship `games` from
   Postgres while `articles` still reads mocks, etc.) rather than a single
   big-bang cutover, given 8 independent content types with no
   cross-dependencies that would force simultaneity.

---

## 13. Localization strategy

**Recommendation: keep translations inline in `content_items.data` as `{
ru: ..., kk: ... }` per field — do not introduce a separate
`content_translations` table for V2.x.**

Reasoning: the existing content is small per record (an article's `content`
field is a handful of paragraphs, not a CMS-scale document), the two-locale
requirement is stable (not "eventually many locales," `en` has been
"reserved but not started" for the entire life of this project per every
doc that mentions it), and a `content_translations` join table earns its
complexity when (a) locale count grows past a handful, (b) different
locales need independent publish states (e.g. `kk` translation still in
review while `ru` is published — not a stated requirement), or (c) content
volume per record grows large enough that loading unused-locale text is a
real cost (jsonb with 2 locale keys inline is not that). None of these are
true here. If `en` is actually activated in V2.x, it's one more key in the
same jsonb shape — additive, not a schema migration.

What **does** need to move from "manual discipline" to "enforced": the
`docs/I18N.md` §9 node-snippet parity check becomes a real Zod
`superRefine` at write time in the Admin CMS (an editor cannot save a
content item with a populated `ru` field and an empty `kk` field for the
same key without an explicit "translation missing" acknowledgment) — turns
a documented convention into a UI-level guarantee.

Translation assistance in the AI Content Studio: a "Translate to kk" action
on any draft that already has `ru` content, calling the model with a
dedicated translation-only system prompt (not the content-generation
prompt) and writing the result into the `kk` half of the same draft's
`data` — stays a `draft`/`review`-gated action like any other AI output,
never auto-published, satisfying "translations must remain reviewable."

---

## 14. Security model

- **Admin auth**: covered in §6.1 — closed admin population via
  `admin_users`, server-side role checks on every `/admin` route and every
  admin server action, never a client-side-only route guard.
- **RLS**: covered per-table in §5.8/§6.4.
- **Storage policies**: covered in §6.3 — public read, admin-only write,
  per-bucket MIME/size limits.
- **API validation**: every admin write goes through a Zod schema matching
  the content type (§5.2, §8) — this is not new practice, it's the same
  pattern `/api/ai-chat` already uses today, extended to the CMS surface.
- **Upload validation**: MIME allowlist + size limit, client- and Storage-
  policy-enforced (§6.3); image uploads are re-encoded server-side via
  `sharp` (already a dependency) rather than trusting the uploaded file's
  claimed type — closes the classic "renamed .php as .jpg" class of issue
  even though this app has no PHP anywhere, the principle (never trust a
  client-declared MIME type for anything that gets served back) still
  applies to `next/image` remote-pattern trust.
- **Rate limiting**: `/api/ai-chat` currently has none — becomes necessary
  the moment it costs real LLM tokens per call (V2.8). Simplest viable
  option at this scale: a per-`profile_id` (or per-IP for anonymous, though
  the assistant already requires no auth today — confirm whether anonymous
  chat should even continue once conversations are persisted and tied to a
  profile, since anonymous+persisted is an odd combination) sliding-window
  counter in Postgres or Upstash Redis if one gets provisioned; do not
  build this ahead of actually wiring the real LLM (no cost exists to rate-
  limit against yet).
- **AI endpoint abuse**: the structured-output constraint in the Content
  Studio (§8) is itself a mitigation — an admin cannot use it as a general-
  purpose chat endpoint to extract arbitrary model output, since the API
  only accepts the fixed parameter forms and only returns schema-validated
  structured data.
- **Sensitive user data / admin-only assistant logs**: `ai_messages` RLS
  (§5.8) explicitly separates "admin can read for observability" from a
  blanket admin-bypasses-everything policy — this is the one place the
  brief calls out as sensitive ("Do NOT expose private Diary content to
  administrators by default") and the same caution extends to assistant
  conversation content, which can contain equally personal detail about a
  child. Diary entries themselves get **no** admin-read policy at all —
  not even restricted — matching the brief exactly.
- **Users section privacy**: the Admin Users screens (§"Users" IA) surface
  aggregates and metadata (registration date, last activity, child age
  ranges, completed-activity counts, checklist progress %, AI usage count)
  — never diary text, never raw AI message content by default (a
  `/admin/assistant/conversations` drill-down is a *separate*, explicitly-
  entered screen with its own audit-logged access, not something visible
  from a user's profile page).

---

## 15. Recommended implementation phases

Adjusted from the brief's suggested V2.0–V2.9 to reflect that **V2.0's core
user-data backend is already ~70% built** (§0). Renumbered to avoid
implying work that doesn't need doing.

### V2.0 — Backend Foundation Completion *(small — mostly ops, not code)*
Provision an actual Supabase project (none exists today), apply the
existing migration, wire real env vars, set up `supabase/config.toml` +
CLI migration workflow (`BACKLOG.md` #4), add the `profiles`
auto-creation trigger on `auth.users` (currently done ad hoc in
`ensureProfile()` — move to a DB trigger so it holds even for writes the
app itself doesn't make). Add `admin_users`/`audit_log` tables and the
server-side admin-route guard scaffold (no admin *features* yet, just the
gate). **This is prerequisite to every later phase touching Supabase.**

### V2.1 — Content Schema + Admin Shell
`content_items`, `checklist_items`, `content_versions`, RLS for both
(§5.2, §5.8). `/admin` route group, layout, auth guard, empty Dashboard,
left nav. No content editing yet — this phase is the skeleton.

### V2.2 — Media Library
`media_assets`, `content_media` (§5.3), Storage buckets + policies (§6.3),
upload UI, thumbnail generation. Built before content editors need to
attach media to anything.

### V2.3 — Content CRUD + Lifecycle
Per-type editor forms (start with `games` and `facts` — smallest schemas,
prove the pattern — then the rest), draft/review/publish/archive workflow,
version history + restore, illustration fallback (§9.2).

### V2.4 — Existing-Content Migration
Importer script (§12), validation pass against a staging project, dry run,
then production import. Parent app's content repositories switch from mock
arrays to Supabase reads, content-type by content-type (feature-flaggable
per type per §12.5).

### V2.5 — Users & Missing Progress Data
`checklist_progress` table + Hybrid repository (§5.7, §11.1), sync-queue
gap fix (§11.2), `/admin/users` list + detail (aggregates only, §14).

### V2.6 — AI Content Studio
Wire the actual Anthropic API call, per-type system prompts embedding
`CONTENT_RULES.md`, structured-output schema binding, `ai_generations`
table, guardrail second-pass, expert-reviewer routing (§8).

### V2.7 — Real Assistant + Observability
Replace `getAssistantReply()` mock with a real model call (system prompt
already fully speced in `docs/AI_COMPANION.md` §7 — reuse verbatim),
`ai_conversations`/`ai_messages` persistence, feedback UI in the parent
app's chat, `/admin/assistant` (conversations/questions/feedback) screens.
**Ships without RAG first** — a real model call grounded only in the
system prompt is already a large upgrade over 16 canned answers, and
decoupling this from RAG means the assistant improvement doesn't wait on
the vector pipeline.

### V2.8 — RAG + Knowledge Feedback Loop
`content_chunks`/embeddings, retrieval step wired into `/api/ai-chat`,
`/admin/assistant/knowledge-gaps` queue, "Create knowledge content" →
Content Studio handoff, publish-triggers-reindex (§10.4).

### V2.9 — Daily Recommendations (curated) + Analytics + Notifications groundwork
`daily_recommendation_overrides` (§5.6), `/admin/recommendations` calendar
UI, minimal analytics dashboard (§"Analytics" scope below), Notifications
settings stub (architecture note only, per brief — no send capability).

Rate limiting (§14) and localization-parity enforcement (§13) are cross-
cutting and should land alongside V2.6/V2.7 (whichever ships the LLM cost
first) and V2.3 (first content editor) respectively, not as their own
phases.

---

## 16. Risks

- **Slug/ID stability is the single highest-risk item.** Getting it wrong
  in V2.4 silently orphans real user progress with no error thrown anywhere
  — it just looks like "my checklist reset." Mitigate with the validation
  pass in §12.3 and by treating slug changes as a reviewed, logged
  operation forever after (never a plain field edit in the CMS — a rename
  action that explicitly asks "this will break progress tracking for users
  who completed this under the old slug, continue?").
- **`ai_generated` reaching `published` without review** is the brief's
  named top content-safety risk. Mitigate at the DB/API layer (§8.3), not
  just UI copy, since UI-only gates are exactly what the brief's security
  section warns against ("never exposed merely by hiding UI routes" — same
  principle applies to workflow states, not just route access).
- **Embedding model lock-in**: `vector(1536)` (or whatever dimension is
  chosen) is baked into the schema; switching embedding providers later
  means a full re-embed + index rebuild, not a config change. Pick
  deliberately in V2.8, document the choice in the migration file.
- **Two parallel localization mechanisms** (UI `next-intl` vs. content
  inline `{ru,kk}`) is an accepted, scoped risk (§13) — the risk is a
  future contributor conflating them, not a technical flaw. Mitigate with a
  clear doc note (this document + an update to `I18N.md`).
- **No CI today.** Every phase above assumes `npm run typecheck`/`lint`/
  `build` gate merges, but nothing currently enforces that automatically
  (`BACKLOG.md` #2 already flags the i18n-parity case specifically). Stand
  up CI before V2.1's admin surface starts growing — an admin tool with
  broken types is worse than a broken parent-facing page (it's how content
  corruption reaches production silently).
- **Single admin population, no `expert_reviewer` seeding plan yet** — the
  role exists in the schema (§5.1) but who actually holds it (a real
  pediatrician/speech therapist on contract or volunteer?) is a staffing
  question outside this document's scope; flag to the team now so V2.6
  doesn't stall waiting on it.

---

## 17. Explicit non-goals

- No monorepo split (§3) — revisit only if a stated trigger from that
  section occurs.
- No Supabase Realtime — nothing here needs live multi-viewer sync.
- No full event-sourced content history — snapshot versioning (§5.2) is
  sufficient per the brief's own instruction not to over-engineer this.
- No `content_translations` table (§13) at 2-locale scale.
- No enterprise RBAC beyond `super_admin`/`editor`/`viewer`/
  `expert_reviewer` (§ below) — no per-content-type or per-field
  permissions.
- No FIFO sync-queue infrastructure unless reconciliation-on-reconnect
  (§11.2) proves insufficient in practice.
- No native push notifications in this phase set — architecturally
  acknowledged (a `Settings → Notifications` stub, a `device_tokens` table
  reserved but not created until needed) and nothing more, per brief.
- No general-purpose chat/prompt box anywhere in the Admin CMS (§8's
  explicit instruction).
- No vanity-metric analytics platform (§"Analytics" below is intentionally
  minimal).
- No ffmpeg/video-processing pipeline in V2.2 — manual poster-frame upload
  is the accepted V2.2 answer; revisit only if video volume justifies it.

### Admin roles (concrete permission matrix)

| Action | `viewer` | `editor` | `super_admin` | `expert_reviewer` |
|---|---|---|---|---|
| View content (any status) | ✅ | ✅ | ✅ | ✅ (assigned items only) |
| Create/edit draft content | ❌ | ✅ | ✅ | ❌ |
| Submit for review | ❌ | ✅ | ✅ | ❌ |
| Approve/reject review | ❌ | ❌ | ✅ | ✅ (assigned items only) |
| Publish/archive | ❌ | ❌ | ✅ | ❌ |
| Run AI Content Studio | ❌ | ✅ | ✅ | ❌ |
| Manage media | ❌ | ✅ | ✅ | ❌ |
| View users/analytics | ✅ | ✅ | ✅ | ❌ |
| View assistant conversations | ❌ | ✅ | ✅ | ❌ |
| Manage admin users/roles | ❌ | ❌ | ✅ | ❌ |
| Manage daily recommendations | ❌ | ✅ | ✅ | ❌ |

### Analytics (minimum useful set, operational vs. vanity separated)

**Operational** (drives actual decisions): registered users, WAU/MAU,
content-item completion counts (from existing `completed_activities`,
already queryable today with zero new schema), checklist completion rates,
per-content-type popularity (top 10 games/articles by completion count),
Calm Moments open-rate, AI questions per day, fallback/low-confidence rate
(the knowledge-gap signal), feedback ratio (`helpful`/`not_helpful`).

**Explicitly not built**: session-length heatmaps, funnel-visualization
tooling, cohort-retention dashboards, or any dedicated analytics service
integration (Umami/Mixpanel, `BACKLOG.md` #3) — that item is real and
valid but is product-analytics for the parent app, orthogonal to the Admin
CMS's own operational dashboard, and out of scope here.

---

## 18. Estimated complexity per phase

Rough, directional (not a committed estimate) — “S/M/L/XL” relative sizing
given this codebase and team shape (per `CLAUDE.md`, effectively an
AI-assisted solo/small-team build):

| Phase | Complexity | Why |
|---|---|---|
| V2.0 | S | Mostly provisioning + one trigger; code already exists. |
| V2.1 | M | New route group, auth guard, nav shell — no novel data logic. |
| V2.2 | L | Storage policies, upload UX, thumbnailing are genuinely fiddly. |
| V2.3 | XL | 8 content types × form + workflow + versioning + preview. |
| V2.4 | L | Importer is mechanical; validation rigor is what takes care. |
| V2.5 | M | One new table/repository (pattern already proven twice) + users UI. |
| V2.6 | L | Prompt engineering + structured-output plumbing + guardrails. |
| V2.7 | L | Real LLM wiring is simple; persistence + feedback UI is the bulk. |
| V2.8 | XL | Embeddings pipeline + retrieval tuning is the hardest technical work in this whole plan. |
| V2.9 | M | Calendar UI + a handful of read-only analytics queries. |

---

## 19. Files/modules that will need to change

- `src/entities/*/model/types.ts` (all 8 content-bearing entities) — become
  the source for the Zod schemas in §5.2/§8, likely restructured into
  `entities/*/model/schema.ts` (Zod) with `types.ts` deriving `z.infer<...>`
  types, matching the pattern `entities/auth/model/schema.ts` already
  establishes elsewhere in the codebase.
- `src/entities/*/mock/*.ts` — retired after V2.4's cutover per type (kept
  as the importer's input until then; delete only after production
  validation passes, not before).
- `src/entities/*/model/repository.ts` (new for the 6 types that don't have
  one yet — today only `diary-entry` and `progress` have this interface;
  `article`, `game`, `checklist`, `milestone`, `fact`, `tip`, `calm-moment`,
  `daily-recommendation` currently read mocks directly with no repository
  seam at all) — needs to be introduced, not just swapped, for those types.
- `src/features/checklists/model/use-checklist-progress.ts` — refactored
  into the Hybrid-repository pattern (§11.1).
- `src/app/api/ai-chat/route.ts` — internals replaced per §10.1, contract
  unchanged.
- `src/features/ai-chat/api/mock-assistant.ts` — retired once V2.7 ships
  (keep as a local dev fallback when `isSupabaseConfigured`/LLM key is
  absent, mirroring how the app already treats `isSupabaseConfigured`
  elsewhere — never break local dev for someone without API keys).
- `next.config.ts` — `images.remotePatterns` needs the Supabase Storage
  project hostname added (and the unused `images.unsplash.com` entry can be
  removed as part of this change, or left — harmless either way).
- `.env.example` / `.env.local` — new keys: `SUPABASE_SERVICE_ROLE_KEY`
  (server-only, never `NEXT_PUBLIC_*`), `ANTHROPIC_API_KEY`, embedding
  provider key if different from the chat model provider.
- `docs/DATABASE.md`, `docs/DATABASE_AND_API.md`, `docs/ARCHITECTURE.md`,
  `docs/CONTENT.md`, `docs/AI.md`, `docs/AI_COMPANION.md`, `docs/ROADMAP.md`
  — all need updates once implementation starts, since several already
  describe a pre-V2.0 state that's stale relative to the code (§0). Treat
  "update docs to match reality" as its own small task before V2.1 starts,
  independent of new-feature work.

---

## 20. New files/modules to introduce

- `src/app/admin/**` — the entire Admin CMS route tree (§7), outside the
  `[locale]` segment.
- `src/entities/content-item/` — new shared entity for the polymorphic
  content model (Zod schema registry keyed by `content_type`, shared
  between admin editors and the AI Content Studio's structured-output
  binding).
- `src/entities/media-asset/`
- `src/entities/admin-user/`
- `src/features/admin-content-editor/`, `admin-media-library/`,
  `admin-ai-studio/`, `admin-users/`, `admin-assistant-observability/`,
  `admin-daily-recommendations/`, `admin-audit-log/`
- `src/shared/api/supabase/admin-server-client.ts` — the service-role
  server-only client, explicitly separate from the existing anon-key
  `client.ts`/`server.ts`, with a lint rule or code comment making clear it
  must never be imported into any client component.
- `src/shared/ai/` — Anthropic API client wrapper, per-content-type prompt
  templates, structured-output schema binding shared between the Content
  Studio and the real assistant.
- `supabase/migrations/*` — one migration per schema phase above (§5),
  following the existing single-file-per-change convention.
- `scripts/import-content.mjs` (§12).
- `scripts/embed-content.mjs` or a Supabase Edge Function (§10.4),
  depending on the synchronous-vs-queued decision made at V2.8 time.

---

## 21. Migration/backward-compatibility concerns

1. **Slug stability** — covered extensively above (§12, §16); repeating
   here because it is the one item that, if missed, causes silent user-
   facing data loss with no exception ever thrown.
2. **Checklist item ID stability** — identical concern, one level down:
   `checklist_items.legacy_id` must be set from the *existing* mock
   `ChecklistItem.id` values during import (§12.2), and `checklist_progress`
   (§5.7, net-new — no existing users to break yet, but future edits to a
   checklist's items must never renumber/reuse an existing item's UUID).
3. **`ChildGender`/DB gender mismatch** (`unspecified` vs. `other`) already
   has a working mapping layer (§1) — no new work, just don't remove it
   under the assumption "the CMS makes this obsolete." It doesn't; it's a
   client-type vs. DB-type boundary, unrelated to content.
4. **`isSupabaseConfigured` semantics must not change for the parent app.**
   The parent app must continue to run fully offline with zero backend
   configured (a deliberate, documented product property — `ROADMAP.md`
   V1.5). Once content lives in Postgres, "no backend configured" needs an
   explicit answer: either (a) ship a small bundled fallback content set for
   the fully-offline case (defeats some of the CMS's purpose but preserves
   the guarantee), or (b) accept that "zero backend" now means "app shell
   only, no content" and update that documented guarantee accordingly. This
   is a product decision, not a technical one — flag to the team before
   V2.4's cutover, don't decide it implicitly by however the code happens
   to fall back.
5. **`Game.youtubeId`** — currently unused, but a live field on a currently
   TypeScript-typed structure; if any external tooling or a not-yet-merged
   branch populates it before migration, the importer (§12.2) must handle
   it, not silently drop it.
6. **`docs/*.md` staleness** — several docs already describe a past-tense
   state incorrectly (§0). Anyone starting V2.x work from the docs alone,
   without reading this audit, will underestimate what's already built and
   may duplicate the existing Hybrid-repository/phone-auth work. Flag this
   loudly to whoever picks up V2.0.

---

## 22. Testing strategy

Matches the project's current near-zero automated-test baseline
(`docs/BACKLOG.md` #11/#12 flag this as an existing gap, not something this
plan invents) — recommend closing the gap specifically where the CMS adds
the highest blast-radius risk, not everywhere at once:

- **Importer (§12)**: script-level tests asserting record-count parity,
  slug uniqueness, i18n-field completeness, and — critically — a
  golden-file diff of "every slug that exists in the mock data before vs.
  every slug in `content_items` after," so a future mock-data edit can't
  silently rename a slug without the test failing.
- **RLS policies**: Supabase's own `pgTAP`-based testing (or hand-rolled
  SQL assertions run in CI against a local Supabase instance) for every new
  table in §5 — specifically, a non-admin user must not be able to read
  `draft` content or another child's `checklist_progress`, and an admin
  with `viewer` role must not be able to write anything. This is the
  highest-value test category given RLS is the actual security boundary.
- **Publish workflow**: an integration test asserting `ai_generated=true`
  content can never reach `published` status without passing through
  `review` — this is the brief's single most emphasized safety
  requirement, so it should be the one guardrail with an explicit
  regression test, not just a code-review convention.
- **Content-safety regex pass (§8.1)**: unit tests against the exact banned-
  phrase list from `CONTENT_RULES.md` §1, asserting each listed phrase is
  actually caught — turns that document from a human checklist into an
  enforced one for AI output specifically (human-authored content still
  relies on editorial review, which is out of scope for automated testing).
- **Repository parity**: for each new Hybrid repository (`checklist-
  progress`, and any content-type repository that gains an offline
  fallback), the same kind of test the existing diary/progress repos would
  benefit from per `BACKLOG.md` #11 — write via Local, write via Supabase
  mock, assert both produce the same shape.
- Parent-app UI regression (visual/E2E) stays out of scope for this plan
  specifically because the brief and this session's framing freeze UI/UX —
  the Admin CMS is new UI and doesn't have a "no redesign" constraint, so
  it's reasonable to hold it to a normal testing bar without implying the
  frozen parent app needs new UI tests as a side effect of this work.

---

## 23. Definition of Done per phase

| Phase | Done when |
|---|---|
| V2.0 | Live Supabase project provisioned; existing migration applied; `admin_users`/`audit_log` tables exist; `profiles` auto-creation is a DB trigger, not app code; `/admin` route group renders a "you are not an admin" page correctly for a non-admin session and nothing for a logged-out session (server-side, verified by direct URL hit, not just nav-hiding). |
| V2.1 | An `admin_users` row with `super_admin` role can log into `/admin` and see an (empty) Dashboard and left nav; a `viewer`-role row can log in but sees read-only chrome; RLS tested per §22. |
| V2.2 | An admin can upload an image, video, and audio file of allowed types/sizes; rejected for disallowed MIME/size; thumbnail generated for images; asset appears in `/admin/media`; deleting a referenced asset is blocked with a clear message. |
| V2.3 | For at least `games` and `facts`: full draft→review→publish→archive cycle works end to end in the UI; a version can be restored; a published item is visible (read-only) to a non-admin session; an unpublished/archived item is not. |
| V2.4 | All 8 content types imported; validation pass (§12.3) green on production data; parent app reads at least one content type from Postgres in production with feature-flag rollback available; zero `completed_activities` rows become orphaned (verified by the join-check in §12.3). |
| V2.5 | `checklist_progress` live with Hybrid repository; a signed-in user's checklist ticks persist across devices; reconnect-reconciliation (§11.2) verified by an offline→online manual test; `/admin/users` shows real aggregate data for at least one real or seeded account. |
| V2.6 | An admin can generate a structured draft for at least 3 of the 8 content types via the Studio; guardrail flag (§8.1) demonstrably fires on a deliberately-bad test prompt; publish is blocked on an unreviewed AI draft (tested per §22). |
| V2.7 | `/api/ai-chat` calls a real model, contract unchanged from the parent app's perspective; every exchange persisted; feedback thumbs-up/down UI works and writes `ai_messages.feedback`; `/admin/assistant` shows real conversations/feedback. |
| V2.8 | A question matching published content returns a grounded, cited answer; a question with no good match is flagged into the knowledge-gap queue automatically (no manual tagging); "Create knowledge content" from a gap pre-fills the Studio; publishing the resulting article visibly changes a subsequent identical question's answer (manually verified end-to-end at least once). |
| V2.9 | An admin can set a curated recommendation for a specific date/audience and confirm it (not the deterministic pick) is what the parent app shows on that date; fallback to `auto` confirmed for a date with no override; Dashboard shows the operational metrics list from §17 with real numbers. |
