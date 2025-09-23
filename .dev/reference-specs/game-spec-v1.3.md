---
id: game-spec-v1-3
title: Pin the Close — Game Specification (v1.3)
sidebar_label: Game Spec v1.3
slug: /specs/game-spec-v1-3
description: Consolidated Pin the Close v1.3 gameplay, scoring, and platform specification.
---

# Pin the Close — Game Specification (v1.3)

**File path (suggested in repo):** `docs/specs/game-spec-v1.3.md`

> v1.3 consolidates v1.2 and promotes all items from the *v1.2 Current State Addendum* to **mandatory**. It also adds new requirements: zero‑friction anonymous play, gamer tags & privacy, inbound social webhooks, i18n, timezone preferences, multi‑user offline isolation, dynamic OG previews, and a swappable market‑data layer.

**Stack:** Next.js 15 · App Router (RSC by default) · next‑auth · Supabase (Auth + Postgres + Storage + RLS + Vault/TCE) · Tailwind + shadcn/ui (Radix) · next‑intl · PWA offline‑first · Vitest · GitHub Actions.

> **Important:** Drizzle has been removed. Author the schema with **SQL migrations** under `/supabase/migrations`. Use **`@supabase/supabase-js`** at runtime so **RLS** is enforced via the user’s JWT.

---

## 0) Scope & Documents of Record
- [Game Specification (v1.2)](/docs/specs/game-spec-v1-2) — baseline rules
- [Game Specification Current-State Addendum (v1.2)](/docs/specs/game-spec-addendum-v1-2-current-state) — all ⚠️/❌ are mandatory in v1.3
- This v1.3 specification

---

## 1) Core Gameplay (unchanged fundamentals)
- **Main Round — “Pre‑Auction Classic”**: player chooses **ABOVE / BELOW / EQUAL** vs reference price `K` (e.g., 3.00) for the **official close**.
- **Side guesses (optional)**: **AHC** (After‑Hours Close target@20:00 ET), **PM** (Pre‑Market first trade ≥04:00 ET), **RMO** (Regular‑Market official opening auction @09:30 ET).
- **Time‑decay scoring**: earlier submissions earn more points.
- **Private Games** (invite‑only single round) are distinct from **Leagues** (series/tournaments).

### 1.1 Player surfaces
- Routes: `/dashboard/rounds/[id]`, `/dashboard/games/[id]` (Private Game), `/dashboard/leagues/[id]` (League).
- Deep‑link invites with OG share images; visibility enforced by RLS.

---

## 2) Instruments & Coverage
- **Universe**: Any **NYSE/Nasdaq** equity with `status='active'`.
- **Tick size**: store per instrument (default `0.01`).
- **Ingestion**: Nightly `cron:refdata` updates listings & tick sizes; Admin **Refresh Now** shows `refdata_runs` metrics.

---

## 3) Settlement Rules (authoritative)
**Close Round** = primary exchange **official close** at **16:00:00 ET** (13:00 ET half days). On extended‑hours charts this appears as **the open of the 21:00 bar** in local time.

**Equality test**
```
Let P_close = settlement price, K = reference price
EQUAL iff round(P_close, 2) == round(K, 2)
Else ABOVE if P_close > K; otherwise BELOW
```

**Side guesses**
- **AHC**: SIP last eligible trade at **20:00:00 ET** (or last before).
- **PM**: First consolidated trade **≥ 04:00:00 ET** (void if none).
- **RMO**: **Official opening auction** at **09:30:00 ET**.

**Halt/fallbacks**
- If no auction, use **Official Last Sale**; if none, **void**.

---

## 4) Scoring & Time‑Decay (unchanged math)
- Base points: **EQUAL +50**, **ABOVE/BELOW +10**, **Wrong −5**.
- Multiplier (minutes to bell): **≥10×1.00**, **5–9×0.60**, **3–4×0.40**, **1–2×0.20**, **&lt;1×0.10**.
- Final: `score = base × multiplier`.
- Side‑guess cent ladder: Exact +50 → 0 at ±$0.10; beyond window = 0 (or −5 if configured).
- **Public helpers (canonical names)**: `timeMultiplier(msToClose)`, `scoreGuess(base, submittedAt, closeTs)`.

---

## 5) HouseBot (default heuristic; storage/audit tightened)
- Use the same features & T‑aware weighting from v1.2 (no privileged auction feeds in default mode).
- Store artefacts (screenshots, parsed OHLC JSON, features JSON) in Supabase Storage; write paths to `bot_inputs`.
- Generate **admin‑only signed URLs**; write rationale & decisions to `audit_trail`.

---

## 6) Zero‑Friction Play for New Visitors (Anonymous)
**Goal:** Let first‑time visitors submit a guess **without sign‑up**; encourage sign‑in/up to save progress.

### 6.1 Anonymous identity
- Prefer **Supabase anonymous sign‑in**. If unavailable, issue a server‑signed **guest token** and store a `guest_user_id` under RLS; guesses are accepted under guest identity.

### 6.2 Gentle upgrade
- After the first guess, show a **non‑blocking nudge** to sign in/up (magic link/OAuth) to save history, reputation, badges, notifications, Private Games/Leagues.

### 6.3 Merge guest→account
- On authentication, merge all guest data (guesses/scores/prefs) to the real `auth.uid()` **exactly once**; write an `audit_trail` entry.

---

## 7) Gamer Identity & Privacy
- Table **`public_profiles`** stores:
  - `gamer_tag` (unique, 3–20 chars; allowed charset; profanity filter),
  - `display_name` (opt‑in), `avatar_url`, `location` (city/country),
  - visibility flags: `public_display_name`, `public_location`, `public_avatar` (default **false**).
- **Default public surface** shows only **gamer_tag**. Leaderboards & social surfaces use gamer_tag unless user opts into more.
- Avatars stored in private bucket; if `public_avatar=true`, serve via signed public URL.

---

## 8) Social Media Bots — Outbound & Inbound (Webhooks)
**Goal:** Viral loop via posts and friction‑free tagged submissions.

### 8.1 Provider contracts (DI)
Define `SocialProvider` with outbound and inbound methods:
```ts
publishDailyChallenge(payload: ChallengePost): Promise<void>
publishWinners(payload: WinnersPost): Promise<void>
publishRoundUpdate(payload: RoundUpdatePost): Promise<void>
handleMention(event: ProviderEvent): Promise<SocialSubmissionResult>
handleReply(event: ProviderEvent): Promise<SocialSubmissionResult>
handleDM(event: ProviderEvent): Promise<SocialSubmissionResult>
```
Providers use Vault‑stored credentials and per‑provider rate limits.

### 8.2 Outbound posts
- When posting a challenge/update, include a **localized dynamic OG image** (see §12) with ticker, date, mode, and `K`.

### 8.3 Inbound tag‑to‑guess (zero friction)
- If a user **tags the app** or replies using a recognisable pattern (e.g., `@PinTheClose AAPL = 190.00`, or `ABOVE/BELOW/EQUAL`), the inbound webhook **parses** symbol, mode, and guess and inserts a **pending** or **accepted** guess.
- **Identity & email linking**:
  - If provider supplies verified email, link immediately.
  - Otherwise, auto‑reply with a **one‑tap claim link** (magic link page) to verify email; upon claim, associate the handle to the account for future submissions.
- The **first inbound guess counts** even before claim; it remains tied to the handle. If not claimed within N days, it remains limited to handle‑only features.
- Security: signature verification, dedupe by provider event ID/hash, profanity checks, rate limiting.

### 8.4 Providers at launch
- **Implement now**: **X (Twitter)** outbound + inbound webhooks end‑to‑end.
- **Stubs (interfaces only)**: TikTok, Bluesky, Instagram, Facebook.

---

## 9) Internationalization (i18n) — next‑intl
- Locales: **en‑US**, **en‑GB**, **ga** (Irish), **fr**, **es**.
- Localize all UI strings, dates/times (user TZ), numbers, and OG images.
- Language switcher (settings); server negotiates from `Accept‑Language` with fallback to `en‑US`.

---

## 10) Timezones
- Persist times in UTC; display in the **user’s preferred timezone** (default: browser/locale).
- Exchange events remain **ET** for rules/settlement but show a local‑time helper.
- Notifications respect user TZ and quiet hours.

---

## 11) Multi‑User Offline & Cache Isolation (shared devices)
- Partition all client caches (IndexedDB, localStorage, SW caches) by **user id or guest token**.
- On **sign‑out**, clear **only current user’s** partitions; do not affect others.
- On guest→auth merge, migrate and then remove guest partitions.

---

## 12) Social OG Previews (dynamic images)
- Route `/og/[roundId|symbol]/[mode]` renders a dynamic OG image (e.g., Satori/OG) with: ticker, company, reference `K`, round date, mode, and a small time‑to‑bell indicator if applicable; localized via next‑intl; high contrast; alt text.

---

## 13) Market Data Provider — Swappable Design
- Define `MarketDataProvider` interface:
```ts
getOHLC(symbol: string, interval: Interval, from: Date, to: Date): Promise<Bar[]>;
getOfficialClose(symbol: string, date: string): Promise<number>;
getOfficialOpenAuction(symbol: string, date: string): Promise<number>;
getAHCLast(symbol: string, date: string): Promise<number>;
getFirstPreMarketTrade(symbol: string, date: string): Promise<number>;
```
- Choose provider by env var `MARKET_DATA_PROVIDER=polygon|iex|tradier|mock`.
- All settlement and HouseBot inputs call through this layer; include a **mock** provider for tests. Handle rate limits, retries, and backoff inside the provider.

---

## 14) UX & Accessibility
- Keyboard: `A`/`B`/`E`, `Enter` submit, `U` undo (3s). Gestures: swipe left/right, tap center, **long‑press confirm**, **double‑tap undo**. Skip‑to‑content. WCAG 2.1 AA (contrast ≥ 4.5:1; no color‑only cues; respects `prefers‑reduced‑motion`).
- Canonical rules copy (Close, AHC/PM/RMO, time‑decay) must appear on round details.

---

## 15) Data Model — v1.3 Additions (selected)
> v1.2 core tables remain: `instruments`, `rounds`, `commits`, `guesses`, `results`, `scores`, `seasons`, `season_totals`, `leagues`, `league_members`, `league_rounds`, `private_games`, `private_game_members`, `user_prefs`, `bot_inputs`, `audit_trail`.

**New tables**
- `public_profiles(user_id pk, gamer_tag unique, display_name, avatar_url, location, public_display_name bool, public_location bool, public_avatar bool, created_at)`
- `notify_log(id, user_id, channel, kind, scheduled_for, sent_at, status, error)`
- `refdata_runs(id, started_at, finished_at, status, added int, updated int, errors jsonb)`
- `announcements(id, provider, payload jsonb, created_at, round_id?, league_id?, game_id?)`
- `featured_tickers(id, symbol, picked_for date, reason, created_at)`
- `social_identities(id, user_id, provider, handle, provider_user_id, email_verified bool, created_at)`
- `social_events(id, provider, event_id, kind, payload jsonb, received_at, dedup_hash unique)`
- `social_submissions(id, social_event_id fk, round_id fk, parsed_guess jsonb, status enum('pending','accepted','rejected'), user_id?, handle, created_at)`
- `badges(id, code unique, name, description, icon)`; `user_badges(user_id, badge_id, awarded_at, season_id?)`

**RLS**
- Pre‑lock privacy; post‑lock visibility by public/private‑game/league membership; admin/housebot audit exemptions.
- Profiles expose only fields with `public_*` flags.

---

## 16) Notifications (real transports)
- Dispatcher supports **web push**, **email**, and **mobile push** (provider DI). Respect **quiet hours** and user TZ.
- Log outcomes to `notify_log`; summarize in Admin.

---

## 17) Admin Console
- **Audit Explorer**: preview HouseBot artefacts (signed URLs); search/filter; export CSV/JSON.
- **Round Manager**: attach to leagues/private games; export results.
- **Refdata Controls**: run ingest; show last `refdata_runs`.
- **Social Inbox**: inbound `social_events`/`social_submissions` moderation & claim status.

---

## 18) Non‑Functional & Security
- **Privacy**: default to gamer_tag‑only public profile. **Vault** for app secrets; **TCE + RLS** for per‑user keys if added. **No secrets in migrations.**
- **Performance**: SSR by default; cache wisely; image optimization; graceful degradation.
- **Reliability**: idempotent crons; retries/backoff; server clock authoritative.
- **Compliance**: audit trail for key actions; profanity filters on gamer_tag & inbound social.

---

## 19) Acceptance Criteria (v1.3)
- All v1.2 addendum ⚠️/❌ items are closed with tests and UI.
- Anonymous visitors can submit a guess on first visit; post‑submit nudge shown; guest→auth merge preserves history.
- Users can set **unique gamer_tag** and opt into displaying name/location/avatar.
- Inbound social tagging produces **pending/accepted** guesses; claim flow via email; outbound posts include **localized OG images**.
- i18n works for **en‑US, en‑GB, ga, fr, es**; OG images localized.
- Timezone preference applied; notifications honor TZ + quiet hours.
- User‑scoped offline caches; sign‑out clears only current user’s data; guest→auth cache migration works.
- Market data accessed only via `MarketDataProvider`; mock provider covered by tests.
- Admin console exposes audit, exports, refdata controls, and social inbox.
- Social providers: **X implemented**; TikTok/Bluesky/Instagram/Facebook **stubbed**. All secrets via Vault.

---

## 20) Disclaimer
This product is a **game** for entertainment/education, not financial advice. Market data is for gameplay only.

