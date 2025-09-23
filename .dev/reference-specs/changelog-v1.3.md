---
id: changelog-v1-3
title: Pin the Close — Changelog (v1.3)
sidebar_label: Changelog v1.3
slug: /specs/changelog-v1-3
description: Summary of changes introduced with the Pin the Close v1.3 release compared to v1.2.
---

# Pin the Close — Changelog (v1.3)

## Summary
v1.3 consolidates the v1.2 spec, closes all items from the *v1.2 Current State Addendum*, and adds: anonymous play, gamer tags & privacy, inbound social webhooks, i18n (5 locales), timezone preferences, multi‑user offline isolation, dynamic OG previews, and a swappable market‑data layer. Drizzle was removed; schema is authored with SQL migrations only.

## Added
- **Anonymous play (zero‑friction):** Accept first guess without sign‑in via Supabase anonymous user (or guest token fallback). One‑click nudge to save progress; merge guest→auth on login.
- **Gamer identity & privacy:** `public_profiles` with unique **gamer_tag**, optional display name, avatar, and location; public flags (`public_*`). Leaderboards default to gamer_tag only.
- **Social providers — outbound & inbound:** Pluggable `SocialProvider` DI. **X (Twitter)** implemented end‑to‑end (posts with OG images; inbound mentions/replies/DMs ➜ guess). Stubs for TikTok, Bluesky, Instagram, Facebook. Claim flow via magic link.
- **Internationalization:** `next-intl` with **en‑US**, **en‑GB**, **ga** (Irish), **fr**, **es**. Localized OG images.
- **Timezone preferences:** Default to user’s locale TZ; editable in settings; notifications respect TZ + quiet hours.
- **Multi‑user offline isolation:** Partition SW caches/IndexedDB/localStorage by user id/guest token; sign‑out clears only current user’s data; guest→auth cache migration.
- **Dynamic OG previews:** `/og/[roundId|symbol]/[mode]` renders localized images (ticker, date, mode, K, time‑to‑bell). Used in social posts & link shares.
- **Market data provider layer:** Swappable `MarketDataProvider` with `mock` + one real provider; all settlement/HouseBot reads go through it.
- **Admin console expansions:** Audit Explorer with signed URL previews; Round Manager exports; Refdata Controls; **Social Inbox** for inbound events/moderation.
- **Notifications dispatcher:** Real transports (web push/email/mobile) with `notify_log` telemetry.

## Changed
- **No Drizzle:** Removed Drizzle ORM/Kit usage. Schema is defined only via **SQL migrations** under `supabase/migrations/`. Runtime access via `@supabase/supabase-js` (RLS enforced).
- **Canonical helper names:** Export `timeMultiplier(msToClose)` and `scoreGuess(base, submittedAt, closeTs)`.
- **HouseBot storage/audit:** Persist screenshots, parsed OHLC, features JSON; admin‑only signed previews; detailed rationale to `audit_trail`.

## Removed
- Drizzle schema DSL, config, and CI steps. Any Drizzle‑generated files or metadata should be deleted.

## Data Model Additions (v1.3)
- `public_profiles` (gamer_tag & privacy flags)
- `notify_log` (notification delivery telemetry)
- `refdata_runs` (instrument ingestion ops)
- `announcements`, `featured_tickers` (social outbound)
- `social_identities`, `social_events`, `social_submissions` (social inbound + claim flow)
- `badges`, `user_badges` (retention/achievements)

## RLS / Policy Updates
- Profiles expose only fields with `public_*` flags (default private).
- Pre‑lock privacy and post‑lock visibility (public/league/private‑game) unchanged but extended to new surfaces and admin/housebot exemptions.
- Social tables restricted to owners/admin; inbound submission visibility consistent with round visibility.

## Admin & Ops
- **Refdata**: nightly ingestion job + manual refresh with `refdata_runs` metrics.
- **Audit Explorer**: signed Storage previews for HouseBot artefacts; CSV/JSON exports.
- **Social Inbox**: moderation & claim status.

## Acceptance Criteria (high‑level)
- Anonymous visitors can submit a guess; nudge to sign in; guest→auth merge preserves history.
- Gamer tags are unique; privacy flags enforced; leaderboards use tags by default.
- X provider supports outbound posts (with OG) and inbound tagged submissions; claim flow works.
- i18n for 5 locales; OG images localized; TZ preferences applied; notifications honor TZ + quiet hours.
- User‑scoped offline caches; sign‑out clears only current user’s data; guest→auth migration works.
- Market data accessed *only* via provider; mock covered by tests.
- Admin console covers audit/exports/refdata/social inbox; notifications logged.

## Migration Guide (from v1.2)
1. **Remove Drizzle** deps/config; ensure CI no longer calls Drizzle.
2. **Add new tables** via SQL migrations and enable RLS with policies as specified.
3. **Rename** any helper imports to `timeMultiplier` and `scoreGuess`.
4. **Introduce** `MarketDataProvider` and refactor settlement/HouseBot to use it.
5. **Wire** anonymous play & guest→auth merge; partition offline caches.
6. **Add** i18n, TZ prefs, notifications dispatcher & `notify_log`.
7. **Expand** admin and social features as above.

---

_End of changelog._

