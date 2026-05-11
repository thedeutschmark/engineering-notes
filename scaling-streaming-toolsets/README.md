# Scaling Per-User Streaming Toolsets on Cloudflare — Edge Push, Hibernation, and Flat Cost-Per-User

**Date:** 2026-05-11
**Author:** deutschmark

---

## Abstract

A per-user streaming toolset on Cloudflare's edge — Workers, KV, Durable Objects, Pages — composed so cost-per-user stays roughly flat as user count grows. Each streamer runs eight OBS browser-source overlays; the hot path is push, not pull. The now-playing widget polls Spotify directly using a worker-minted short-lived token, eliminating server-mediated polling. The seven event-driven overlays subscribe to a per-user Durable Object via hibernatable WebSocket; dashboard saves and Twitch EventSub webhooks dispatch through service bindings. KV is cold persistence — read on session start and on save, never on a poll. The shape changes cost from `O(N × K × polls/hour)` to `O(N × events/hour)`, where `events ≪ polls` at any non-trivial usage.

---

## 1. Architecture

Static-export Next.js on Cloudflare Pages, three Workers (`auth`, `spotify`, `overlay-do`), one KV namespace (`AUTH_KV`). The per-user product surface is eight overlays: now-playing, death counter, lurk-peek, BRB player, emote rain, clip play, video shout-out, chat box.

### 1.1 Layer A — Thin-client Spotify

The browser polls `api.spotify.com` directly. The worker's role is reduced to minting a short-lived Spotify access token on demand.

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

The Spotify upstream poll volume is unchanged; it moves from the worker's quota into the streamer's per-user OAuth quota where it belongs. Spotify's 30/min rate cap protects against runaway abuse downstream.

### 1.2 Layer B — Durable Object substrate with hibernatable WebSocket

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

The DO broadcasts a typed event to all subscribed sockets, filtered by overlay kind. The 7 event-driven overlays are connected to this substrate; the EventSub registration covers `channel.raid`, `channel.follow`, and `channel.update` (the last drives Twitch-category autofill for the death counter).

### 1.3 Layer C — KV as cold persistence

After Layers A and B, KV is touched in three places only: session-start config reads, user-driven save writes, and EventSub-secret verification on webhook delivery. The hot path — every poll, every event broadcast — does not touch KV.

### 1.4 `wid` as the access boundary

Overlays carry an opaque `wid` (widget token) in URL query strings; the user's `twitchId` never appears in client URLs. The DO router resolves `/by-wid/<wid>/...` to `twitchId` server-side via one KV read; service-binding callers (auth, spotify worker) use the direct `/by-user/<twitchId>/...` path with no resolution step.

---

## 2. Cost analysis

### 2.1 Cloudflare tier shape

| Resource | Free | Workers Paid ($5/mo) |
|---|---|---|
| KV reads | 100k/day | 10M/month (~333k/day) |
| KV writes | 1k/day | 1M/month |
| Worker requests | 100k/day | 10M/month |
| DO requests | 100k/day | 1M/month + $0.15/M |
| DO duration | 13k GB-s/day | 400k GB-s/month + $12.50/M |

Workers Paid is a single umbrella: KV, Durable Objects, R2, and Workers requests are billed against one $5/mo subscription. Architectures that treat these as separate upgrades waste design budget on a fiction.

### 2.2 Now-playing path (measured)

Steady-state, single user, four-hour streaming session:

| Metric | Naive polling | Thin-client | Δ |
|---|---|---|---|
| KV reads | ~7,200 | ~20 | −99.7% |
| Worker requests | ~1,440 | ~5 | −99.65% |
| KV reads per poll | 5 | 0 | path eliminated |
| Hidden-tab 24h burn (KV reads) | ~43,200 | 0 | structurally eliminated |
| Spotify API calls per overlay-hour | ~360 | ~360 | unchanged (moved to user's OAuth quota) |

Naive baseline is 10-second polling without an edge cache. The hidden-tab elimination is the combination of two changes: a `document.visibilityState` gate on the poll loop and the structural shift that the runaway-tab path (5 KV reads × 360 polls/hour × 24 hours = 43,200) now lives entirely client-side against Spotify, not against this project's KV.

### 2.3 Event-driven paths (projected)

The seven event-driven overlays exchanged their polling endpoints for WebSocket subscriptions. Per-event cost on the post-migration architecture, accounted at the call-site level:

- EventSub callback verifies a per-subscription secret: 1 KV read.
- DO routing resolves `wid → twitchId` on each WebSocket connect/reconnect: 1 KV read.
- `channel.update` fan-out across `M` death-counter widget tokens: 1 (index) + M (per-record) KV reads.
- Each `config-changed` broadcast triggers an overlay refetch: 1 KV read per connected overlay.

So a single `!death`-style chat event costs `1 + M + K·M` KV reads where `M` is widget-token count and `K` is connected overlays per token. This is non-zero but bounded by event frequency, not polling frequency. A 4-hour streaming session with 50 chat events and one overlay per token costs ~150 KV reads on the event-driven path — versus the naive-polling projection below.

| Users | Naive (worker req/day) | Architecture as deployed |
|---|---|---|
| 1 | 138,240 | ~200 |
| 10 | 1.38M | ~2,000 |
| 100 | 13.8M | ~20,000 |
| 1,000 | 138M | ~200,000 |
| 10,000 | 1.38B | ~2M |

Naive baseline: 8 overlays × 5-second polling × 4-hour session. Per-architecture-as-deployed estimates assume 50 chat events per session and the call-site KV cost above. These are projections, not measurements — see §4 for the architectural changes that would tighten them further.

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

The same logic does not transfer cleanly to WebSocket clients: they don't poll when hidden, but they do reconnect on transient network errors. A flapping WebSocket with exponential backoff topping at 30 s on a hidden tab costs ~2,880 KV reads/day per overlay if each reconnect resolves `wid → twitchId` against KV. The mitigation is in §4.

### 3.3 Push over polling for event-shaped data

For data that changes on discrete events (chat commands, webhook deliveries, user actions), polling is structurally wrong. Cloudflare's Hibernation API for Durable Objects is the supported path: WebSocket connections persist across DO eviction, in-memory state hydrates lazily on first event, duration billing accrues only during active CPU. Cost shape goes from `O(polls)` to `O(events)`.

### 3.4 Static-export feature flags

Next.js bakes `NEXT_PUBLIC_*` env vars into the static bundle at build time. With `output: "export"` on Cloudflare Pages, dashboard env-var changes do nothing until the next build; non-prefixed runtime env vars are unavailable to client-side code (no Node process at request time).

The implication: a static-export Pages app has no live runtime feature flags. Every "flag" is either build-time (rebuild to flip) or moves into a config endpoint (which itself becomes a polling source). Flag-then-delete on a build-time literal is a defensible idiom for migration rollout; flag-as-permanent is anti-pattern on this stack.

---

## 4. Known optimization paths

The architecture-as-deployed in §1 is not the lower bound. Three changes would meaningfully reduce the event-driven KV cost of §2.3:

- **Signed `wid → twitchId` claim.** The DO router currently resolves `wid` to `twitchId` via a KV read on every WebSocket connect/reconnect. A signed (HMAC) claim baked into the wid at issuance eliminates that read entirely. WebSocket reconnect storms drop to zero KV cost. Largest single hidden-cost source on the event path.
- **DO SQLite for widget config.** Widget configuration currently lives in KV under `widget_token:<wid>`; every `config-changed` broadcast triggers `K` overlays to refetch via KV. Moving config into the DO's SQLite storage and including the new state inside the broadcast payload eliminates the post-event refetch storm. The `K·M` factor in §2.3's accounting collapses to zero.
- **Coalesced broadcast payload.** The current `config-changed` event carries a `key` field; overlays then refetch. Embedding the full new state in the event removes the refetch round-trip entirely. Pairs naturally with the SQLite-storage change above.

### Surviving polling

Two dashboard-side polls remain after the overlay migration: `useSupportPool` (60 s, fetches the community-fund total) and the chat-bot health check (5 s against `localhost`). The first is candidate-for-SSE; the second is local-loopback only and does not consume Cloudflare quota.

---

## 5. Cost projection by user count

Assumptions: 4-hour streaming session per user per day, all 8 overlays connected, 50 chat events per session.

| Users | Naive polling (worker req/day) | Architecture-as-deployed | With §4 optimizations applied |
|---|---|---|---|
| 1 | 138,240 | ~200 | ~50 |
| 10 | 1.38M | ~2,000 | ~500 |
| 100 | 13.8M | ~20,000 | ~5,000 |
| 1,000 | 138M | ~200,000 | ~50,000 |
| 10,000 | 1.38B | ~2M | ~500k |

Naive crosses the free-tier daily ceiling at one user, the paid-tier daily-equivalent at ten. Architecture-as-deployed stays in free-tier headroom through 1,000 active streamers; with §4 applied, through 10,000.

---

## 6. Conclusion

The system is push, not pull. Polling on the hot path is replaced with a worker-minted Spotify token (the browser polls the upstream directly) and a per-user Durable Object holding hibernatable WebSockets (event sources dispatch through service bindings; the DO broadcasts to subscribed overlays). KV is cold persistence — read on session start, written on user-driven save events.

The now-playing path is measured; the event-driven path is projected at the call-site level, with three named architectural changes in §4 that would tighten those projections further. For per-user real-time tools on Cloudflare's edge, the patterns documented here — cache-layer composition, visibility gating on poll loops, push for event-shaped data, signed claims over KV lookups on the connect path — compose to a system that scales with user count, not with overlay count × polling frequency.
