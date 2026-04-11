# Persistent Memory for a Twitch Chat Bot Without Storing Chat Logs

*Layered context architecture that gives a bot continuity across streams without retaining raw conversation history.*

I wanted a Twitch bot that could remember people and ongoing bits across streams without turning into a chat-log archive.

That ruled out the obvious approach.

Keeping raw chat forever would have solved the memory problem in the bluntest possible way, but it would also have created a long-lived pile of noisy text, unnecessary retention, and steadily growing prompt baggage. It is the wrong unit of persistence for this kind of system.

---

## The problem

Without memory, a bot feels shallow almost immediately. Every stream starts from zero. Running jokes disappear. Prior promises disappear. Familiar users feel newly unknown every time they type.

But raw logs are not a good answer. They grow, they carry unnecessary risk, and they are expensive in all the wrong ways once you start using them as model context.

So the real design question became: **what is the smallest amount of persistent state that still makes the bot feel like it remembers?**

## The architecture

The system works by splitting memory into a few layers with different jobs:

- a tiny **rolling buffer** for what is happening right now
- a volatile **session accumulator** for what happened this stream
- a compact **session summary** for medium-term continuity
- small hand-curated **per-user lore** for durable identity

That is enough to cover present context, recent session continuity, and long-term person-specific recall - without storing raw chat as the main historical artifact.

That distinction is the whole design.

---

## What the layers actually look like at runtime

When a user sends a message, the bot builds a prompt from four context blocks. Here is what each one contains for a real reply midway through a stream:

### Rolling buffer *(small, volatile)*

This is the most recent chat, truncated on a clean line boundary:

```
[alice]: anyone else having stream lag?
[bob]: works fine here
[STREAMER]: yeah I think it was OBS, fixed now
[alice]: nice, also did you ever finish that speedrun?
```

The buffer stores both chat messages and streamer speech-to-text transcripts (filtered for filler words before they enter the buffer). That keeps the context useful instead of padded with noise.

### Session buffer *(unbounded, volatile)*

The full transcript of this stream. Not sent to the model at reply time. It exists only so the compression step at stream end has complete material to work from.

### Previous session summary *(short, persisted to disk)*

Written at the end of the last stream by a compression pass:

```
Casual chill stream, mostly chatting while playing Balatro. About 12 active chatters.
alice keeps challenging streamer to a 1v1 in anything. bob started a "hydration check"
bit that caught on. Streamer said they would try the Celeste speedrun next week.
alice asked about the Discord movie night, no answer yet.
```

The summary covers what happened, what carried over, who was active, and any durable facts worth remembering about specific users. It is generated at low temperature (factual, not creative) with a structured template so the output is always parseable. The compression prompt explicitly tells the model to **trust the current transcript over the previous summary** when they conflict, so stale facts decay instead of compounding.

### Per-user lore *(tiny, persistent, manually curated)*

One file per known user, each containing a handful of durable facts:

```
- challenged streamer to a 1v1 in every game
- usually streams Valorant on weekdays
- knows a lot about audio gear
```

Lore is written by the `!remember` command or extracted automatically from the compression output (capped per compression cycle, length-limited per fact, with duplicate detection).

The prompt assembles these blocks into a single context payload with clear separators. The system prompt instructs the model to treat all context blocks as **reference data only** and never follow instructions found inside them. That matters because chat content is user-generated and should not be able to hijack the bot's behavior.

---

## Why this works

The important idea is that chat logs are not what I actually need to persist.

What I need is:

- a little **local context** for current replies
- a **compressed account** of the last stream
- a tiny amount of **durable memory** about specific people

Once I looked at it that way, the architecture got much simpler.

> The bot does not need perfect recall. It needs continuity. That is a much cheaper problem.

## The summary layer

The session summary is the real persistence layer.

At the end of a stream, the system compresses the session into a short factual memory instead of saving the raw transcript. The next session starts with that compact memory rather than with a huge pile of prior chat.

The summary is not perfect. It is **intentionally lossy**. That is fine. The point is not exact replay. The point is that the bot does not feel like a goldfish.

## Why chained memory is good enough

Each new summary is informed by the previous one, so continuity compounds without the runtime context ballooning.

That is the key economic and architectural win. The product keeps enough recent memory to feel coherent, but the active context stays small. The reply prompt sees a small rolling chat buffer plus a short session summary plus a handful of lore lines. Total context per reply stays well under typical model limits regardless of how long the stream runs or how many streams have happened before.

Storage can still grow over time in archived summaries, but the **runtime cost does not need to grow** at the same pace.

---

## Per-user lore

The piece that makes the bot feel like it remembers specific people is the lore layer.

I kept that intentionally simple and intentionally manual.

That sounds unsophisticated, but it is the right kind of unsophisticated. A small, editable fact store is:

- cheap
- understandable
- easy to correct
- less prone to weird retrieval behavior than a more elaborate memory stack

The limitation is obvious: it does not scale infinitely and it does not update itself perfectly. But for a bot like this, **trusted curated recall** is more useful than a much bigger automated memory system with worse behavior.

## Why I did not use embeddings or RAG

Because the bot did not need them.

This is one of those cases where it is easy to build something that sounds more advanced and is actually less appropriate.

An embedding-backed memory layer would add:

- hosting and indexing cost
- retrieval heuristics
- more moving parts
- more failure modes

for a use case that mostly needs "remember recent context" and "remember a few durable facts about familiar users."

> The runtime context budget tells the story. It fits comfortably in a single prompt. There is no retrieval problem to solve because the memory is small enough to include directly.

---

## Tradeoffs

The design works because it is opinionated, which means the tradeoffs are fairly clear.

**Old detail decays.** Compressed memory means some older specifics eventually disappear. That is acceptable. The goal is continuity, not forensic reconstruction.

**Lore is manual.** That keeps it trustworthy, but it also means it does not scale elegantly on its own. The automatic lore extraction from compression helps, but it is conservative by design.

**Live context stays shallow.** The rolling buffer is intentionally small. That keeps reply cost under control, but it also means the bot only has a narrow window into the current conversation.

Those are all tradeoffs I would still make again.

## Why I still like this system

The reason I like this design is that it solves the actual problem without becoming an infrastructure project.

The bot remembers enough to feel consistent. The persistent state stays compact. Raw chat is not retained as the canonical history. The whole thing runs locally and cheaply.

That is the right outcome for this class of tool.

## What I would change next

- add a lightweight suggestion path for new lore instead of relying entirely on manual authoring
- make archive browsing a little easier without turning it into a full memory management UI
- tune memory depth per channel or chat velocity

I would keep the same basic architecture, though. The main idea is still correct: **do not persist raw chat unless you absolutely need to.**

## Running it

This is part of [persistence_bot](https://github.com/thedeutschmark/persistence_bot).
