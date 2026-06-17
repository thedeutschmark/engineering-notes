# Cost-Per-User as a Design Constraint: Per-User Real-Time on Cloudflare's Edge

**Date:** 2026-05-23
**Author:** deutschmark

---

## Abstract

A per-user streaming toolset on Cloudflare's edge — Workers, KV, Durable Objects, Pages — designed from day one so that the operating bill scales with active users, not with overlay count × polling frequency. Each streamer runs ~eight OBS browser-source overlays; the hot path is push, not pull. The now-playing widget polls Spotify directly using a worker-minted short-lived token. The event-driven overlays subscribe to a per-user Durable Object via hibernatable WebSocket. A centralized auth worker at `auth.deutschmark.online` issues a single signed session that every product surface — toolset, dev portal, collab planner — consumes, so no per-product worker has to reimplement OAuth, refresh, or cookie scoping. KV is cold persistence: read on session start and on save, never on a poll. The shape changes cost from `O(N × K × polls/hour)` to `O(N × events/hour)`, where `events ≪ polls` at any non-trivial usage.

---

## 1. The constraint

Per-user real-time tooling has a specific cost shape. Each user runs `K` overlays. Each overlay either polls some upstream every `P` seconds or holds an open connection waiting for events. Naively, a polling architecture costs `O(N × K × polls/hour)` requests against your edge — a curve that crosses the free-tier ceiling at one active user and the paid-tier monthly budget somewhere between ten and one hundred. That curve is not a problem you can solve by tuning poll intervals or adding cache; it's the wrong shape, and the only fix is to change the shape.

The design constraint this paper documents: cost-per-user has to stay flat (or sublinear) as `N` grows, on a platform where every user runs the full set of overlays simultaneously, served from a free-tier-by-default Cloudflare account. That constraint shaped every architectural choice in §2 and §3 — not because anything broke at small scale, but because the curve would have made the product un-shippable at scale, and re-architecting after launch costs more than designing for the cliff from the start.

---

## 2. Architecture

Static-export Next.js on Cloudflare Pages, four Workers (`auth`, `spotify`, `overlay-do`, plus a small `toolkit-redirect` legacy-URL shim), one KV namespace (`AUTH_KV`). The per-user product surface is ~eight overlays: now-playing, death counter, lurk-peek, BRB player, emote rain, clip play, video shout-out, chat box. Several non-overlay surfaces — toolset dashboard, dev portal, collab planner — share the same session layer.

### 2.1 `auth.deutschmark.online` — the session boundary

A single Worker at `auth.deutschmark.online` is the only thing on the platform that talks to Twitch / Spotify / YouTube OAuth, mints sessions, refreshes tokens, and sets cookies. It issues a signed `dm_session` cookie scoped to `Domain=.deutschmark.online`, so every product subdomain (`toolset`, `dev`, `collab`, the apex) sees the same logged-in user without each shipping its own auth dance.

The shape is deliberate: per-product workers (`spotify`, `overlay-do`) don't implement OAuth at all. They accept either the SSO cookie or a service-binding call from `auth` and trust the session claim. New product surfaces inherit auth by virtue of being on the right domain — adding `dev.deutschmark.online` was a CORS allowlist line, not a re-implementation. The same worker handles widget-token issuance (the opaque `wid` strings in overlay URLs), Stripe webhook verification, and EventSub callback HMAC.

Centralizing this in one Worker also keeps the security boundary auditable: there's one place that knows how to read a session, one place that mints tokens, one place that talks to OAuth upstreams. Per-product workers can't accidentally accept an unsigned token because they don't have the verification key.

### 2.2 Layer A — Thin-client Spotify

The browser polls `api.spotify.com` directly. The `spotify` worker's role is reduced to minting a short-lived Spotify access token on demand.

```
Browser (once per ~50 min)
  └─→ Worker POST /spotify/access-token
        └─→ KV: wid → twitchId
        └─→ KV: twitchId → spotify-creds
        └─→ KV: twitchId → spotify-tokens
        └─→ Spotify token refresh (if needed)
        └─→ { accessToken, expiresAt }

Browser (every 10 s)
  └─→ Spotify API directly
```

The Spotify upstream poll volume is unchanged; it moves from the worker's quota into the streamer's per-user OAuth quota where it belongs. Spotify's 30/min rate cap protects against runaway abuse downstream. The hot path never crosses this project's KV or worker request budget — every poll the streamer's OBS makes is paid for by Spotify's infrastructure, not Cloudflare's.

### 2.3 Layer B — Durable Object substrate with hibernatable WebSocket

A per-user Durable Object (`OverlayDO`, named by `twitchId`) holds in-memory state. Overlays open WebSocket connections via `state.acceptWebSocket()` — Cloudflare's Hibernation API keeps the connection live across DO eviction; the class methods `webSocketMessage` / `webSocketClose` deliver events without holding CPU. DO duration billing accrues only while CPU is active.

Event sources converge on the same dispatch chain:

```
[bot / dashboard / EventSub]
   ↓ HTTPS POST
[auth worker: HMAC verify, route]
   ↓ service binding fetch
[OverlayDO: applyEvent → broadcast]
   ↓ WebSocket message
[subscribed overlay clients]
```

The DO broadcasts a typed event to all subscribed sockets, filtered by overlay kind. The event-driven overlays connect to this substrate; the EventSub registration covers `channel.raid`, `channel.follow`, and `channel.update` (the last drives Twitch-category autofill for the death counter). The KV→DO move for hot state was planned from the start as the shape the system should take once it had non-trivial event traffic; an early observability alarm just provided the trigger to cut over on a calm day rather than under pressure.

### 2.4 Layer C — KV as cold persistence

After Layers A and B, KV is touched in three places only: session-start config reads, user-driven save writes, and EventSub-secret verification on webhook delivery. The hot path — every poll, every event broadcast — does not touch KV. KV's role in this system is durable, infrequently-accessed configuration; not a request-path cache.

### 2.5 `wid` as the access boundary

Overlays carry an opaque `wid` (widget token) in URL query strings; the user's `twitchId` never appears in client URLs. The DO router resolves `/by-wid/<wid>/...` to `twitchId` server-side via one KV read; service-binding callers (auth, spotify worker) use the direct `/by-user/<twitchId>/...` path with no resolution step. The `wid` is also the unit of revocation — rotating a widget token invalidates only that overlay, not the user's session.

---

## 3. Patterns

### 3.1 Cache-layer composition

Workers expose three caching surfaces with distinct scopes:

| Surface | Scope | Latency | Use |
|---|---|---|---|
| In-isolate `Map` | One isolate | ~0 ms | Per-isolate dedup of identical concurrent requests |
| `caches.default` | One data center, all isolates | ~1 ms | Per-DC response cache for identical URLs |
| KV with `cacheTtl` | All edge data centers | ~5 ms | Cross-DC cache for identical keys |

A response that crosses isolate boundaries without a `caches.default` wrap pays the full origin cost on every cross-isolate hop. The fix is small — wrap the response, set `Cache-Control`, write through `ctx.waitUntil(cache.put(...))` — and removes a class of cost invisible in development (single isolate) but real in production.

### 3.2 Visibility gating for any polling client

A polling client without a `document.visibilityState` gate is a latent runaway. The cost-when-it-fires is roughly the day-budget burned by one forgotten browser tab. The fix is ~10 lines of React per polling site. There is no excuse to ship a poll loop without it.

The same logic does not transfer cleanly to WebSocket clients: they don't poll when hidden, but they do reconnect on transient network errors. A flapping WebSocket with exponential backoff topping at 30 s on a hidden tab costs ~2,880 KV reads/day per overlay if each reconnect resolves `wid → twitchId` against KV. The mitigation is in §5.

### 3.3 Push over polling for event-shaped data

For data that changes on discrete events (chat commands, webhook deliveries, user actions), polling is structurally wrong. Cloudflare's Hibernation API for Durable Objects is the supported path: WebSocket connections persist across DO eviction, in-memory state hydrates lazily on first event, duration billing accrues only during active CPU. Cost shape goes from `O(polls)` to `O(events)`.

### 3.4 Centralized auth as a platform discipline

Every product subdomain on `.deutschmark.online` shares one signed session, one cookie, one OAuth upstream contract. The discipline is platform-level: per-product workers can read the session claim but cannot mint one. New surfaces inherit auth by being on the right domain.

This is the inverse of the common "every app has its own auth" pattern. It costs slightly more cognitive load upfront — there's one Worker to maintain that owns a lot — but it removes per-product auth-bug surface, lets the security boundary be reviewed in one place, and means a Stripe webhook arriving for a user signed in on the toolset can identify them on the collab planner without any cross-app session shuttle.

### 3.5 Static-export feature flags

Next.js bakes `NEXT_PUBLIC_*` env vars into the static bundle at build time. With `output: "export"` on Cloudflare Pages, dashboard env-var changes do nothing until the next build; non-prefixed runtime env vars are unavailable to client-side code (no Node process at request time).

The implication: a static-export Pages app has no live runtime feature flags. Every "flag" is either build-time (rebuild to flip) or moves into a config endpoint (which itself becomes a polling source). Flag-then-delete on a build-time literal is a defensible idiom for migration rollout; flag-as-permanent is anti-pattern on this stack.

---

## 4. Cost shape

### 4.1 Cloudflare tier shape

| Resource | Free | Workers Paid ($5/mo) |
|---|---|---|
| KV reads | 100k/day | 10M/month (~333k/day) |
| KV writes | 1k/day | 1M/month |
| Worker requests | 100k/day | 10M/month |
| DO requests | 100k/day | 1M/month + $0.15/M |
| DO duration | 13k GB-s/day | 400k GB-s/month + $12.50/M |

Workers Paid is a single umbrella: KV, Durable Objects, R2, and Workers requests bill against one $5/mo subscription. Architectures that treat these as separate upgrades waste design budget on a fiction.

### 4.2 What this design actually costs to operate

Per-user, four-hour streaming session, all overlays connected. KV reads per session sum to roughly:

- 1 read for session-start auth (the SSO cookie's session claim resolves via one KV lookup)
- 1 read per overlay token resolution on connect (`wid → twitchId`), so ~K per session
- 1 read per Spotify token refresh (~5 per session, once per ~50 min)
- 1 read per webhook callback (EventSub HMAC secret verification — a couple per session)
- 1 read per overlay refetch triggered by a config-changed broadcast (rare; dashboard-driven)

That sums to ~50 KV reads per user per session for the typical case, with the long tail driven by event volume (chat commands, EventSub callbacks). Worker requests follow the same shape — a handful per session, scaling with events rather than polls.

### 4.3 Projection to user count

Per-user-per-day usage, all 8 overlays connected, 50 chat events per session, projected against the Workers Paid monthly budget:

| Active users | KV reads/day | Worker req/day | Headroom on $5/mo plan |
|---|---|---|---|
| 1 | ~50 | ~200 | 99.99% |
| 10 | ~500 | ~2,000 | 99.93% |
| 100 | ~5,000 | ~20,000 | 99.34% |
| 1,000 | ~50,000 | ~200,000 | 93% |
| 10,000 | ~500,000 | ~2M | 34% |

These are call-site estimates from the architecture in §2, not measurements. The shape is right; the absolute numbers will move with event mix. The Workers Paid tier (one $5/mo subscription covering Workers, KV, DO, and R2) absorbs through ~10,000 active streamers before DO duration or KV read overage become the binding constraint — and the §5 changes push the binding constraint further.

For context, a polling-style architecture at the same user count and overlay count crosses the *daily* free-tier ceiling at one active user and the Workers Paid *monthly* budget somewhere between ten and one hundred. The shape isn't a tuning problem; it's the difference between costing flat and costing quadratic.

---

## 5. Known optimization paths

The architecture in §2 is not the lower bound. Three changes would meaningfully reduce the event-driven KV cost:

- **Signed `wid → twitchId` claim.** The DO router currently resolves `wid` to `twitchId` via a KV read on every WebSocket connect/reconnect. A signed (HMAC) claim baked into the `wid` at issuance eliminates that read entirely. WebSocket reconnect storms drop to zero KV cost. Largest single hidden-cost source on the event path; the auth worker already mints and verifies enough signed material to make this a small extension rather than a new system.
- **DO SQLite for widget config.** Widget configuration currently lives in KV under `widget_token:<wid>`; every `config-changed` broadcast triggers connected overlays to refetch via KV. Moving config into the DO's SQLite storage and including the new state inside the broadcast payload eliminates the post-event refetch storm.
- **Coalesced broadcast payload.** The current `config-changed` event carries a `key` field; overlays then refetch. Embedding the full new state in the event removes the refetch round-trip entirely. Pairs naturally with the SQLite-storage change above.

### Surviving polling

Two dashboard-side polls remain after the overlay migration: `useSupportPool` (60 s, fetches the community-fund total) and the local-loopback chat-bot health check (5 s against `localhost`). The first is candidate-for-SSE; the second does not consume Cloudflare quota.

---

## 6. Cost model and funding shape

Cloudflare cost is not the only operating cost. The full per-month operating expense (DNS, Workers Paid, Pages, occasional R2, Stripe processing) sits at a known floor `F`, and grows roughly linearly with active users `N` once usage clears the free-tier shoulders:

```
T(N) = F + c · N
```

Concretely on this platform: `F ≈ $6` and `c ≈ $0.03` per active user per month, derived from the §4 usage shape and the Workers-Paid overage rates. That cost model is the input to the platform's `/support` community-fund target — the funding ask scales with active user count rather than being a fixed number that drifts out of relevance — and recomputes monthly so the goal tracks reality. The design discipline this enforces is symmetric to the technical one: the bill is allowed to grow, but it has to grow in proportion to something users can see (themselves), not as a surprise.

---

## 7. Conclusion

The system is push, not pull. Polling on the hot path is replaced with a worker-minted Spotify token (the browser polls the upstream directly) and a per-user Durable Object holding hibernatable WebSockets (event sources dispatch through service bindings; the DO broadcasts to subscribed overlays). KV is cold persistence — read on session start, written on user-driven save events. A centralized auth Worker at `auth.deutschmark.online` issues one signed session that every product surface consumes, so per-product workers don't reimplement OAuth.

The cost shape that result composes — flat in poll frequency, linear in event volume, sublinear in per-user fixed cost amortized across the platform — is the design constraint, not a happy accident. It was chosen from the start because the polling alternative would have made the product un-shippable past the free-tier shoulders, and re-architecting after launch costs more than designing for the cliff at the start.
