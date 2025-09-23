---
id: coding-agent-brief-v1-3
title: Pin the Close — Coding Agent Brief (v1.3)
sidebar_label: Coding Agent Brief v1.3
slug: /specs/coding-agent-brief-v1-3
description: Implementation brief for agents delivering the Pin the Close v1.3 specification.
---

# Pin the Close — Coding Agent Brief (v1.3)

**Repository name:** `pin-the-close`

**Authoritative spec:** `docs/specs/game-spec-v1.3.md`
**Implementation guide:** `docs/specs/orchestration-plan-v1.3.md`
**Historical context (read‑only):**
- `docs/specs/game-spec-v1.2.md`
- `docs/specs/game-spec-addendum-v1.2-current-state.md`

> Follow **v1.3** exactly for gameplay rules, settlement, scoring, RLS, offline, i18n, and privacy. Where ambiguity exists, **v1.3 overrides** earlier docs. Any deviation must be documented in PRs with rationale.

---

## 0) Ground Rules
- **Framework:** Next.js 15 (App Router), **RSC by default**, **Server Actions** for mutations.
- **Auth:** Supabase Auth is the **source of truth**. next‑auth for session ergonomics only. All DB access uses **`@supabase/supabase-js`** with the user’s JWT so **RLS** applies. **No Drizzle.**
- **Database:** Supabase Postgres; schema authored in **SQL migrations** under `/supabase/migrations`. **RLS everywhere**.
- **Secrets:** Supabase **Vault** via server‑only RPCs. (Optional) TCE+RLS for per‑user keys. **No secrets in migrations.**
- **Storage:** Supabase Storage (private buckets; **signed URLs** for previews/avatars/HouseBot artefacts).
- **UI:** Tailwind + shadcn/ui (Radix); Framer Motion (respect `prefers-reduced-motion`); lucide-react.
- **i18n:** `next-intl` with `en-US`, `en-GB`, `ga`, `fr`, `es`; localized OG images.
- **PWA:** Offline‑first; **user‑scoped caches** (SW/IndexedDB/localStorage) + Background Sync.
- **Security:** Bots are **first‑class users** (`user|housebot|socialbot|admin` in JWT claims). Webhook signatures, rate limits, profanity filters.
- **Testing/CI:** Vitest + Playwright/axe + Lighthouse + policy tests. GitHub Actions blocks merges on failures.

---

## 1) Environment & Secrets
Create `.env.local` (dev) and provision Vault:
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=         # server/CI only
NEXTAUTH_SECRET=
NEXTAUTH_URL=http://localhost:3000
MARKET_DATA_PROVIDER=mock|polygon|iex|tradier
```
- Use `scripts/bootstrap-vault.ts` to seed provider credentials from `.vault.local` (idempotent create/update). Vault reads happen **only** server‑side via RPC.

---

## 2) Work Breakdown (Milestones + AC)

### M1 — Anonymous Play (Zero Friction)
**Build:** Anonymous user flow (Supabase anonymous sign‑in; fallback guest token). Accept first guess without auth; show gentle upgrade nudge; merge guest→account on auth.  
**AC:** First‑time visitor can submit a guess; upgrade preserves history; `audit_trail` records merge.

### M2 — Gamer Profiles & Privacy
**Build:** `public_profiles` (unique `gamer_tag`, avatar, location, opt‑in visibility flags). Leaderboards show **gamer_tag** by default. Avatar served via signed URL if public.  
**AC:** Unique tag enforced with profanity filter; PII hidden unless opted in; RLS prevents leakage.

### M3 — Social Providers (Outbound + Inbound Webhooks)
**Build:** DI interface `SocialProvider`; implement **X (Twitter)** outbound posts + inbound webhooks (`handleMention/Reply/DM`) with signature verification, dedupe, and rate limits. Add `social_identities`, `social_events`, `social_submissions`; claim flow via magic link by email.  
**AC:** Tagging the app on X submits a **pending/accepted** guess; claim links associate handle↔email; outbound posts include localized OG image.

### M4 — i18n & Timezones
**Build:** `next-intl` setup; localized UI/OG images; TZ preference with default from browser; notifications honor TZ + quiet hours.  
**AC:** All strings localized; dates/times/nums formatted per locale & TZ; switcher works.

### M5 — Multi‑User Offline & Cache Isolation
**Build:** Partition SW caches, IndexedDB, and localStorage by **user id or guest token**. On sign‑out clear only current user’s partition; migrate guest partitions on upgrade.  
**AC:** Shared device tests pass; no cross‑user leakage.

### M6 — Market Data Provider (Swappable)
**Build:** `MarketDataProvider` interface; include **mock** + one real provider. Route settlement & HouseBot reads through the provider; handle retries/backoff; select via env.  
**AC:** Provider swap requires no refactor; tests run on mock; settlement uses provider.

### M7 — OG Images & Social Copy
**Build:** Dynamic `/og/[roundId|symbol]/[mode]` (Satori/OG) with ticker, name, date, `K`, mode, time‑to‑bell; localized; accessible colours/alt.  
**AC:** Social posts render correct previews; a11y contrast met.

### M8 — v1.2 Addendum Closures
**Build:** Private Games & Leagues pages; Notifications dispatcher + `notify_log`; HouseBot artefacts + signed previews; Admin exports; Refdata ingest + controls; Commit‑reveal; RLS behavioural tests; Badges, Rematch, Daily Recap.  
**AC:** All gaps closed; tests green; admin features operational.

---

## 3) Data Model & RLS
**Add/maintain tables:**
- Core (v1.2): `instruments`, `rounds`, `commits`, `guesses`, `results`, `scores`, `seasons`, `season_totals`, `leagues`, `league_members`, `league_rounds`, `private_games`, `private_game_members`, `user_prefs`, `bot_inputs`, `audit_trail`.
- v1.3 additions: `public_profiles`, `notify_log`, `refdata_runs`, `announcements`, `featured_tickers`, `social_identities`, `social_events`, `social_submissions`, `badges`, `user_badges`.

**RLS policies:** Pre‑lock privacy; post‑lock visibility (public/league/private members); admin/housebot audit exemptions; profiles expose only opted‑in fields. Add executable **policy tests** for each scenario.

---

## 4) Settlement, Scoring & Helpers (canonical)
- **Equality:** `round(P_close, 2) == round(K, 2)` ⇒ EQUAL; else sign ⇒ ABOVE/BELOW.
- **Base points:** EQUAL +50; correct ABOVE/BELOW +10; wrong −5.
- **Time multiplier:** ≥10m×1.00; 5–9m×0.60; 3–4m×0.40; 1–2m×0.20; &lt;1m×0.10.
- **Final:** `score = base * timeMultiplier(msToClose)`.
- **Add‑ons:** AHC @20:00 SIP last (or last before); PM first trade ≥04:00 (void if none); RMO official open @09:30. Cent ladder: Exact +50 → 0 at ±$0.10 (or −5 if configured).  
- **Exports:** `timeMultiplier(msToClose)`, `scoreGuess(base, submittedAt, closeTs)`.

---

## 5) Admin Console
- **Audit Explorer:** HouseBot artefact previews (signed URLs), filters, CSV/JSON exports.
- **Round Manager:** attach rounds to private games/leagues, export results.
- **Refdata Controls:** trigger ingestion; show last `refdata_runs`.
- **Social Inbox:** review inbound `social_events`/`social_submissions`, claim status, moderation.

---

## 6) Security & Abuse Controls
- Webhook signature checks; dedupe by provider event ID/hash; rate limiting; profanity filter for `gamer_tag` and inbound text. Vault RPCs server‑only; Storage via signed URLs only.

---

## 7) Tests & CI Gates
- **Unit:** scoring helpers, provider selector, i18n formatters, OG renderer.
- **Integration:** anon→auth merge; RLS behaviours; social inbound parse/claim; SW partition isolation; notification dispatch logging.
- **A11y:** axe smoke on GuessPad, round detail, admin, profile.
- **PWA:** Lighthouse ≥ 90; offline queue round‑trip.
- **DB:** migration parse + policy tests in CI; seeds for badges/instruments.

---

## 8) Deliverables & DoD
- Code, migrations, seeds, and fixtures implementing **all v1.3 sections**.
- Updated docs: `docs/specs/game-spec-v1.3.md`, `docs/specs/orchestration-plan-v1.3.md`, README, and a short `docs/specs/changelog-v1.2-to-v1.3.md`.

---

### COPY‑PASTE PROMPT FOR AUTONOMOUS AGENTS
> Implement **v1.3** per `docs/specs/game-spec-v1.3.md` using Supabase **SQL migrations** (no Drizzle) and **@supabase/supabase-js** at runtime with RLS. Ship: anonymous play; gamer profiles & privacy; social providers (X outbound+inbound; others stubbed) with OG previews; next‑intl i18n & timezone prefs; multi‑user offline cache isolation; swappable market‑data provider; and all v1.2 addendum closures (notifications, HouseBot artefacts, admin, refdata, commit‑reveal, badges, recap). Include tests and CI gates as specified in `docs/specs/orchestration-plan-v1.3.md`. Ensure all secrets use Vault and no secrets appear in migrations.

