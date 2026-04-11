# Multi-Signal Collab Detection for Twitch

*Confidence-ranked collaboration inference from multiple imperfect sources, with rerun-safe persistence.*

I built a Twitch collab planner, and one of the first hard problems was collab history.

The useful version of that feature is not just "who follows who" or "who appeared in a title once." It is a reasonably trustworthy record of **who likely streamed together and when**.

Twitch does not hand you that as one clean data source, so the system had to infer it from multiple imperfect signals.

---

## The problem

Manual entry sounds simple until you try to scale it. People do not log every collaboration, and the most useful clues often show up after the fact in titles, VOD metadata, scheduling artifacts, or platform-specific session data.

The challenge was not to find a perfect signal. It was to **combine weak and strong signals** without letting noisy reruns degrade better data that already existed.

That second part turned out to matter as much as the detection logic itself.

## The signal model

The system uses four signal sources, each with an assigned confidence tier:

| Source | Confidence | What it catches |
|--------|:----------:|----------------|
| Confirmed events | **high** | A planned event marked completed with both streamers as participants. Ground truth. |
| Title @mentions | **high** | An @handle in a past broadcast title. Strong because streamers tag collaborators intentionally. |
| Title + keyword context | **high** | A friend's name appears alongside collab language ("ft.", "w/", "collab", "duo"). Gated to reduce false positives. |
| Stream overlap | **varies** | Both streamers were live at the same time for a meaningful duration. Same-game overlap is stronger evidence than unrelated streams. |

Each one tells a different kind of truth. Some signals are explicit but sparse. Some are broad but noisy. Some are very convincing when present but inconsistently used.

That meant the system needed to **rank evidence** rather than pretend every hit meant the same thing.

---

## The rule that made the system stable

The most important design choice was simple:

> Stronger evidence should be allowed to replace weaker evidence, but weaker evidence should not be allowed to overwrite stronger evidence later.

In code, every signal carries a numeric confidence rank. On upsert, the system checks whether a signal already exists for the same partner on the same date. If the existing signal has higher confidence, the new signal is skipped. If the incoming signal is at least as strong, it updates.

That sounds obvious, but it changes the whole feel of the system.

Detection runs repeatedly. New VODs appear, schedules refresh, manual syncs happen, APIs fail and recover. If every rerun can rewrite prior conclusions freely, the dataset gets worse over time instead of better. Once I treated the persistence layer as an **evidence hierarchy** instead of a dumb upsert loop, the system became much more reliable.

That was the real turning point.

---

## Title signals

Title analysis ended up being more useful than I expected, but only after I stopped being naive about it.

A raw name match is too loose. "Practicing w/ no mic" is not a collaboration even if someone named "no" exists in the friend list. The title signal became useful only once it was **gated by collab keywords**:

The system checks for a curated set of collab phrases ("collab," "ft.," "w/," "guest," "duo," "joined by," and others). A name match only becomes a signal if at least one keyword is also present, or if the match is an explicit @handle (which is always intentional enough to trust on its own).

@handles use a stricter pattern that enforces Twitch's username format. Display name matching requires a minimum length to avoid false hits on short common words. Self-references are filtered at every detection source so a streamer mentioning their own name does not create a phantom collab.

That moved the signal from "interesting text coincidence" to "**plausible collaboration evidence**."

## Overlap signals

Stream overlap is the broadest signal and the noisiest one.

Two people being live at the same time does not necessarily mean they collaborated. Even overlap plus game similarity is still not proof. But it is useful as supporting evidence because it can surface likely relationships that stronger sources miss.

The system requires a meaningful minimum overlap duration to count. Same-game overlap gets **high confidence** (both playing the same game for a significant stretch is a strong signal). Different-game overlap gets **medium confidence** (both live at the same time, but in unrelated contexts).

The trick is to treat it like a fallback, not like a verdict.

> Overlap can suggest. Stronger sources can confirm.

---

## Data model

Each detected collab is stored with a unique constraint on (friend, partner, date). That means one signal per partner per day per detection source. The evidence field stores the actual VOD title or event name for **auditability**.

Separately, confirmed events generate a higher-fidelity history record when an event is marked "completed." The two storage paths serve different purposes: signals are inferred and ranked, history is confirmed and curated.

The UI shows each collab partner with a badge: **"confirmed"** if any high-confidence signal exists, **"possible"** if only medium or weak signals support the relationship. Partners are sorted by signal count, then by recency.

## Why the system works

The system works because it is not trying to do too much with any one source.

The architecture is:

- **detect** from several imperfect places
- **rank** what each signal is worth
- **store** evidence in a way that protects higher-confidence history
- **keep reruns safe**

That is a much better fit than trying to invent one universal collab detector.

It also matches the actual product need. The planner does not need philosophical certainty. It needs a useful, stable memory of likely collaborations that can **improve over time** instead of thrashing.

## Constraints that shaped it

A lot of the design came from platform constraints:

- some useful metadata is only available in narrow contexts
- historical windows are finite (bounded per friend to avoid quadratic explosion)
- naming is inconsistent enough to require conservative matching
- broad inference is possible, but only if the persistence rules are disciplined

Those constraints are why the system leans so heavily on confidence ranking and rerun safety.

---

## What worked

- explicit title signals were stronger than I expected once properly gated
- weaker overlap signals were still useful when treated as supporting evidence rather than as conclusions
- protecting stronger history from weaker reruns made the whole system stable
- tying detection into existing data refresh paths kept the feature current without inventing a second orchestration layer

## What I would change next

- add better user feedback loops for dismissing false positives
- improve multilingual and non-English collaboration phrasing
- let repeated medium-confidence signals strengthen a relationship more gracefully over time
- keep tightening display-name matching without pushing it back into noisy territory

## Running it

This is part of [Twitch Friends Organizer](https://collab.deutschmark.online).
