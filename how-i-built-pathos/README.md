# How I Built P.A.T.H.O.S.

P.A.T.H.O.S. started from a simple frustration: most job-search tools either give you generic AI polish or they give you a tracker that falls apart the moment the process gets messy.

I wanted something stricter than that.

The score needed to be explainable. The rewriting needed to stay inside the truth. The system needed to account for ghosting, weak listings, recruiter latency, and the fact that most users are navigating a high-friction process rather than a neat funnel.

That became P.A.T.H.O.S., short for Personal Automated Tracking & Hiring Optimization System.

## The problem I was trying to solve

Modern hiring stacks are fragmented. Applicants are writing for recruiters, parsers, ranking heuristics, and increasingly AI-assisted review systems at the same time.

Most products respond by adding more opacity:

- a black-box ATS score
- resume rewriting that sounds polished but generic
- pipeline tracking that assumes the user will manually maintain every state change

I wanted the opposite:

1. deterministic scoring wherever deterministic scoring makes sense
2. an AI layer that edits rather than fabricates
3. support systems for the parts of the process that usually waste the most time

## The design rules

Three rules shaped almost every major decision.

### 1. Deterministic first

If a decision can be made with visible rules, that is usually better than hiding it behind a model.

That does not mean the rules are perfect. It means they can be inspected, tuned, and challenged.

### 2. AI as a constrained editor

I wanted the model helping with translation, prioritization, and wording, not inventing substance.

That means the useful AI work is:

- reframing existing experience in the language of the role
- tightening bullets
- changing emphasis
- making the document read more clearly for both humans and systems

The destructive AI work is:

- inventing tools
- inflating titles
- fabricating outcomes
- flattening the user's voice into generic optimizer prose

The whole product is built around trying to stay on the first side of that line.

### 3. The product has to reflect the real process

A job search is not just document editing. It is also inbox noise, weak listings, silent rejection, false hope, and a lot of uncertainty about what actually happened after the application left your hands.

If the product only optimizes the resume, it solves too small a part of the problem.

## Deterministic ATS scoring

The core scoring layer is deliberately formula-driven.

I wanted the product to tell the user something more useful than "your ATS score is 82." The number by itself is not interesting. What matters is why the score moved and what kinds of changes are most likely to improve it.

So the scoring model focuses on a few things that are explainable and usually worth caring about:

- whether the JD's important terms are present
- whether they appear in meaningful context rather than only in a list
- whether the document is easy for a parser to read
- whether the writing itself is structurally strong enough to survive first-pass review

This is not a claim to emulate every real ATS exactly. It is a practical scoring system for resume quality under ATS-like constraints.

That distinction matters.

## AI rewriting without fabrication

Once the deterministic layer is in place, the AI layer becomes more useful because it has a narrower job.

I do not need the model to invent qualifications. I need it to improve how existing qualifications are presented.

That means the optimizer is designed around two constant tensions:

- improve alignment without making false claims
- improve language without erasing the user's own voice

Those tensions are the whole product, really.

If the model gets too conservative, the output is timid and unhelpful. If it gets too aggressive, the output becomes false, generic, or both.

So a lot of the work in P.A.T.H.O.S. is really about guardrails:

- preserving role identity
- preserving factual claims
- filtering cliché-heavy AI language
- keeping generated output close enough to the author's original voice to still feel authored

I think that is a much better frame than "resume generation." It is closer to constrained transformation.

## Brand Voice and Voice & Style

One thing I did not want was a system that optimized everyone into the same tone.

The product ended up with a `Voice & Style` layer because voice preservation is not a cosmetic feature. It is part of whether the result still feels credible.

The useful version of this is not "let the user write a massive style prompt." It is extracting a compact style profile from real writing samples and then using that as a boundary condition on later generation.

The goal is simple: if a user is naturally terse, technical, and evidence-heavy, the optimized output should not suddenly sound like a motivational LinkedIn post.

## Graceful degradation

I cared a lot about the product still being useful when the model path is unavailable or not the right tool.

That led to a layered approach:

- deterministic analysis and guidance should still stand on their own
- model-assisted rewriting should improve the system, not define the entire product
- failure in one layer should not make the whole experience collapse

This ended up mattering for user trust as much as for architecture. A product feels much more reliable when it can degrade into a simpler truthful mode instead of simply failing.

## Glass Box transparency

One of the ideas that became more important over time was what the product calls `Glass Box Transparency`.

The short version is that I did not want the system making consequential changes in silence.

The longer version is that there are really three different Glass Boxes in the app:

- the persona layer
- the optimizer and its multi-stage process
- the inbound intelligence layer

That distinction matters because each surface is answering a different trust question.

I go into that in more detail in [Glass Box transparency](../glass-box-transparency/), but the basic point is simple: transparency should show the user enough of the basis for action that the system feels inspectable without dumping raw internals or pretending to reveal some magical private chain of thought.

That ended up being especially important in the optimizer, where the system is moving through multiple stages of analysis, inference, generation, and review.

## The ATS visibility gap

One of the framing choices on the landing page is that the product does not start from reassurance. It starts from the fact that hiring systems are structurally messy.

That matters because the product only makes sense if you accept the environment it is built for:

- ATS software is now normal infrastructure, not an edge case
- screening systems often exclude qualified people for procedural reasons
- application volume can rise faster than actual hiring
- ghosting and recruiter overload distort the process even when the market is not uniformly collapsing

That is why P.A.T.H.O.S. is not just a resume optimizer. It is trying to help the user operate inside a process that is noisy, crowded, and often indifferent to nuance.

The public sources I use in the paper are there to ground that claim, not to inflate it.

## Ghost-job detection

One of the obvious ways job seekers waste time is by pouring effort into listings that are weak, stale, or structurally suspicious.

The ghost-job layer exists to score risk, not to declare that a listing is fake with total certainty.

That distinction is important.

The product is looking for patterns that correlate with low-signal or low-trust listings:

- vague evergreen language
- weak specificity
- contradictory expectations
- signs that the posting is more of a funnel than an actively staffed role

I wanted that feature to feel like a warning system, not a conspiracy engine.

## Job Status Sync

The other big support layer is the inbound email system, surfaced in the product as `Job Status Sync`.

I added this because manual pipeline tracking is one of the first things users abandon. The more applications they submit, the less likely they are to keep every state updated by hand.

So the system watches for inbound recruiter signals, tries to classify them, matches them back to the right application, and gives the user a correction path when the automation is wrong.

That last part matters more than the automation itself. Auto-classification without a visible correction path is just another black box.

## The companion layer

I also did not want the product to act like a generic assistant pasted onto a dashboard.

The companion layer exists because tone affects whether the product feels steady, irritating, useful, or fake. But I wanted that layer to be controlled, not improvisational.

So the persona is bounded. It can be direct. It can be sharp. It can respond differently by mode. But it is still there to support the rest of the system, not to become the system.

That is why I think of it as a companion layer rather than as a chatbot feature.

## Learning from outcomes

Once enough application outcomes exist, the product can start looking for signals that are specific to that user rather than generic to the market.

That is where the system becomes more than a one-shot optimizer.

The interesting part is not "machine learning" in the abstract. The interesting part is whether the app can notice that some patterns appear to correlate with better outcomes for this user and then use that information carefully, without overreacting to tiny samples or turning noise into policy.

That is the line I try to hold there: useful adaptation without false certainty.

## Task Board and Daily Missions

Once the product had scoring, pipeline state, inbox signals, and outcome learning, it made sense to turn some of that into action.

That is what the `Task Board` and `Daily Missions` layer is really doing. It is not just listing work. It is trying to convert a noisy background system into a smaller set of actions that are worth doing now.

That ended up being one of the most practical parts of the app because it closes the loop between analysis and behavior.

## What worked

The pieces that held up best were the ones built around explicit constraints:

- deterministic scoring instead of opaque scoring
- AI as a constrained editor instead of a free author
- Glass Box transparency instead of post-hoc handwaving
- style preservation instead of generic personalization
- inbox automation with review and correction paths
- a task layer built on live state rather than static reminders

Those choices made the product easier to explain and, more importantly, easier to trust.

## What I would change next

- make the learning layer more recency-aware
- improve false-positive handling in ghost-job warnings
- keep tightening the boundary between strong rewriting and overreach
- continue breaking larger subsystems into clearer product modules

## External references

These are representative public references that informed the framing of the product and the market conditions it is built around.

### ATS and hiring-friction references

- [Jobscan, 2025 Applicant Tracking System (ATS) Usage Report](https://www.jobscan.co/blog/fortune-500-use-applicant-tracking-systems/)
- [Harvard Business School + Accenture, Hidden Workers: Untapped Talent](https://www.hbs.edu/managing-the-future-of-work/research/Pages/hidden-workers-untapped-talent.aspx)
- [U.S. Bureau of Labor Statistics, JOLTS](https://www.bls.gov/jlt/)

### Additional market context used in the app's benchmark layer

- [Greenhouse, 2024 State of Job Hunting report](https://www.greenhouse.com/blog/greenhouse-2024-state-of-job-hunting-report)
- [LinkedIn Economic Graph, January 2024 Workforce Report](https://economicgraph.linkedin.com/resources/linkedin-workforce-report-january-2024)
- [iCIMS, January 2025 Workforce Report](https://www.icims.com/company/newsroom/januaryinsights2025/)

## Running it

P.A.T.H.O.S. is live at [yourpathos.app](https://yourpathos.app).
