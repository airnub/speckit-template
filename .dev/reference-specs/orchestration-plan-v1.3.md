---
id: orchestration-plan-v1-3
title: Pin the Close — Orchestration Plan (v1.3)
sidebar_label: Orchestration Plan v1.3
slug: /specs/orchestration-plan-v1-3
description: Repository orchestration plan aligning autonomous agents to the Pin the Close v1.3 specification.
---

# Pin the Close — Orchestration Plan (v1.3)

**Purpose**: This file tells autonomous coding agents exactly **what to build**, **how to build it**, and **how to verify it**. It binds implementation to the v1.3 spec and preserves extensibility (Market Data & Social providers), security (RLS, Vault), accessibility, and offline UX.

---

## 0) Documents of Record (DoR)
- **Authoritative spec**: `docs/specs/game-spec-v1.3.md`
- **Historical context (read‑only)**: `docs/specs/game-spec-v1.2.md`, `docs/specs/game-spec-addendum-v1.2-current-state.md`

> The code **must** conform to **v1.3**. Where ambiguity exists, v1.3 overrides earlier docs. Any deviation must be documented in PRs.

---

## 1) Tech Stack & Conventions (v1.3)
- **Framework**: Next.js 15, **App Router**, **RSC by default**, **Server Actions** for mutations
- **Auth**: Supabase Auth (source of truth, RLS via `auth.uid()`); next‑auth for session ergonomics only
- **Database**: Supabase Postgres; **schema authored via SQL migrations** in `/supabase/migrations` (**Drizzle removed**)
- **RLS**: Enabled on all app tables; JWT claim `role ∈ {user, housebot, socialbot, admin}`
- **Runtime data access**: **`@supabase/supabase-js`** (always with user JWT) so **RLS** is enforced
- **Storage**: Supabase Storage (private buckets, **signed URLs**); avatars & HouseBot artefacts
- **Secrets**: Supabase **Vault** via **server‑only RPCs**. Per‑user secrets (if any) via **TCE + RLS**. **No secrets in migrations**
- **UI**: Tailwind + **shadcn/ui** (Radix); Framer Motion (respect `prefers-reduced-motion`); lucide-react
- **i18n**: **next‑intl** with locales **en‑US**, **en‑GB**, **ga**, **fr**, **es**; localized OG images
- **PWA**: Offline‑first; **user‑scoped caches** (SW/IndexedDB/localStorage) + Background Sync
- **Type Safety**: TypeScript strict, zod validation
- **CI**: GitHub Actions (lint, typecheck, build, SQL migration parse, policy/RLS tests, axe, Lighthouse)

---

## 2) Directory Layout
```
/ (repo root)
├─ app/
│  ├─ dashboard/
│  │  ├─ rounds/[id]/             # round detail + GuessPad
│  │  ├─ leagues/[id]/            # league space
│  │  └─ games/[id]/              # private game space
│  ├─ api/
│  │  ├─ cron/settle/             # settle jobs
│  │  ├─ cron/notify/             # dispatcher
│  │  ├─ cron/refdata/            # instruments ingestion
│  │  └─ webhooks/x/              # X (Twitter) inbound webhooks
│  ├─ og/[...slug]/               # dynamic OG images (localized)
│  └─ layout.tsx / page.tsx
├─ components/                    # SSR‑friendly UI + shadcn components
├─ lib/
│  ├─ supabase/                   # server/client helpers (token passthrough)
│  ├─ auth/                       # next‑auth config (Supabase token in session)
│  ├─ game/                       # scoring helpers, settlement oracles, contracts
│  ├─ providers/
│  │  ├─ market-data/             # pluggable MarketDataProvider impls
│  │  └─ social/                  # SocialProvider DI (X implemented, others stubs)
│  ├─ notify/                     # dispatcher, templates, next‑intl integration
│  ├─ pwa/                        # SW, bg sync, user‑scoped caches
│  ├─ i18n/                       # next‑intl helpers, message catalogs
│  └─ admin/                      # admin helpers, exports
├─ public/                        # icons, manifest, static assets
├─ supabase/
│  ├─ migrations/                 # SQL (schema, RLS, triggers)
│  └─ seed/                       # seeds (badges, instruments, etc.)
├─ docs/specs/                    # v1.3 spec + history
├─ scripts/                       # bootstrap‑vault, ops scripts
└─ .github/workflows/             # CI pipelines
```

---

## 3) Environment & Secrets
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=            # server/CI only
NEXTAUTH_SECRET=
NEXTAUTH_URL=http://localhost:3000
MARKET_DATA_PROVIDER=mock|polygon|iex|tradier
```
- Use `scripts/bootstrap-vault.ts` to seed Vault from `.vault.<env>` (idempotent). Vault reads are **server-only** via RPC. Keep provider tokens (X API, SMTP, etc.) in Vault.

---

## 4) Data Model (v1.3)
**Core (from v1.2):** `instruments`, `rounds`, `commits`, `guesses`, `results`, `scores`, `seasons`, `season_totals`, `leagues`, `league_members`, `league_rounds`, `private_games`, `private_game_members`, `user_prefs`, `bot_inputs`, `audit_trail`.

**Additions (v1.3):** `public_profiles`, `notify_log`, `refdata_runs`, `announcements`, `featured_tickers`, `social_identities`, `social_events`, `social_submissions`, `badges`, `user_badges`.

**RLS**: Pre‑lock privacy; post‑lock visibility by (public | league member | private‑game member); admin/housebot exemptions; profiles expose only fields with `public_*` flags. Include **behavioural policy tests** in CI.

---

## 5) Auth Strategy (Anonymous → Account)
- **Anonymous play**: Prefer Supabase anonymous sign‑in; else issue a server‑signed **guest token** and store guest rows with RLS‑safe access
- **Zero‑friction**: Allow first guess without auth; submit via Server Action with guest identity
- **Upgrade nudge**: After submit, prompt sign‑in/up (magic link/OAuth) to save history, reputation, badges, notifications, Private Games/Leagues
- **Merge**: On auth, **merge guest→auth** data once (guesses/scores/prefs) and log to `audit_trail`
- **Session**: next‑auth embeds Supabase access token; server actions use tokened client

---

## 6) Core Game Flows
- **Create Round**: Admin sets `mode`, `ticker`, `reference_price`, `tick_size`, `lock_ts`, `close_ts`, `is_public`, `settle_rule`; attach to private game/league
- **Submit Guess**: GuessPad with A/B/E, countdown, time‑decay badge; add‑on chips (AHC/PM/RMO). Offline queue; server receipt decides time‑band; `audit_trail` entry
- **Lock & Settle**: `cron:lock-rounds` and `cron:settle-rounds`; settlement sources go through **MarketDataProvider**; write `results`, `scores`, streaks; audit
- **Leaderboards**: Rankings, streaks, and badges per mode/season; respect privacy (gamer_tag default)

---

## 7) Settlement & Scoring (canonical)
- **Close**: official close @ 16:00 ET (13:00 ET half‑days) → equals open of 21:00 bar locally
- **Equality**: `round(P_close, 2) == round(K, 2)` ⇒ EQUAL; else sign ⇒ ABOVE/BELOW
- **Base**: EQUAL +50; correct ABOVE/BELOW +10; wrong −5
- **Time multiplier**: ≥10m×1.00; 5–9×0.60; 3–4×0.40; 1–2×0.20; &lt;1m×0.10
- **Final**: `score = base * time_multiplier(msToClose)`
- **Add‑ons**: AHC @20:00 SIP last (or last before); PM first trade ≥04:00 (void if none); RMO official open @09:30; cent ladder Exact +50 → 0 at ±$0.10 (or −5 if configured)
- **Exports**: `timeMultiplier(msToClose)`, `scoreGuess(base, submittedAt, closeTs)`

---

## 8) HouseBot (default heuristic)
- Uses only user‑visible data; **no privileged auction feeds** by default
- Features & T‑aware equations per spec; deterministic on fixtures
- Store artefacts (screenshots, OHLC JSON, features JSON) in Storage; save paths in `bot_inputs`; generate **admin‑only signed URLs**; `audit_trail` rationale

---

## 9) Notifications & Preferences
- `user_prefs` toggles with **T‑15 ON by default**; per‑channel controls (web push/email/mobile)
- Dispatcher with provider DI; respects quiet hours and user TZ; logs to `notify_log`

---

## 10) PWA Offline (User‑Scoped)
- SW caches app shell + APIs; Background Sync queues guesses
- **Partition caches/IndexedDB/localStorage by user id or guest token**
- On sign‑out, clear **only** current user’s partitions; on guest→auth, migrate then delete guest partitions

---

## 11) Private Games & Leagues
- **Private Game**: invite‑only, one round (+ add‑ons); `/dashboard/games/[id]`
- **League**: series of rounds with leaderboard; `/dashboard/leagues/[id]`
- Deep‑link invites with OG image; visibility via RLS

---

## 12) Admin Console
- **Audit Explorer**: search/filter; preview HouseBot artefacts (signed URLs); export CSV/JSON
- **Round Manager**: attach to leagues/private games; export results
- **Refdata Controls**: trigger ingestion; show `refdata_runs`
- **Social Inbox**: list inbound `social_events`/`social_submissions`, claim status, moderation

---

## 13) Social Providers (Outbound + Inbound Webhooks)
- DI interface `SocialProvider` with:
  - Outbound: `publishDailyChallenge`, `publishWinners`, `publishRoundUpdate`
  - Inbound: `handleMention`, `handleReply`, `handleDM`
- **X (Twitter)** implemented end‑to‑end (signature checks, dedupe, rate limits, claim via magic link)
- **Stubs** for TikTok, Bluesky, Instagram, Facebook (interfaces only at launch)
- Outbound posts must include **localized dynamic OG image** (see §16)

---

## 14) Market Data Provider (swappable)
- Interface methods:
  - `getOHLC(symbol, interval, from, to)`
  - `getOfficialClose(symbol, date)`
  - `getOfficialOpenAuction(symbol, date)`
  - `getAHCLast(symbol, date)`
  - `getFirstPreMarketTrade(symbol, date)`
- Select via `MARKET_DATA_PROVIDER`; include **mock** + one real; retries/backoff inside provider

---

## 15) OG Images & Social Copy
- Route `/og/[roundId|symbol]/[mode]` renders localized dynamic OG images (Satori/OG) with ticker, name, date, `K`, mode, time‑to‑bell; accessible colours; alt text

---

## 16) Accessibility Checklist (PR gate)
- Keyboard: A/B/E, Enter submit, U undo (3s), skip links, ESC closes dialogs
- Gestures: swipe L/R, tap center, **long‑press confirm**, **double‑tap undo**
- Screen reader: ARIA labels & live regions; contrast ≥ 4.5:1; respects `prefers-reduced-motion`

---

## 17) CI/CD & Quality Gates
- **Checks**: typecheck, lint, build; SQL migration parse; RLS behavioural tests; axe a11y; Lighthouse PWA ≥ 90; OG route snapshot
- **Preview**: Vercel previews per PR; block on failures
- **Secrets**: injected server‑side; Vault bootstrap allowed in CI; **no service role in browser**

---

## 18) Labels & PR Template
- **Labels**: `spec-v1.3`, `schema`, `rls`, `auth`, `ui`, `pwa`, `bot`, `notifications`, `admin`, `providers`, `i18n`, `og`, `social`
- **PR template must include**: spec sections implemented; migrations; RLS policies; screenshots/GIFs; a11y checklist; risks/rollback; deviations from spec

---

## 19) Milestones (strict order)
1. **Anon Play** — anon/guest flow; nudge; guest→auth merge; audit
2. **Profiles & Privacy** — `public_profiles`; unique `gamer_tag`; visibility; avatars
3. **Social (X) Outbound+Inbound** — DI + X provider; webhooks; claim flow; moderation inbox
4. **i18n & TZ** — next‑intl; localized OG; TZ pref; quiet hours
5. **User‑Scoped Offline** — cache partitioning; sign‑out clear; guest→auth migration
6. **Market Data Provider** — interface + mock + one real; settlement through provider
7. **OG Images & Social Copy** — `/og/*` route; localized templates
8. **Addendum Closures** — notifications dispatcher + `notify_log`; HouseBot artefacts + signed previews; admin exports; refdata ingestion + controls; commit‑reveal; RLS tests; badges; rematch; daily recap

---

## 20) Non‑Negotiables
- **No Drizzle**. SQL migrations only; Supabase JS at runtime.
- **RLS everywhere**; bots as first‑class users.
- **Accessibility + Offline** are core.
- **Secrets** via Vault/TCE; **no secrets in migrations**.

---

## 21) Agent Run Prompt (copy‑paste)
> Implement **v1.3** per `docs/specs/game-spec-v1.3.md` using Supabase **SQL migrations** (no Drizzle) and **@supabase/supabase-js** at runtime with RLS. Ship: anonymous play; gamer profiles & privacy; social providers (X outbound+inbound; others stubbed) with OG previews; next‑intl i18n & timezone prefs; multi‑user offline cache isolation; swappable market‑data provider; and all v1.2 addendum closures (notifications, HouseBot artefacts, admin, refdata, commit‑reveal, badges, recap). Include tests and CI gates as specified in `docs/specs/orchestration-plan-v1.3.md`.

