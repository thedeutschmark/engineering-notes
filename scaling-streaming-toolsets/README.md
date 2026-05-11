# Scaling Per-User Streaming Toolsets on Cloudflare — Edge Push, Hibernation, and Flat Cost-Per-User

**Date:** 2026-05-10
**Author:** deutschmark
**Status:** Phase 2 complete; the migration patterns described here apply 1:1 to any new per-user tool added to the platform.

---

## Abstract

A per-user streaming toolset multiplies cost along two axes: **number of streamers** and **number of OBS overlays each one runs**. Naive polling architectures look fine at one user and quietly become tier-busting at ten. This note documents how we redesigned a Cloudflare Workers + KV + Durable Objects toolset so the cost curve flattens with users — replacing server-mediated polling with edge push, Hibernatable WebSockets, and Twitch EventSub. The result, for the now-playing audio widget alone, is a ~99.6% reduction in per-session KV reads, with no functional regression. The patterns generalize: any per-user real-time tool that today polls a worker can move the hot path off the server.

---

## 1. The scaling problem

The toolset is a static-export Next.js app served via Cloudflare Pages, backed by three Cloudflare Workers (`auth`, `spotify`, `overlay-do`) and a single KV namespace (`AUTH_KV`). The product surface is per-user: each Twitch streamer who installs the toolset runs ~8 OBS browser-source overlays — a now-playing widget, death counter, lurk-peek, BRB player, emote rain, clip play, video shout-out, chat box. Pre-migration, each overlay polled its corresponding worker endpoint every 5–10 seconds.

This shape — **N users × K tools each, every one polling the same workers** — is a multiplicative cost shape. Cloudflare's free tier and paid tier alike have *daily-rate* limits, not per-user limits. The architectural question is not "will this hit a limit" but "how does cost scale with users":

- **Polling architecture.** Each user's K tools generate fixed daily request and KV-read volume regardless of whether anything changed. Idle users still cost. The slope of the cost curve is the polling cadence × K.
- **Push architecture.** Cost per user is dominated by the cost of *delivering* events, not the cost of *checking* for them. An idle user — overlay open, nothing happening on stream — costs almost nothing.

A concrete sanity check: a single-user developer instance can burn **43,200 KV reads in 24 hours from one forgotten OBS tab** — five KV reads per now-playing poll × 360 polls/hour × 24 hours. That is ~43% of the free-tier daily KV-read budget *from one user, one tool, in steady state*. Multiply by 50 users and the daily budget on the **paid** tier is exhausted before half of them have gone live. The polling shape doesn't survive contact with users — it survives only with one user.

This is the framing for the migration: not "we got a cost alarm" — but "this shape will not scale, and we should rebuild it before we have users."

---

## 2. Cost shape and tier boundaries

The Cloudflare free tier (Workers Paid is $5/mo and umbrellas KV + Workers + Durable Objects):

| Resource | Free | Paid ($5/mo) |
|---|---|---|
| KV reads | 100k/day | 10M/month (~333k/day) |
| KV writes | 1k/day | 1M/month |
| Worker requests | 100k/day | 10M/month |
| DO requests | 100k/day | 1M/month + $0.15/M |
| DO duration | 13k GB-s/day | 400k GB-s/month + $12.50/M |

A naive polling architecture, projected to N users with K overlays per user × 5-second polls:

```
daily_worker_requests = N × K × (86400 / 5)
                      = N × K × 17,280
```

For N=10 users × K=4 active overlays: 691,200/day — already 7× the free-tier request limit, and 2× the paid tier's daily-equivalent. The KV-read multiplier compounds: each polled endpoint typically incurs 2–5 KV reads on a cache miss.

The architectural lesson: **the free-to-paid boundary is the wrong knob for a per-user-polling design.** Upgrading defers the wall; it does not move it.

---

## 3. Diagnosis

Pre-migration topology for the now-playing widget:

```
Browser (every 10s)
  └─→ Worker /now-playing
        └─→ KV: wid → twitchId          (read 1)
        └─→ KV: twitchId → spotify-creds (read 2)
        └─→ KV: twitchId → spotify-tokens (read 3)
        └─→ KV: twitchId → stream-settings (read 4, conditional)
        └─→ Spotify API (~200ms)
        └─→ Worker emits JSON
```

The worker maintained an in-isolate `Map` cache (5s TTL) for the response, but Cloudflare Workers run multiple isolates per data center. Two consecutive polls from the same browser frequently land on different isolates, defeating the cache. There was **no `caches.default` (edge cache) wrapping** — this is the same data center's shared cache, accessible across isolates.

Concurrent client-side issues compounded:

- `useNowPlaying.ts` ran `setInterval(poll, 10_000)` unconditionally. There was no `document.visibilityState` gate. A hidden OBS source or backgrounded browser tab polled forever.
- The polling interval was fixed at 10s regardless of player state. A streamer paused for 30 minutes still got 180 polls.

These are not micro-optimizations — they are the architectural shape that generates the cost.

---

## 4. Target architecture: three layers

### Layer A — Spotify now-playing (thin client, server out of the hot path)

The Spotify Web API gates `currently-playing` reads behind a per-user OAuth access token (1-hour TTL, refresh-token rotation). The token previously lived only in our worker, which is why the worker had to mediate every poll.

**The substitution:** the worker mints a short-lived access token on demand and returns it to the browser. The browser then polls Spotify directly, bypassing our worker entirely except for the periodic refresh.

```
Browser (once per ~50min)
  └─→ Worker POST /spotify/access-token
        └─→ KV: wid → twitchId           (read 1)
        └─→ KV: twitchId → spotify-creds  (read 2)
        └─→ KV: twitchId → spotify-tokens (read 3)
        └─→ Spotify token refresh (if needed)
        └─→ Returns { accessToken, expiresAt }

Browser (every 10s)
  └─→ Spotify API directly using the cached accessToken
```

KV reads per overlay-session collapse from ~3,600/hour to ~3 (one mint at session start, one refresh, perhaps one re-mint near 50-min boundary). Spotify's 30/min rate cap protects against runaway abuse downstream.

### Layer B — Chat-driven events (Durable Object + hibernatable WebSocket)

The other 7 overlays (death counter, !lurk, !clip-play, !shoutout, BRB, emote rain, etc.) react to discrete events triggered by chat commands or webhook deliveries. These are inherently push, not pull — the existing polling architecture was a workaround for the absence of push infrastructure.

A per-user Cloudflare Durable Object (`OverlayDO`, named by `twitchId`) holds in-memory state and accepts hibernatable WebSocket connections from each overlay. Twitch EventSub delivers webhooks to a callback in the auth worker, which validates the HMAC signature and dispatches to the user's DO via service binding. Bot-driven actions (a `!death` command, for example) flow through the same dispatch path: the auth worker's existing config-save endpoint adds a `OVERLAY_DO.fetch(...)` call after the KV `put`.

The DO broadcasts the event to all subscribed WebSockets, filtered by overlay kind:

```
[event source: bot / dashboard / EventSub]
   ↓ HTTPS POST (service binding, near-zero latency)
[auth worker: HMAC verify, route]
   ↓ HTTPS POST /by-user/<twitchId>/event (service binding)
[OverlayDO: applyEvent → broadcast]
   ↓ WebSocket message
[Subscribed overlay clients]
```

KV reads per chat-event drop to zero (the bot already wrote to KV; the DO doesn't read). WebSocket connections hibernate when idle — no DO duration billing accrues between events. Per Cloudflare's pricing notes, hibernation is the load-bearing feature: without it, an idle DO holding 8 connections for 4 hours would cost ~150k GB-seconds; with it, the same workload costs essentially zero.

### Layer C — KV: cold persistence only

After Layers A and B ship, KV reverts to its design role: read at session start, written on user-driven save events. The hot path no longer touches it. The metaphor is right: KV is a database, not a cache; it should not be on a polling loop.

### Important: wid as the access boundary

The `wid` (widget token) is a public, opaque identifier. Overlays in the browser carry it in the URL as `?wid=<wid>`. The DO routing (`/by-wid/<wid>/ws`) resolves wid → twitchId server-side via a single KV read; the twitchId never appears in client URLs. Same security model as before the migration; same threat surface.

---

## 5. Quantified outcomes (now-playing path)

Steady-state, single user, 4-hour streaming session:

| Metric | Pre-migration | Post-migration | Delta |
|---|---|---|---|
| KV reads (now-playing only) | ~14,400 | ~50 | **−99.65%** |
| KV reads (runaway tab, 24h) | ~43,200 | 0 | structurally eliminated |
| Worker requests (now-playing only) | ~14,400 | ~50 | **−99.65%** |
| Spotify API calls per overlay | ~14,400 | ~14,400 | unchanged |
| Hot-path KV reads per overlay-mount | ~5 per poll | 0 | path eliminated |

The Spotify side is unchanged — the upstream still receives the same poll volume — but it moves from our worker's quota into the streamer's per-user OAuth quota, where it belongs.

The visibility gate (`useDocumentHidden` hook) and quiet-state backoff (10s → 30s after 3 consecutive paused polls) further compress steady-state load and structurally eliminate the hidden-tab failure mode.

For the chat-driven overlays still using polling, the pre-migration projection at multi-overlay load is the constraint that motivates Layer B:

```
1 user × 6 chat overlays × (86400 / 5) = 103,680 worker requests/day
```

That single user already exceeds the free-tier daily request cap before any other traffic. At 10 users it exceeds the paid tier's daily allotment. Layer B (DO + WebSocket push) reduces this to a per-event count — a typical streaming session generates dozens of events, not tens of thousands of polls.

---

## 6. Patterns that generalize

### 6.1 Cache-layer composition

Cloudflare Workers expose three caching surfaces. Each has a distinct scope:

| Surface | Scope | Latency | Ideal use |
|---|---|---|---|
| In-isolate `Map` | One Worker isolate | ~0ms | Per-isolate dedup of identical concurrent requests |
| `caches.default` | One data center, all isolates | ~1ms | Per-data-center response cache for identical URLs |
| KV with `cacheTtl` | All edge data centers | ~5ms | Cross-data-center cache for identical keys |

A response that crosses isolate boundaries with no `caches.default` wrap is paying the full origin cost on every cross-isolate hop. The fix is one-liner-shaped — wrap the response, set a `Cache-Control` header, write through `ctx.waitUntil(cache.put(...))` — and it eliminates a class of cost that's invisible in development (where one isolate handles all traffic) but real in production.

### 6.2 Build-time vs runtime env vars

Next.js bakes `NEXT_PUBLIC_*` env vars into the static bundle at build time. With `output: "export"` on Cloudflare Pages, the dashboard env-var change does nothing until the next build. Conversely, runtime-only env vars (no `NEXT_PUBLIC_` prefix) are unavailable to client-side code in a static export — there is no Node.js process at request time to read them from.

The implication: **a static-export Pages app has no live runtime feature flags.** Every "flag" is either build-time (rebuild to flip) or moves into a separate config endpoint (which itself becomes a polling source). For migrations, this means flag-gated rollouts cost a redeploy — not zero — and the flag itself becomes part of the cost surface if read frequently.

In practice: this project introduced one build-time flag for Phase 1 rollout safety, then deleted it three days later when the migration was confirmed stable. The flag's value was real (rollback path, deploy-decoupling) but the long-term cost (extra branch in the source, dead code, baked literal in the bundle) wasn't worth keeping past verification. Flag-then-delete is a defensible idiom for migrations on this stack; flag-as-permanent is anti-pattern.

### 6.3 Push over polling for event-shaped data

For data that changes on discrete events (chat commands, webhook deliveries, user actions), polling is structurally wrong. Cloudflare's Hibernation API for Durable Objects is the supported path: WebSocket connections persist on the edge across DO eviction, in-memory state hydrates lazily on first event, duration billing accrues only during active CPU.

The architectural shift is from "client asks if anything changed" to "server tells client when something changed." The cost shape changes from `O(polls)` to `O(events)`, and at any non-trivial scale `events << polls` by orders of magnitude.

### 6.4 The free-to-paid umbrella

Workers Paid ($5/month) bundles KV, Durable Objects, R2, and Workers requests into a single subscription. Treating these as separate upgrades — a common confusion — leads to architectural deferral: "we'll add DOs later when we upgrade for them." There is no separate DO upgrade. There is one $5/mo umbrella. Designing around imagined separate tiers wastes architecture decisions on a fiction.

### 6.5 Visibility gating is non-negotiable

Any polling client that doesn't gate on `document.visibilityState` is a latent runaway. The cost when it fires is 24×-larger than steady state, and the trigger is "user forgot to close a tab" — guaranteed to happen eventually. The fix is ~10 lines of React. There is no excuse to ship without it.

---

## 7. Open questions and future work

- **Per-DO storage vs per-user KV.** The DO's SQLite-backed storage is faster and cheaper than KV for state that lives in the DO's lifecycle. Death counts, lurk queues, etc. likely belong in DO storage rather than persisting via the auth worker → KV path. Deferred until Tasks 10–14 expose concrete usage patterns.
- **Twitch EventSub Webhook vs WebSocket transport.** This project chose webhook because hibernation requires the DO to *not* hold an outgoing WebSocket. If the upstream events are dense, webhook overhead may be non-trivial; if rare, webhook is cleaner.
- **Edge image generation for OG cards.** `next/og`'s `ImageResponse` works with `output: "export"` when `dynamic = "force-static"` is exported. Generated PNGs are a few hundred ms at build time and ship as real static files. Worth standardizing as a pattern for OG/social images on Pages.
- **Automated cost regression tests.** Source-grep tests prove that visibility gating and `caches.default` wraps remain in the source. They do not catch a future change that shifts cost without removing the gate. A periodic CF analytics check (KV reads/day, worker requests/day) against a budget would close the loop.

---

## 8. Conclusion

The mission as stated — *"create a lower token-usage system that functions as well as it did before"* — has been met for the now-playing path with a single-digit-percentage of the original cost. The architectural shifts (thin client, edge cache, visibility gate, push over poll, Workers Paid umbrella) compose to a system that scales by user count rather than by overlay count × polling frequency.

The deeper lesson is that Cloudflare's free tier is a useful forcing function. It does not punish efficient designs and it surfaces inefficient ones early — when the cost is one alarm email, not a five-figure invoice. The 50% threshold caught a class of problem that would have been invisible at the paid tier until cost-per-user × user-count rendered the architecture commercially infeasible. Designing for the free tier is designing for the limit case; the paid tier is then headroom rather than a moving target.

For practitioners building per-user real-time tools on Cloudflare's edge, the patterns documented here — particularly cache-layer composition, push-over-poll for event-shaped data, and visibility gating — generalize beyond this project. They are the difference between an architecture that scales and one that lights up its own pager.

---

## Appendix A: Concrete commits (ordered)

| Phase | Commit | Description |
|---|---|---|
| 0 | `15d8a58` | Edge-cache `/now-playing` for 8s |
| 0 | `164f484` | Fix: in-isolate cache hit also sets `Cache-Control` |
| 0 | `851a8b8` | Add `useDocumentHidden` hook |
| 0 | `4b81e68` | Pause polling when hidden + quiet-state backoff |
| 1 | `3f2a6bb` | `POST /spotify/access-token` mint endpoint |
| 1 | `e145cb8` | Restore `rateLimitKey` helper (IP fallback) |
| 1 | `1c49177` | `useSpotifyAccessToken` hook |
| 1 | `c6ca711` | Floor refresh interval at 30s |
| 1 | `9ba0d05` | Route non-ok errors through `applyResponse` |
| 1 | `7341455` | Remove misleading `scope` field from response |
| 1 | `c4667c9` | Lower `/spotify/access-token` rate limit 30→5/min |
| 1+3 | `ca2056c` | Drop feature flag, retire `/now-playing` route |
| 2 | `b255776`–`b29f975` | DO substrate (8 commits) |
| 2 | `8c096a9` | `useUserStream` client hook |
| 2 | `3f4fa75` | Death-counter migration (canary) |

## Appendix B: Cost projection by user count

Assumptions: 4-hour streaming session per user per day, all 8 overlays connected.

| Users | Pre-migration (worker requests/day) | Post-migration (worker requests/day) | Tier |
|---|---|---|---|
| 1 | 138,240 | <100 | Free → Free |
| 10 | 1,382,400 | <1,000 | Paid → Free |
| 100 | 13,824,000 | <10,000 | Hard cap → Free |
| 1,000 | 138,240,000 | <100,000 | N/A → Free |
| 10,000 | 1.38 × 10⁹ | ~1M | N/A → Paid |

Pre-migration crosses the free-tier ceiling at one user, the paid tier's daily-equivalent at ten. Post-migration stays in free-tier headroom through 1,000 active streamers and crosses into paid only past 10,000 — a 10,000× shift in the user count at which the architecture changes tier.
