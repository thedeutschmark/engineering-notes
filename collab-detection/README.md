# Multi-Signal Collab Detection for Twitch

How I built a confidence-ranked system for detecting likely collaborations between Twitch streamers from multiple imperfect signals, and why the persistence rules mattered as much as the detection logic.

## The problem

I built a [collab planner](https://collab.deutschmark.online) for Twitch streamers. One of the core features is collab history: who someone has streamed with, and when.

The obvious version is manual entry. In practice that does not scale. Streamers rarely log every collab, and many of the most useful signals only appear after the fact in titles, VODs, or event data.

There is also no single authoritative source. Twitch does not expose a universal "these two creators collaborated" endpoint, so the system has to infer from several sources with different failure modes.

## Signal tiers

I ended up with four sources, ordered by reliability:

```text
Tier 1: confirmed_event     user-created event with explicit participants
Tier 2: guest_star          Twitch Guest Star session data
Tier 3: vod_title_mention   @mentions or title patterns indicating a collab
Tier 4: stream_overlap      both streamers live at the same time for >= 30 min
```

Each source tells a different kind of truth:

- `confirmed_event` is explicit and user-authored
- `guest_star` is platform-native but scope-limited
- `vod_title_mention` is strong when present but inconsistently used
- `stream_overlap` is broad coverage with the most noise

That matters because the downstream design is not "find one perfect signal." It is "combine imperfect signals, preserve the strongest evidence, and avoid corrupting good data on later runs."

## Confidence mapping

Each detected signal is assigned a confidence level:

```typescript
const CONFIDENCE_RANK: Record<string, number> = {
  high: 3,
  medium: 2,
  weak: 1,
};
```

The mapping is:

```text
confirmed_event                -> high
guest_star                     -> high
vod_title_mention (@handle)    -> high
vod_title_mention (keyword)    -> high
stream_overlap + same game     -> high
stream_overlap + different game -> medium
```

Here, `high` does not mean "ground truth in the abstract." It means "strong enough within this system to win conflict resolution and surface confidently in the UI."

One important exclusion: a bare name mention without a collab keyword is not persisted. That version produced too many false positives early on.

## The persistence rule that made the system stable

The key design choice was simple: never replace a stronger signal with a weaker one.

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
    continue;
  }

  await prisma.collabSignal.upsert({ ... });
}
```

Detection runs repeatedly: after stream-history refreshes, schedule syncs, and manual triggers. Without this guard, a later low-signal pass could overwrite a confirmed event with a weaker overlap-based inference. With it, the system becomes monotonic in the useful direction: better evidence can replace weaker evidence, but weaker evidence cannot erode stronger evidence.

That one rule made reprocessing safe.

## Stage 1: VOD title analysis

The first stage looks at title text in two passes.

### Pass A: explicit `@mentions`

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
      });
    }
  }
}
```

This was the cleanest inferred signal in practice. Streamers do not usually `@mention` someone in a title unless that person is relevant to the stream.

### Pass B: keyword-gated name matching

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
      });
    }
  }
}
```

The keyword gate is what kept this useful. Matching on names alone was too permissive. Requiring both a collab-like phrase and a candidate name pushed the precision high enough to keep.

## Stage 2: confirmed events

User-created events provide explicit participant lists:

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
    });
  }
}
```

These are always treated as high confidence. More importantly, once persisted, they remain protected from later lower-confidence passes because of the never-downgrade rule.

## Stage 3: stream overlap

Stream overlap is the noisiest signal and the broadest net:

```typescript
const MIN_OVERLAP_MS = 30 * 60 * 1000;

for (const myVod of myVods) {
  const myStart = myVod.startedAt.getTime();
  const myEnd = myStart + parseDuration(myVod.duration);

  for (const theirVod of theirVods) {
    const theirStart = theirVod.startedAt.getTime();
    const theirEnd = theirStart + parseDuration(theirVod.duration);

    const overlap = Math.min(myEnd, theirEnd) -
                    Math.max(myStart, theirStart);

    if (overlap >= MIN_OVERLAP_MS) {
      const sameGame = myVod.gameId && myVod.gameId === theirVod.gameId;

      signals.push({
        source: "stream_overlap",
        confidence: sameGame ? "high" : "medium",
        evidence: sameGame
          ? `Both streamed ${myVod.gameName} for ${Math.round(overlap / 60000)} min`
          : `Overlapping streams for ${Math.round(overlap / 60000)} min`,
      });
    }
  }
}
```

The 30-minute threshold filters out incidental overlap. Game matching then separates stronger inferences from weaker ones. Two people streaming the same game at the same time is still not proof, but it is materially stronger than two unrelated streams that happened to overlap.

## Self-reference filtering

Every stage uses the same self-reference check:

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

That prevents common nonsense cases like a streamer being detected as collaborating with themselves because their own name appears in the title.

## Data model

```prisma
model CollabSignal {
  id           Int      @id @default(autoincrement())
  friendId     Int
  partnerName  String
  partnerLogin String   @default("")
  detectedAt   DateTime
  source       String
  evidence     String
  confidence   String
  fetchedAt    DateTime @default(now())
  friend       Friend   @relation(fields: [friendId], references: [id], onDelete: Cascade)

  @@unique([friendId, partnerLogin, detectedAt])
}
```

The composite unique key is what makes the upsert semantics clean: one friend-partner-date record, many possible ways to discover it, strongest evidence wins.

## Execution model

Detection is triggered by data activity rather than a separate background scheduler:

```text
stream history refresh -> fetchAndStoreStreamHistory() -> detectCollabSignals()
schedule sync          -> refreshSchedules()           -> detectCollabSignals()
manual trigger         -> POST /api/friends/[id]/collabs
bulk sync              -> Promise.allSettled(...)
```

`Promise.allSettled` matters on the bulk path. One friend's API failure should not block the rest of the batch.

## Twitch constraints that shaped the system

Several API limits pushed the design in this direction:

1. VOD history is only useful once a user is known to the system.
2. Game metadata is inconsistent enough that archived-stream enrichment sometimes needs a fallback path.
3. Guest Star data is scope-locked; you cannot use it as a universal public signal.
4. Historical windows are finite, so very old collabs may only reappear if they recur or were explicitly confirmed.

Those constraints are why the system leans so heavily on repeatable inference plus durable persistence rules.

## What worked and what did not

What held up well:

- Explicit `@mentions` were the highest-precision inferred signal
- The keyword gate prevented most name-only false positives
- Never-downgrade kept reruns safe
- Piggybacking on existing fetch paths kept the data fresh without adding scheduler complexity

What is still imperfect:

- The keyword list is English-only
- Overlap-based inference still produces some medium-confidence noise
- Guest Star support is structurally limited by OAuth scope
- Deleted or unarchived streams are invisible to this approach

## What I would add next

- User feedback loops for false positives and dismissals
- Recency weighting for repeated overlaps
- Better display-name matching with careful thresholds
- Language-specific keyword packs instead of one English-centric list

## Running it

This is part of [Twitch Friends Organizer](https://collab.deutschmark.online). The detection engine lives in `lib/twitch/detectCollabs.ts`, and `POST /api/friends/[id]/collabs` triggers a full run and returns the detected signal set.
