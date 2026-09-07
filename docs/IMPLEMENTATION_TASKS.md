# IMPLEMENTATION_TASKS.md — CMS + Admin + Backend build tracker

Working checklist for the build described in
[CMS_BACKEND_ARCHITECTURE.md](CMS_BACKEND_ARCHITECTURE.md). Updated as work
lands — every `[x]` below has been exercised end-to-end against a real
local Supabase stack (`npx supabase start`, Docker-based), not just
written and typechecked. Local dev is NOT a live cloud project (none
exists; provisioning one requires the team's own Supabase account,
flagged wherever it blocks a task).

Legend: `[x]` done & verified · `[~]` in progress / partial ·
`[ ]` not started · `[BLOCKED: reason]` needs something only the team can
provide (an API key, a cloud account).

## V2.0 — Backend foundation completion — DONE

- [x] Supabase CLI + local stack (`npx supabase start`), existing
      migration verified to apply cleanly.
- [x] `handle_new_user` trigger — auto-creates `profiles` on signup.
- [x] `admin_users` + `audit_log` tables, RLS. **Found and fixed a real
      bug during testing**: the first version of `admin_users`' own
      "super_admin can manage all rows" policy queried `admin_users`
      recursively and Postgres threw `infinite recursion detected in
      policy` — fixed by introducing `public.is_admin()`, a
      `security definer` helper that bypasses RLS for exactly this
      check. Every other admin-domain policy uses the same helper.
- [x] `public.is_admin(roles[])` helper — the single admin-role check
      used by every later migration's RLS policies.
- [x] Normalized `children.gender` / `diary_entries.category` /
      `completed_activities.activity_type` from `text + check` to native
      Postgres enums — the original hand-written `Database` type stand-in
      had these as literal unions, but `supabase gen types typescript`
      only infers literal unions from real enum types, not `check`
      constraints. Needed once real codegen replaced the hand-written stub.
- [x] Real generated `Database` type (`src/shared/types/database.ts`,
      `npm run supabase:types`) replacing the hand-written stand-in.
- [x] Server-only service-role client
      (`shared/api/supabase/admin-server-client.ts`) — used by the
      importer script; regular admin CRUD deliberately uses the
      cookie-bound client instead, so RLS stays the real enforcement
      boundary (defense in depth) rather than being bypassed by default.
- [x] **Corrected a mistake from the original architecture-audit pass**:
      that document claimed no server-side session handling existed
      because no `middleware.ts` was found. Next.js 16 renamed
      `middleware.ts` → `proxy.ts`, which this project already had
      (`src/proxy.ts`), already calling `updateSupabaseSession()`. Fixed
      by extending the real `proxy.ts` (skip next-intl routing for
      `/admin`, keep the session refresh) instead of adding a duplicate
      file, and corrected the architecture doc's inventory table.
- [x] `.env.local` / `.env.example` updated with `SUPABASE_SERVICE_ROLE_KEY`
      and `ANTHROPIC_API_KEY` (documented, not set).
- [x] `scripts/seed-local-admin.mjs` + `npm run seed:admin` — reusable
      local-dev helper to create a super_admin account.
- [ ] `[BLOCKED: needs the team's Supabase account]` Provision a real
      cloud Supabase project (staging + production), `supabase link` +
      `supabase db push`. Every migration in `supabase/migrations/*.sql`
      has been verified against a real (local) Postgres — pushing them to
      a cloud project is a `supabase db push` away, no migration content
      changes expected.
- [ ] `[BLOCKED: needs an Anthropic API key]` `ANTHROPIC_API_KEY` for
      real AI Content Studio / assistant calls (V2.6/V2.7).

## V2.1 — Content schema + Admin shell — DONE

- [x] Migrations: `content_items`, `checklist_items`, `content_versions`,
      `media_assets`, `content_media`, storage policies,
      `checklist_progress`, `ai_conversations`/`ai_messages`/
      `ai_generations`/`knowledge_gap_resolutions`/`content_chunks`
      (pgvector), `daily_recommendation_overrides` — all applied and
      RLS-tested locally.
- [x] **Found and fixed a second real gap during Users-page work**: the
      original `init_schema` migration scoped every user-data RLS policy
      to "own data only", correctly for a pre-admin app, but that leaves
      the admin's Users section unable to read anything. Added
      `20260906120800_admin_user_visibility.sql` granting admin read-only
      access to `profiles`/`children`/`completed_activities`/
      `checklist_progress` — deliberately **not** `diary_entries`, per
      the brief's explicit "never expose diary content to admins by
      default."
- [x] `/admin` route tree: root layout (own `<html>`, reuses the parent
      app's design tokens/ThemeProvider, no next-intl), `/admin/login`
      (handles signed-out, signed-in-as-non-admin, and signed-in-as-admin
      states distinctly), `(protected)` route group with a server-side
      `requireAdminSession()` guard, left nav, dashboard with real content
      counts + registered-user count.
- [x] `entities/content-item`, `entities/media-asset`, `entities/admin-user`
      (Zod schemas in `shared/content/schemas.ts` + types).
- [x] End-to-end verified in-browser: login → role check → dashboard →
      content CRUD → publish, all against the real local stack.

## V2.2 — Media library — DONE (core), verified via API-level test

- [x] Storage buckets declared in `supabase/config.toml` + created
      idempotently in a migration; admin-write/public-read policies.
- [x] **Verified via direct Supabase-client script** (not just code
      review, since the sandboxed browser can't drive a native file
      picker): anonymous upload → rejected by RLS; authenticated
      super_admin upload → succeeds; public unauthenticated read →
      200. This is the security-critical mechanism and it works as
      designed.
- [x] `/admin/media` — upload form (title/alt-text ru+kk/tags), grid with
      type filter, delete-with-reference-guard
      (`getMediaAssetReferenceCount`), thumbnail generation for images via
      `sharp` (already a project dependency).
- [~] The upload **form's own submit path** (FormData → server action)
      uses the same Supabase client calls verified above, but wasn't
      driven through an actual `<input type=file>` in the browser (sandbox
      limitation, not a code gap) — worth a manual click-through once a
      real browser is available.

## V2.3 — Content CRUD + lifecycle — DONE

- [x] Zod schemas for all 8 content types
      (`shared/content/schemas.ts`) + declarative per-type form config
      (`shared/content/field-config.ts`) driving ONE generic admin editor
      component (`features/admin-content-editor`) — not 8 bespoke forms.
- [x] Server actions: create / update / transition-status / restore-version
      (`entities/content-item/api/admin-content-actions.ts`), checklist
      item CRUD (`admin-checklist-item-actions.ts`).
- [x] Draft → Review → Published → Archived workflow. **Verified live**:
      created a game draft, submitted for review, published it, confirmed
      it became publicly readable via the anon key.
- [x] AI-content safety guardrail (`shared/content/safety-guardrails.ts`,
      phrases ported from `docs/CONTENT_RULES.md` §1) — **verified live**:
      seeded an `ai_generated` draft containing "музыка развивает
      интеллект", moved it to review, attempted to publish — blocked with
      a clear error, row confirmed still in `review` status via direct DB
      query. The `draft → published` direct transition is additionally
      impossible for any content (not just AI) since the state machine
      only allows `draft → review | archived` — the AI-specific check in
      the code is now provably redundant defense-in-depth, not the only
      guard.
- [x] Version history + restore.
- [x] Illustration fallback: cover-less content continues to use the
      existing `shared/illustrations/*` components in the parent app,
      untouched.
- [x] Found and fixed a real UI bug during testing: the `(protected)`
      layout used `min-h-screen` with two independent `overflow-y-auto`
      containers, which fought each other for scroll. Fixed to `h-screen
      overflow-hidden` on the outer flex row.

## V2.4 — Existing-content migration — DONE (import), PARTIAL (app cutover)

- [x] `scripts/import-content.ts` (run via `tsx` — Node's native TS
      stripping can't resolve the mock files' extensionless relative
      imports, `tsx` can) — imports all 8 content types plus
      `checklist_items`, preserving `legacy_id`, upserting on
      `(content_type, slug)` (idempotent — verified by running it twice
      and confirming identical row counts).
- [x] **Real import run against the local stack**: 403 content_items (95
      games, 7 daily games, 16 articles, 100 facts, 100 tips, 29
      milestones, 50 calm moments, 6 checklists) + 31 checklist_items, all
      `published`. Milestones had no `slug` field in the mock data at all
      — discovered during implementation, resolved by reusing each
      milestone's existing `id` (already lowercase-kebab-case, e.g.
      "m-phys-1") as its slug.
- [x] Parent-app cutover, done and verified for **games, daily games,
      facts, checklists** (the four simplest/no-cross-dependency types):
      new read-only repositories
      (`entities/{game,fact,checklist}/model/content-repository.ts`,
      sharing `shared/content/read-published-content.ts`) that read
      published Postgres content when configured and fall back to the
      bundled mock array otherwise (same `isSupabaseConfigured` pattern
      used everywhere else in this app) — **verified live in-browser**:
      games list, game detail (incl. mark-as-completed writing to the
      existing `completed_activities` table via the content's slug),
      daily games, checklists list, checklist detail, and toggling
      checklist-item progress (which still uses `legacy_id`-preserved
      string ids, so the existing LocalStorage-based
      `useChecklistProgress` needed zero changes).
  - `games/[slug]` and `checklists/[slug]` lost `generateStaticParams` —
    they're now server-rendered on demand (ƒ) instead of statically
    generated at build time, which is the correct tradeoff once content
    is admin-editable. Confirmed via the build output (route count
    dropped from 279 prerendered pages to 90).
- [x] **All 8 content types now cut over** — `article`, `tip`,
      `milestone`, `calm_moment` followed the same mechanical pattern as
      `game`/`fact`/`checklist`
      (`entities/{article,tip,milestone,calm-moment}/model/
      content-repository.ts`), and
      `features/daily-recommendations/lib/get-today-recommendations.ts`
      (the home-page "Today" feed, the single highest-traffic surface in
      the app) is now fully async and reads all 6 of its content pools
      from Postgres in parallel. **Verified live**: home page renders
      Today's game/fact/article, the "Continue" checklist card correctly
      shows "1 из 5" reflecting the same LocalStorage progress set earlier
      in the games-detail test, tip-of-the-day, calm-moment-of-the-day,
      and the "may be useful this week" article list — all from the
      403-row import, not the bundled mocks.
  - `article/[slug]` also lost `generateStaticParams` for the same reason
    as `games`/`checklists`.
  - `dailyRecommendation` (the one-sentence snippet, distinct from the 8
    CMS content types) deliberately stays on its mock array — see
    docs/CMS_BACKEND_ARCHITECTURE.md §5.6; it was never in scope for the
    content migration, only for a future curated-override layer.
  - Every repository falls back to its mock array when
    `isSupabaseConfigured` is false or a query fails — the app still runs
    fully offline/misconfigured exactly as before, this is additive.

## V2.5 — Users & missing progress data — PARTIAL

- [x] `/admin/users` list + detail — aggregates only (locale, child count,
      completed-activities count, checklist-items-done count), explicitly
      no diary/AI-conversation content, per the brief.
- [ ] `HybridChecklistProgressRepository` mirroring the existing
      diary/progress Hybrid pattern — **deliberately not started**: it
      needs the *real* `checklist_items.id` (UUID), not the
      `legacy_id`-preserving id the parent app currently renders (see
      V2.4 note above) — building it now would mean either exposing two
      different ids per checklist item to the frontend or reworking
      `useChecklistProgress`'s signature. Cleaner to do this once
      `checklist_progress` is actually needed by a feature, not
      speculatively.
- [ ] Reconnect-reconciliation for the documented sync-queue gap
      (`docs/CMS_BACKEND_ARCHITECTURE.md` §11.2) — untouched this pass.

## V2.6 — AI Content Studio — WORKFLOW DONE, real model call unverified

- [x] `/admin/ai-studio`: content-type picker, structured parameter form
      (age range, developmental goal, category, duration, difficulty,
      materials, tone, brief) — not a freeform chat box, per the brief's
      explicit instruction. `shared/ai/generate-content-draft.ts` builds a
      per-type system prompt embedding `docs/CONTENT_RULES.md` §1-2
      verbatim, and derives the tool-use `input_schema` directly from the
      same Zod schema (`z.toJSONSchema()`, native in this Zod version) the
      admin editor and importer already validate against — one source of
      truth for the shape.
- [x] **Fully wired and verified live end-to-end** via the template
      fallback (`isAiConfigured` false ⇒ no LLM call, returns a
      correctly-shaped empty draft instead — same pattern as
      `isSupabaseConfigured` elsewhere): generated a Games draft from the
      Studio, confirmed it landed as `draft` + `ai_generated: true` with a
      real `ai_generations` row (model: "template-fallback...", full
      `input_params` logged), redirected into the normal content editor,
      confirmed the guardrail scan and draft→review→publish gate both
      still apply to AI-sourced drafts exactly like hand-authored ones.
- [ ] `[BLOCKED: needs ANTHROPIC_API_KEY]` The real (non-fallback)
      Anthropic call in `generate-content-draft.ts` has **not been
      exercised against the live API** in this environment — it's written
      to the documented Messages API tool-use contract (pinned model:
      `claude-sonnet-5`) but needs a manual smoke test the first time a
      real key is configured. Treat it as "written, not proven" until then.

## V2.7 — Real assistant + observability — OBSERVABILITY DONE, real LLM call blocked

- [x] **Observability layer wired and verified live**, independent of
      whether the assistant itself is the deterministic mock or a real
      model — persistence doesn't care which produced the reply.
      `/api/ai-chat` now persists every exchange for signed-in parents
      (`features/ai-chat/api/persist-conversation.ts`) into the real
      `ai_conversations`/`ai_messages` tables, tagging `is_fallback` /
      `is_emergency`. Anonymous/guest chat (explicitly supported per
      `docs/REQUIREMENTS.md` §1.1) is deliberately NOT persisted — see the
      doc comment for why persisting it would need a product decision this
      pass doesn't make.
  - **Found and fixed a real bug during testing**: the first version
    batched the user+assistant message insert into one `.insert([...])`
    call; Postgres' `now()` is constant per transaction, so both rows got
    an *identical* `created_at`, making it impossible to tell which user
    message a flagged assistant reply answered. Fixed by splitting into
    two separate insert calls (two transactions, two real timestamps).
  - `/admin/assistant` (conversation list + fallback/negative-feedback
    flags), `/admin/assistant/conversations/[id]` (full transcript),
    `/admin/assistant/knowledge-gaps` (the queue from §10.3: fallback OR
    negative feedback, minus resolved) + a working "Mark resolved" action
    (`resolveKnowledgeGap`) — **verified live end-to-end**: sent an
    off-topic message in the parent chat, confirmed it hit the generic
    fallback, confirmed it appeared correctly in the admin dashboard
    (1 conversation, 1 fallback, 1 open gap) and the knowledge-gaps queue
    with the actual question text (not the assistant's own reply), marked
    it resolved, confirmed it disappeared from the open queue.
- [ ] **No parent-facing feedback UI (thumbs up/down) yet** — deliberately
      deferred: the streaming `/api/ai-chat` response is currently plain
      text with no message-id round-tripped to the client, so there's
      nothing for a feedback click to reference yet without changing the
      response protocol. Worth doing as its own small slice: have the
      route return the persisted `ai_messages.id` (e.g. as a leading
      metadata line or a trailing header) so the client can call a
      feedback server action against it.
- [ ] `[BLOCKED: needs ANTHROPIC_API_KEY]` The assistant itself is still
      `getAssistantReply()` (deterministic mock) — swapping it for a real
      model call is the one piece of V2.7 that needs a key, per the
      already-written system prompt in `docs/AI_COMPANION.md` §7.

## V2.8 — RAG + knowledge feedback loop

- [ ] Not started. `content_chunks` (pgvector, HNSW index) exists and is
      tested at the schema level (extension installs, index creates) but
      has no embeddings pipeline yet — depends on V2.7 shipping first per
      the architecture doc's sequencing rationale.

## V2.9 — Curated daily recommendations + analytics + notifications groundwork — recommendations DONE

- [x] `/admin/recommendations` — date/audience/content-item picker over
      all 403 published items, list of existing overrides with remove.
      **Wired all the way through and verified live**: curated today's
      Fact-of-the-day to a specific fact via the admin UI, reloaded the
      parent-app home page, confirmed "Факт дня" showed the curated pick
      instead of the deterministic rotation's pick — the full
      admin-curates → parent-app-reflects loop, not just the admin side.
  - `entities/daily-recommendation/model/get-today-overrides.ts` is the
    read-time "recommendation_mode" resolution described in
    docs/CMS_BACKEND_ARCHITECTURE.md §5.6 — a matching override wins,
    otherwise `pickOfTheDay()` runs unchanged. Wired into
    `get-today-recommendations.ts` for game/article/checklist/fact/
    calm_moment/tip; `dailyRecommendation` (no CMS content type) can't be
    curated, consistent with the schema design.
- [ ] Minimal analytics dashboard beyond the counts already on
      `/admin` (dashboard page) — not started.

## Content enrichment pass — DONE (user-requested: real images, real playable audio, new facts)

- [x] **5 new Fact content items**, published, each with a real generated
      cover image (not the category-illustration fallback): classical
      music as calm shared time, early brain development (explicitly
      correcting the "after three it's too late" framing rather than
      repeating it — see the note below), reading aloud from infancy,
      sensory/tactile play, and moving/dancing to music. Each ran through
      the same guardrail scan (`shared/content/safety-guardrails.ts`) and
      draft-workflow bookkeeping (`ai_generations`, `content_versions`,
      `audit_log`) the admin UI itself uses, tagged `ai_generated: true`
      — this content was drafted by Claude at the user's request, held to
      the same standard as anything from the AI Content Studio.
  - **Content-safety note**: the user's brief referenced the pop-science
    book "После трёх уже поздно" (Ibuka) and its classical-music/early-
    genius claims. The brief's underlying interest (music and early
    stimulation matter) is covered, but not its stronger claims — "after
    three it's too late" is both developmentally inaccurate and exactly
    the kind of urgency-inducing framing `docs/CONTENT_RULES.md` and
    `docs/SAFETY.md` exist to prevent, and "classical music makes children
    smarter" is the literal example banned by `CONTENT_RULES.md` §1. The
    new brain-development fact explicitly names and corrects the "too
    late after three" framing rather than repeating it; the music fact
    states a calm-atmosphere benefit and explicitly disclaims an
    intelligence effect.
- [x] **5 real cover images**, generated (flat vector illustration style,
      on-brand warm palette, no photorealistic children — an ethics/brand
      choice, not just an aesthetic one) via `recraft_v4_1`, rasterized to
      PNG, uploaded to the `media-images` bucket, linked via
      `content_items.cover_media_id`. **Found and fixed a real RLS gap
      while verifying**: the existing `media_assets` public-read policy
      only covered assets linked through the `content_media` join table —
      an asset linked via the simpler, more common `cover_media_id`
      direct FK had no public-read policy at all, so the image row
      existed and the FK was correct but anon reads silently got `null`.
      Fixed in `20260906130100_media_cover_public_read.sql`.
- [x] **Found and fixed a second real bug in the same area**: adding the
      `media_assets` embed to the shared `readPublishedContent`/
      `readPublishedContentBySlug` helpers broke ALL content reads for
      every cut-over type, silently — PostgREST refuses an ambiguous
      embed when two relationships exist between the same tables
      (`cover_media_id` direct FK vs. the `content_media` many-to-many),
      returns `PGRST201`, and the helpers' existing "return null on error
      → caller falls back to mocks" design (correct for its original
      purpose) completely masked the failure. New content simply didn't
      appear, with no visible error anywhere. Fixed by disambiguating the
      embed (`media_assets!content_items_cover_media_fkey(...)`).
      **This is the second time in this project a graceful fallback has
      masked a real bug during this work** (see V2.0's `proxy.ts` note) —
      worth remembering that `isSupabaseConfigured`-style fallbacks are
      exactly the kind of code path that needs to be spot-checked
      directly (query the DB, don't just trust "the page didn't crash")
      rather than trusted by absence of errors.
- [x] **8 synthesized ambient/calm audio tracks** — white noise, pink
      noise, brown noise, fan hum, rain, sea waves, wind, and a heartbeat
      rhythm (`scripts/generate-calm-audio.mjs`, pure DSP, no external
      dependency: the generation tooling available in this environment is
      text-to-speech only and explicitly not licensed for standalone
      music/SFX, so real synthesis — not a fake/mislabeled substitute —
      was the honest option). ~40s mono WAV loops with an edge crossfade
      for seamless looping (heartbeat uses an exact whole number of beat
      cycles instead). Linked to 8 **existing** published Calm Moments by
      slug via `content_media` (role='audio') — no new Calm Moment
      records created, real content got real sound.
  - Needed a schema decision: `audio/wav` added to `ACCEPTED_MIME_TYPES`
    and the `media-audio` bucket's allowed MIME types
    (`20260906130000_audio_wav_mime_type.sql`) since no lossy encoder is
    available in this environment and file sizes are trivial at these
    durations (a few MB per loop).
- [x] **Real audio playback UI shipped** (`entities/calm-moment/ui/
      calm-audio-player.tsx`) — a genuine, if small, parent-app UI
      addition (the "freeze" applies to redesigning what exists; playing
      real sound where a non-functional "Listen later" placeholder stood
      is what the user explicitly asked for here). Play/pause via a
      native `<audio>` element, loops for whiteNoise/natureSounds
      categories, single module-level "currently playing" singleton so
      starting one moment's sound stops another's. Cover-image rendering
      (`entities/fact/ui/fact-card.tsx`) uses the real image when present
      and falls back to the existing category illustration otherwise —
      the "illustration fallback" design from
      `docs/CMS_BACKEND_ARCHITECTURE.md` §9.2 working as intended for the
      ~430 records that still have no attached media.
- [x] **Verified live end-to-end**: real cover image rendering on the
      Facts page (screenshot-confirmed, alongside icon-illustration facts
      for contrast), real audio playing on the Calm Moments page
      (play→pause button state confirmed, network tab showed a `206
      Partial Content` range request against Supabase Storage — correct
      audio-streaming behavior), zero console errors from playback,
      full-site regression pass (`/`, `/knowledge/facts`, `/calm`,
      `/games`, `/checklists`, `/admin`) all green after every fix.
- [ ] **Not done / explicitly out of scope this pass**: Kazakh
      translations for the 5 new facts were written directly (not via a
      separate review step) — flag for native-speaker review before wide
      release, same caveat as every other kk string in this project.
      Cover images and audio were attached only to the specific records
      requested/most relevant (5 facts, 8 calm moments) — extending this
      to more of the ~430 imported records is available but wasn't done
      broadly, to keep this pass scoped to what was asked.

## Settings & admin management — DONE (not in the original V2.x list, built alongside V2.6)

- [x] `/admin/settings`: grant/revoke admin access and change role for an
      *existing* auth account (never self-service — matches
      docs/CMS_BACKEND_ARCHITECTURE.md §6.1), plus a recent-activity view
      over `audit_log`. Uses the service-role client
      (`admin-server-client.ts`) specifically because `auth.users`/email
      lookups aren't exposed to PostgREST at all — one of the few
      legitimate uses for that client outside the importer script.
- [x] **Found a real local-testing gotcha while verifying this** (not a
      product bug): Next.js's `output: "standalone"` build does not
      auto-load `.env.local` when you run
      `node .next/standalone/server.js` from the repo root — server-only
      vars like `SUPABASE_SERVICE_ROLE_KEY` silently read as `undefined`,
      while `NEXT_PUBLIC_*` vars still appear to work because those are
      inlined into the JS bundle at *build* time, masking the problem.
      `NEXT_PUBLIC_*`-only features can look fully correct while
      server-only-secret features silently no-op. Fixed locally by
      copying `.env.local` into `.next/standalone/` and running `node
      server.js` from that directory instead. Not a code change — the
      existing `Dockerfile` already gets this right for real deployment,
      since a real deployment target injects env vars directly into the
      container/process rather than shipping a `.env` file — but worth
      remembering the next time something works from `/admin` dashboard
      but a service-role-dependent feature (Settings → Admins, the
      importer) mysteriously returns empty.

---

## Next up (first tasks of the next work session)

1. Cut over `article`, `tip`, `milestone`, `calm_moment` the same way as ~~
   `game`/`fact`/`checklist`~~ — **done, see V2.4 above.**
2. `HybridChecklistProgressRepository` once a concrete feature needs it
   (see V2.5's note on why it's deliberately deferred).
3. The real (non-template-fallback) Anthropic call in AI Content Studio
   and the real assistant swap in `/api/ai-chat` — both are the single
   remaining blocker across V2.6/V2.7, and it's the same blocker:
   `ANTHROPIC_API_KEY`. Everything both features depend on is built and
   tested.
4. Parent-facing feedback UI (thumbs up/down) for assistant messages —
   needs the streaming route to round-trip a message id first (see V2.7's
   note).
5. V2.8 (RAG/embeddings pipeline) once V2.7's real model call ships.
6. Minimal analytics beyond current dashboard counts (popular content,
   completion rates) — `completed_activities` already has everything
   needed for this, just no query/UI written yet.
