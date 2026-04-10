# Multi-Signal Collab Detection for Twitch

How I built a confidence-ranked system that detects past collaborations between Twitch streamers using four independent signal sources — and why the "never downgrade" upsert policy makes it work.

## The problem

I built a [collab planner](https://collab.deutschmark.online) for Twitch streamers. A core feature is collab history — showing who you've collaborated with and when. The obvious approach is to let users manually log every collab. Nobody does this.

So the system needs to detect collabs automatically from available data. The challenge: there's no single reliable signal. The Twitch API doesn't have a "these two people collaborated" endpoint. You have to triangulate from multiple imperfect sources and decide how much to trust each one.

## Signal sources and their failure modes

I ended up with four tiers, ranked by reliability:

```
Tier 1: confirmed_event     User explicitly created an event with participants
Tier 2: guest_star          Twitch Guest Star API session records
Tier 3: vod_title_mention   @mentions or "collab" keywords in VOD titles
Tier 4: stream_overlap      Both streamers live at the same time, ≥30 min
```

Each has different failure modes:

**Confirmed events** are ground truth but require manual input. Most streamers don't pre-plan — they just raid someone and start streaming together.

**Guest Star** is Twitch's native collab feature. The API exists (`/helix/guest_star/session`) but requires `channel:read:guest_star` scope on channels you own or moderate. You can't query other people's guest star history. Not all collabs use Guest Star anyway.

**VOD title mentions** catch collabs that streamers announce in their title ("Playing Lethal Company w/ @Alice"). High signal when present, but many streamers don't update their title, use display names instead of handles, or write titles in other languages.

**Stream overlap** is the broadest net — if two people were live at the same time for 30+ minutes, maybe they were together. But two streamers can be live simultaneously while playing completely different games in different time zones. This is the noisiest signal.

## The confidence hierarchy

Every detected signal gets a confidence level:

```typescript
const CONFIDENCE_RANK: Record<string, number> = {
  high: 3,
  medium: 2,
  weak: 1,
};
```

The mapping:

```
Signal                         Condition                    Confidence
──────────────────────────────────────────────────────────────────────
confirmed_event                always                       high
guest_star                     always                       high
vod_title_mention (@handle)    explicit @mention in title   high
vod_title_mention (keyword)    name + collab keyword        high
stream_overlap                 same game_id                 high
stream_overlap                 different games              medium
```

A bare name mention *without* a collab keyword is deliberately **not persisted**. Early versions did this and generated false positives like "practicing w/ no mic" matching a friend named "mic" or "playing with fire" matching a friend named "fire."

## The never-downgrade rule

This is the design decision that makes the whole system work. When persisting a signal:

```typescript
for (const s of signals) {
  const existing = await prisma.collabSignal.findFirst({
    where: {
      friendId: s.friendId,
      partnerLogin: s.partnerLogin,
      detectedAt: s.detectedAt,
    },
  });

  if (
    existing &&
    (CONFIDENCE_RANK[existing.confidence] ?? 0) >
      (CONFIDENCE_RANK[s.confidence] ?? 0)
  ) {
    continue; // never overwrite higher confidence with lower
  }

  await prisma.collabSignal.upsert({ ... });
}
```

Why this matters: detection runs repeatedly — on schedule refresh, stream history fetch, and manual trigger. Without the guard, a re-run could downgrade a user-confirmed collab to a medium-confidence stream overlap. The never-downgrade policy means ground truth is permanent. You can re-run detection as often as you want without corrupting good data.

The composite unique key `(friendId, partnerLogin, detectedAt)` allows multiple signals from the same partner on different dates while preventing duplicates for the same date.

## Detection pipeline

### Stage 1: VOD title analysis

Two parallel passes over a friend's VOD history:

**Pass A — @mention extraction:**

```typescript
const mentionRegex = /@([a-zA-Z0-9_]{4,25})/g;

for (const vod of vods) {
  for (const match of vod.title.matchAll(mentionRegex)) {
    const handle = match[1].toLowerCase();
    if (isSelfReference(handle, subject)) continue;

    const friend = activeFriends.find(
      f => f.username.toLowerCase() === handle
    );
    if (friend) {
      signals.push({
        source: "vod_title_mention",
        confidence: "high",
        evidence: vod.title,
        // ...
      });
    }
  }
}
```

@mentions are high confidence because streamers rarely @-mention someone in their title unless they're actually collaborating.

**Pass B — keyword + name combo:**

```typescript
const COLLAB_KEYWORDS = [
  "collab", "collaboration", "ft.", "feat.", "with ",
  " w/ ", " w/", "guest", "duo", "trio", "squad",
  "together", "joined by", "join", "hosted",
  "stream together", "co-stream", "co stream",
];

for (const vod of vods) {
  const lower = vod.title.toLowerCase();
  const hasKeyword = COLLAB_KEYWORDS.some(kw => lower.includes(kw));
  if (!hasKeyword) continue;

  for (const friend of activeFriends) {
    if (isSelfReference(friend.username, subject)) continue;
    const nameInTitle =
      lower.includes(friend.username.toLowerCase()) ||
      (friend.displayName.length >= 3 &&
        lower.includes(friend.displayName.toLowerCase()));

    if (nameInTitle) {
      signals.push({
        source: "vod_title_mention",
        confidence: "high",
        evidence: vod.title,
        // ...
      });
    }
  }
}
```

The **keyword gate** is critical. Without it, any title containing a friend's name would register as a collab. With it, you need both a collab-indicating keyword AND the friend's name. Display names have a minimum length check (≥3 chars) to avoid matching common words.

### Stage 2: Confirmed events

Events are user-created and have explicit participant lists:

```typescript
const confirmedCollabs = await prisma.collabHistory.findMany({
  where: { friendId, eventId: { not: null } },
  include: {
    event: { include: { participants: { include: { friend: true } } } },
  },
});

for (const collab of confirmedCollabs) {
  for (const participant of collab.event.participants) {
    if (participant.friend.isMe) continue;
    if (participant.friendId === friendId) continue;

    signals.push({
      source: "confirmed_event",
      confidence: "high",
      evidence: `Event: ${collab.event.title} on ${collab.event.startTime}`,
      // ...
    });
  }
}
```

Since these come from user input, they're always high confidence. The never-downgrade rule means once an event is confirmed, no amount of re-running title parsing can change it.

### Stage 3: Stream overlap

The most algorithmically interesting stage. For each pair of friends with VOD history, check for temporal overlap:

```typescript
const MIN_OVERLAP_MS = 30 * 60 * 1000; // 30 minutes

for (const myVod of myVods) {
  const myStart = myVod.startedAt.getTime();
  const myEnd = myStart + parseDuration(myVod.duration);

  for (const theirVod of theirVods) {
    const theirStart = theirVod.startedAt.getTime();
    const theirEnd = theirStart + parseDuration(theirVod.duration);

    const overlap = Math.min(myEnd, theirEnd) -
                    Math.max(myStart, theirStart);

    if (overlap >= MIN_OVERLAP_MS) {
      const sameGame = myVod.gameId &&
                       myVod.gameId === theirVod.gameId;

      signals.push({
        source: "stream_overlap",
        confidence: sameGame ? "high" : "medium",
        evidence: sameGame
          ? `Both streamed ${myVod.gameName} for ${Math.round(overlap / 60000)} min`
          : `Overlapping streams for ${Math.round(overlap / 60000)} min`,
        // ...
      });
    }
  }
}
```

**Game matching** is the confidence differentiator. Two people streaming the same game at the same time is a much stronger signal than two people streaming different games simultaneously. Same-game overlap gets high confidence; different-game overlap gets medium.

**The 30-minute threshold** filters out brief overlaps. If Alice streams 2-10pm and Bob streams 9:55-11pm, the 5-minute overlap isn't meaningful. 30 minutes is long enough that if they're playing the same game, they're probably playing together.

### Self-reference filtering

Every stage filters out self-references to prevent a streamer from being detected as collaborating with themselves:

```typescript
function isSelfReference(
  handleOrName: string,
  subject: { username: string; displayName: string }
): boolean {
  const norm = handleOrName.toLowerCase();
  return (
    norm === subject.username.toLowerCase() ||
    norm === subject.displayName.toLowerCase()
  );
}
```

This catches the case where a streamer's title says "w/ @themselves" or their VOD title contains their own display name.

## Data model

```prisma
model CollabSignal {
  id           Int      @id @default(autoincrement())
  friendId     Int
  partnerName  String
  partnerLogin String   @default("")
  detectedAt   DateTime
  source       String        // "confirmed_event" | "guest_star" | "vod_title_mention" | "stream_overlap"
  evidence     String        // human-readable proof
  confidence   String        // "high" | "medium" | "weak"
  fetchedAt    DateTime @default(now())
  friend       Friend   @relation(fields: [friendId], references: [id], onDelete: Cascade)

  @@unique([friendId, partnerLogin, detectedAt])
}
```

The composite unique key `[friendId, partnerLogin, detectedAt]` is the backbone of the upsert logic. It says: for a given friend-partner pair on a given date, there's exactly one signal. If we detect the same collab from multiple sources (title mention AND stream overlap), the higher-confidence one wins.

## Execution triggers

Detection doesn't run on a timer. It piggybacks on data fetches:

```
Stream history refresh → fetchAndStoreStreamHistory() → detectCollabSignals()
Schedule sync          → refreshSchedules()           → detectCollabSignals()
Manual trigger         → POST /api/friends/[id]/collabs
Bulk sync              → Promise.allSettled(friends.map(detectCollabSignals))
```

`Promise.allSettled` is important for the bulk path — one friend's Twitch API error shouldn't block detection for the other 30.

## Twitch API constraints that shaped the design

1. **No public VOD access for arbitrary users.** You can only fetch VODs for users whose IDs you know. The system requires friends to be registered first.

2. **Game data is unreliable.** Helix often returns `null` for `game_id` on archived streams. The system falls back to Twitch's undocumented GQL endpoint to pull chapter markers from VODs.

3. **Guest Star is scope-locked.** `channel:read:guest_star` only works on channels you own or moderate. You can't query whether your friend had guest stars on their channel. This limits Guest Star detection to the authenticated user's own channel.

4. **50-VOD window.** Helix returns up to 100 videos per request. The system caps at 50 most recent to keep detection fast. Collabs older than ~50 streams back won't be rediscovered unless they recur.

## Results and tradeoffs

**What works well:**
- @mentions are nearly perfect — streamers who use them in titles get accurate, high-confidence collab history with zero manual input.
- The keyword gate eliminated false positives from name-only title matching.
- Never-downgrade means users can trust that manually confirmed collabs won't get clobbered.
- Running detection on every data fetch means the system is always up to date without a separate cron job.

**Known limitations:**
- English-only keyword list. Non-English streamers' collab announcements are missed.
- Stream overlap without game matching produces medium-confidence noise. Two streamers in different time zones who happen to stream simultaneously get flagged.
- Guest Star integration is stubbed but not wired. Requires adding an OAuth scope and handling broadcaster-level tokens.
- The system can't detect collabs that weren't recorded as VODs (stream not archived, VOD deleted).

**What I'd do differently:**
- Add a feedback loop: let users dismiss false positives, which would train a per-user keyword blocklist.
- Weight overlap confidence by recency — two people who overlapped 3 times this month are more likely collaborating than two people who overlapped once 6 months ago.
- Consider fuzzy name matching (Levenshtein distance) for display names, with a higher confidence threshold to compensate for the fuzziness.

## The philosophy

The system is designed around a pragmatic insight: **it's better to surface a collab with medium confidence than to miss it entirely.** Users can always confirm or dismiss. The never-downgrade rule means the system gets more accurate over time as users interact with it, never less.

The four-tier hierarchy isn't about finding one perfect signal — it's about layering imperfect signals so that in aggregate, the system catches most real collabs while keeping false positives manageable.

## Running it

This is part of [Twitch Friends Organizer](https://collab.deutschmark.online). The detection engine is at `lib/twitch/detectCollabs.ts`. The API endpoint at `POST /api/friends/[id]/collabs` triggers detection and returns the full signal list.
