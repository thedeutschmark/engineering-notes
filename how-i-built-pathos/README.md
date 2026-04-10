# How I Built P.A.T.H.O.S.

P.A.T.H.O.S. started from a simple observation: job-search tooling is full of opaque scoring, generic AI rewriting, and pipeline trackers that stop being useful the moment the inbox gets messy. I wanted something more rigorous. The scoring should be explainable, the rewriting should stay truthful, and the product should reflect what the market actually feels like instead of pretending the process is cleaner than it is.

## The problem I was trying to solve

Modern hiring stacks are fragmented. Applicants write for recruiters, parsers, scoring rules, and AI filters at the same time. Most tools in this space react by adding even more opacity:

- a black-box "ATS score" with no explanation
- AI rewriting that sounds generic after one pass
- pipeline tracking that depends on the user manually updating every response

I wanted the opposite:

1. A scoring system I could explain line by line
2. An AI layer constrained hard enough that it could improve wording without inventing substance
3. Support systems for the two biggest sources of wasted effort: ghost jobs and missed inbox signals

That became P.A.T.H.O.S., short for Personal Automated Tracking & Hiring Optimization System.

## Design principles

Three principles shaped the product from the beginning:

### 1. Deterministic first

If a decision can be made with transparent rules, it should be. That makes debugging easier, auditing possible, and product behavior more stable.

### 2. AI as a constrained editor, not an author

The model can translate, tighten, reorder, and suggest. It does not get to invent work history, credentials, or metrics.

### 3. The product has to acknowledge the real process

A job search is not just resume editing. It is also ghost listings, slow response cycles, buried rejections, and a lot of uncertainty. The system needed to account for that, not just optimize one document in isolation.

## Deterministic ATS scoring

The core score is formula-based:

```text
Score = (KeywordMatch x 0.60) + (SemanticContext x 0.20)
      + (((Readability + Parseability) / 2) x 0.20)

Hard cap: 98
```

The cap is deliberate. A perfect score suggests a level of certainty that is not realistic for resume screening.

### Keyword extraction and weighting

Keywords are pulled from the job description and bucketed by salience:

| Category | Signal | Weight |
|---|---|---|
| MUST-HAVE | repeated frequently or emphasized in requirements | 5 |
| SHOULD-HAVE | repeated moderately or emphasized in preferred qualifications | 3 |
| NICE-TO-HAVE | present but lower emphasis | 1 |

I then weight them by where they appear in the resume:

```text
summary section         -> 1.5x
first bullet per role   -> 1.2x
skills section          -> 1.3x
everything else         -> 1.0x
```

This is not a claim that every ATS behaves identically. Different systems parse and rank differently. The point is that some resume locations are consistently more important for both parsing and human review, so the scoring model reflects that.

### Semantic context

Presence alone is not enough. A term in a skills list is weaker than the same term used in evidence-bearing context.

That is what the semantic component measures. `"Python"` under skills is useful. `"Built a Python pipeline processing 2M records/day"` is stronger because it ties the term to an action and an outcome.

### Readability

Each bullet gets scored for features that usually correlate with stronger resume writing:

```text
action verb present         +25
quantified result           +35
ideal length range          +20
starts with action verb     +20
```

This is essentially structured writing guidance disguised as a score. Bullets with a clear action and a clear result tend to read better whether a human or parser sees them first.

### Parseability

Parseability starts from a baseline and adds or subtracts based on structure:

- expected sections present
- consistent bullets
- plain text that survives parser extraction
- penalties for formatting choices that commonly break parsing or degrade extraction quality

This is the least glamorous part of the system and one of the most useful. A technically strong resume can still lose information if the parser output is messy.

## AI rewriting without fabrication

The deterministic score tells the user what changed and why. The AI layer exists to help with the wording, not to replace judgment.

### Truth constraints

The prompt allows:

- translating existing experience into job-description language
- tightening bullets
- reordering emphasis
- inferring a small number of directly supported skills

The prompt forbids:

- adding unsupported tools or frameworks
- changing dates, titles, employers, or education
- inventing outcomes or metrics

That boundary matters. Resume optimization becomes useless the moment the model starts making the candidate sound more qualified than the record supports.

### Voice preservation

Before rewriting, the system extracts a lightweight voice profile from the original resume:

```javascript
{
  avgSentenceLength: 14,
  avgBulletLength: 16,
  usesMetrics: true,
  actionVerbs: ["built", "designed", "led"],
  technicalDensity: "high",
  toneProfile: "confident-technical",
  passiveVoiceRate: 0.08,
  specificityScore: 0.65,
  flaggedJargon: ["synergy"],
}
```

That profile gets fed back into the rewrite prompt so the output stays recognizably close to the original author.

I did not want "optimization" to mean "flatten everything into generic AI prose." If the original resume is terse and technical, the rewritten version should still feel terse and technical.

### Brand Voice and Voice & Style

That same system extends into the app's `Voice & Style` surface. Users can provide raw writing samples, and the system extracts a compact "Brand Voice Directive" that gets fed back into generation.

The point is not to let users write a giant style prompt. It is to derive something narrower and more usable:

- sentence length and cadence
- tone and formality
- jargon tolerance
- directness vs narrative style

In product terms, this shows up as Brand Voice. In implementation terms, it is a constrained style layer sitting on top of the same truth-preserving optimizer.

### Verb control and cliché filtering

One failure mode in this category is overcorrection. A weak bullet becomes a cartoonishly aggressive bullet because the system keeps escalating the verb.

To avoid that, the rewrite pass tracks weak and strong verb usage, but it also blocks cliché-heavy language and overblown phrasing. Terms like `synergy`, `leverage`, `paradigm`, `move the needle`, and similar filler get flagged explicitly.

The goal is not "more intense." It is "clearer, more specific, still believable."

### Bullet validation

Each bullet is checked against a simplified STAR-style structure:

```javascript
{
  score: 3,
  maxScore: 4,
  components: {
    hasSituation: true,
    hasTask: false,
    hasAction: true,
    hasResult: true
  },
  missing: ["task"],
  isValid: true
}
```

I do not require every bullet to contain every STAR element. That would force unnatural writing. What I care about most is whether the bullet clearly says what happened and what changed.

## Graceful degradation

The product is designed so the useful part still works even if the model path is unavailable:

```text
Tier 1: full AI      -> complete optimization
Tier 2: partial AI   -> simplified rewrite path
Tier 3: local only   -> deterministic scoring and guidance
```

That meant a user could still get meaningful feedback from local extraction, keyword matching, and structural analysis even if the external model call failed.

It also affected billing. Credits are deducted after a successful response, not before. If an upstream call fails, the user should not pay for the failure mode.

## Glass Box transparency and Advanced Mode

One of the product ideas that kept surviving iteration was what the landing page now calls `Glass Box Transparency`.

Most resume tools hide their reasoning. I wanted the opposite. When the optimizer runs, the app exposes a live process view of what it is doing: parsing the job description, extracting weighted keywords, identifying gaps, generating rewrites, and checking the result against its guardrails.

That matters for two reasons:

1. It makes the system easier to trust.
2. It makes debugging much easier when the output is wrong.

This is also where `Advanced Mode` fits. Standard mode is optimized for speed. Advanced Mode exposes more of the review surface, especially around skill verification and generation control, so users can intervene earlier instead of only inspecting the final output.

I did not want "transparent AI" to mean a vague explanation after the fact. I wanted a working surface that shows the system's reasoning while the job is in progress.

## The ATS visibility gap

One part of the landing page is built around a blunt claim: applicant-tracking systems are optimized to reduce recruiter workload, not to identify the best candidate in any deep or holistic sense.

That section exists because I did not want the product to start from fake reassurance. If the user is struggling, the system should name the actual terrain first.

The three reference points behind that section are doing different jobs.

### 1. ATS prevalence

The first point is simple: ATS usage is not edge-case infrastructure anymore. It is the default operating environment for corporate hiring.

The landing page references [Jobscan's ATS usage report](https://www.jobscan.co/blog/fortune-500-use-applicant-tracking-systems/) for that reason. I am not using it to claim every employer screens the same way. I am using it to establish the baseline fact that a large share of hiring funnels pass through ATS software before a recruiter reviews anything meaningful.

That matters because it changes the product requirement. If the resume is not parseable, structurally legible, and keyword-aligned enough to survive early filtering, better writing further down the document does not help. That is why P.A.T.H.O.S. starts with deterministic ATS analysis instead of jumping straight to generation.

### 2. Hidden-worker exclusion

The second point comes from [Harvard Business School and Accenture's Hidden Workers research](https://www.hbs.edu/managing-the-future-of-work/research/Pages/hidden-workers-untapped-talent.aspx).

That reference matters less as a resume statistic and more as a systems argument. It supports the idea that hiring pipelines frequently exclude qualified candidates for structural reasons: overly rigid filters, proxy requirements, fragmented screening logic, and process choices that optimize throughput over judgment.

That is the philosophical center of the product. The goal is not to game hiring with fabricated credentials. The goal is to reduce avoidable exclusion by translating real experience into forms the pipeline is more likely to recognize.

### 3. Market pressure and volume

The third point is [BLS JOLTS](https://www.bls.gov/jlt/), which gives macro labor-market context around openings, hires, and churn.

On its own, JOLTS does not tell you how one ATS ranks one resume. What it does provide is grounding for the broader claim that the market is noisy, high-volume, and uneven. It helps anchor the product against real labor-market movement rather than turning the whole story into anecdote.

In practice, that macro context combines with what users see in the app itself:

- longer response cycles
- more silent applications
- more competition per posting
- more need to prioritize where effort actually goes

That is why the product is not just a resume editor. It also tracks the pipeline, flags ghosting risk, intercepts inbound signals, and tries to narrow attention onto the applications most worth pursuing.

### Why that landing-page section matters

I wanted that section to do three things before the user ever signs up:

1. explain that their difficulty may be structural, not purely personal
2. justify why the product begins with ATS visibility and pipeline intelligence
3. make it clear that the system is built for a hostile market, not an idealized one

Without that framing, the rest of the product can sound like any other optimizer. With it, the architecture makes more sense: deterministic scoring, truth-constrained rewriting, Glass Box transparency, ghost detection, and Job Status Sync all become responses to the same underlying problem.

### What the market reports are actually saying

The broader market framing is also grounded in three recurring external sources that the app uses for its benchmark layer.

#### Greenhouse: candidate experience is breaking down

The [Greenhouse 2024 State of Job Hunting report](https://www.greenhouse.com/blog/greenhouse-2024-state-of-job-hunting-report) is useful because it describes the hiring market from the candidate side rather than from the employer-system side alone.

A few points from that report line up directly with why P.A.T.H.O.S. exists:

- the report says `79%` of U.S. workers surveyed felt heightened anxiety in the current job market
- `61%` said they had been ghosted after a job interview
- Greenhouse also reported recruiter workload rising `26%` in the prior quarter
- `38%` of job seekers reported mass-applying to roles
- Greenhouse says `18-22%` of jobs posted on its platform in a given quarter were classified as ghost jobs

That combination matters. It is not just that candidates are stressed. It is that candidate experience degrades when recruiter load increases and application volume spikes. That is exactly the environment where ghosting rises, signal quality drops, and a tool that only rewrites resumes is not enough.

#### LinkedIn: the market can improve month to month and still stay harder year over year

The [LinkedIn Workforce Report, January 2024](https://economicgraph.linkedin.com/resources/linkedin-workforce-report-january-2024) is useful because it gives a macro hiring snapshot rather than just anecdotal job-search sentiment.

The January 5, 2024 report notes that U.S. hiring increased `5.5%` in December 2023 compared with November 2023, but was still down `9.9%` compared with December 2022. That is the kind of mixed market signal I wanted the product to respect.

The point is not "the market is always getting worse." The point is that localized improvement does not automatically make the funnel easier for individual candidates. A market can show short-term hiring gains while still being crowded, inconsistent, and difficult to navigate. That is part of why the app tracks pipeline state and timing instead of treating every application as an isolated document exercise.

#### iCIMS: competition is rising even when hiring is not collapsing

The [iCIMS January 2025 Workforce Report](https://www.icims.com/company/newsroom/januaryinsights2025/) adds another piece of the picture: competition per opening.

Their January 2025 report says:

- applications were up `13%` from the end of 2023
- applicants per opening rose `11%`
- job openings were up `3%`
- hires were down `1%`

That pattern is important. You can have stable or slightly improving openings and still make the market meaningfully harder for candidates if application volume rises faster than hiring does.

That is exactly the kind of environment P.A.T.H.O.S. is designed for. The product assumes the user is operating inside a crowded funnel where better targeting, clearer ATS visibility, ghost detection, and faster response handling all matter because brute-force applying gets less efficient as applicants-per-opening rises.

## Ghost job detection

A lot of wasted effort happens before the application is even submitted. Some listings are stale, some are soft evergreen roles, and some are written so vaguely that they should be treated cautiously.

The ghost-job detector scores listings from 0 to 100 using deterministic patterns:

### Language signals

- future-opportunity phrasing
- "always hiring" or evergreen language
- language suggesting the role is paused, internal, or not actively hiring

### Structural signals

- too few concrete responsibilities
- unusually generic copy
- high buzzword density with little operational detail

### Contradiction checks

- entry-level framing with implausibly senior expectations
- compensation and seniority mismatches

The output is not "this job is fake." It is "this listing carries a higher risk profile than average." That framing is more honest and more actionable.

## Ghost Listener: inbox state without manual tracking

The second large system inside P.A.T.H.O.S. is the [inbound email sync pipeline](https://github.com/thedeutschmark/engineering-notes/tree/main/email-sync). In the product, this is surfaced as `Job Status Sync`.

Users forward recruiter mail to a private alias. The system then:

- verifies webhook authenticity
- matches sender and company
- classifies the message
- applies or stages a job-status update
- stores an audit trail without keeping the raw body

Most of that path is deterministic. The fallback model exists for ambiguity, not as the default strategy.

### Response latency intelligence

The email system also recovers original message timestamps from forwarded-message headers. That makes it possible to measure actual company response latency rather than the time the user forwarded the mail.

Those timings feed back into company intelligence:

- extremely fast rejections often indicate automated filtering
- multi-week silence is useful ghosting signal
- response-time distributions help users calibrate expectations by company

The dashboard side of this system matters too. The app does not just update status quietly in the background. It exposes the event stream, confidence level, match method, and override path so the user can see what was intercepted and correct it when needed.

## The companion layer

Most job-search software either ignores the emotional side of the process or responds with generic encouragement. I did not want either extreme.

P.A.T.H.O.S. has three response modes:

- `Lite`: minimal and functional
- `Professional`: direct, strategic, and less chatty
- `Untethered`: sharper and more opinionated, but still bounded

The implementation is not "let the model improvise personality." Most responses are template-driven with variable substitution and only a light synthetic pass on top. That keeps tone more stable and makes novelty easier to manage.

### Tone control

I used a hostility-budget system so the voice does not jump from mild to abrasive without context:

```text
onboarding   -> low ceiling
ramping      -> moderate ceiling
full mode    -> full range
```

There is also a safety valve that pulls the tone down when the system sees distress signals such as repeated rejections or sudden pipeline collapse.

The point was not to build a mascot. It was to build a voice that could be blunt without becoming chaotic.

## Learning from outcomes

Once the system has enough application outcomes, it starts looking for user-specific patterns:

- skills that correlate with better interview rates
- titles or industries that perform unusually well
- writing patterns associated with stronger results

Those insights are gated by a Wilson score lower bound so the system does not overreact to tiny samples:

```text
lb = (p + z^2/2n - z*sqrt(p(1-p)/n + z^2/4n^2)) / (1 + z^2/n) x 100
```

That matters because "50% interview rate" means very different things at `n = 2` and `n = 40`.

When the confidence gate passes, the optimizer can prioritize genuinely useful personal signals instead of treating every job description as a fresh start.

## Task Board and Daily Missions

Once the pipeline, ghost detection, and outcome data existed, it made sense to stop treating the app as a passive tracker.

That is where the `Task Board` and `Daily Missions` layer came from. Instead of showing users a static list of applications, the system generates next actions from live state:

- follow up with applications approaching a ghost threshold
- apply now when the user's response patterns suggest a better send window
- prioritize cleanup or profile work when the pipeline is thin
- surface setup milestones for incomplete account state

Under the hood this is a scoring problem, not a checklist. Missions are ranked by urgency, impact, freshness, and ML relevance. The result is that the app can turn a messy pipeline into a smaller set of actions worth doing today.

This ended up being one of the more useful product layers because it translated all the background analysis into an actual operating cadence.

## Prompt-injection defense

Job descriptions are user-supplied input embedded in prompts. That makes them a natural prompt-injection vector.

To reduce that risk, the system strips common injection patterns before prompt construction:

- direct instruction overrides
- system-prompt extraction attempts
- role hijacking phrases
- jailbreak patterns
- encoding or decode tricks

The sanitized content is then embedded inside tagged sections treated as data, not instructions.

This is not a perfect security boundary, but it is an important hardening step when the prompt includes untrusted text.

## Data handling and sovereignty

The public product language leans on privacy and data sovereignty, so the architecture had to support that in a defensible way.

The practical rules are straightforward:

- keep raw sensitive inputs only where operationally necessary
- encrypt long-lived account data at rest
- transmit over TLS
- separate personal learning from shared aggregate intelligence
- make export possible instead of trapping user history inside the product

Not every feature is local-first, and I would not overclaim that. But the system is designed to minimize unnecessary retention and to keep the shared intelligence layer de-identified rather than user-exposed.

## What worked

The pieces that held up best were the ones built around explicit constraints:

- deterministic scoring instead of opaque scoring
- AI rewriting with hard truth boundaries
- Glass Box transparency instead of post-hoc handwaving
- Brand Voice as a constrained style layer instead of generic personalization
- inbox automation with guardrails and undo support
- Task Board / Daily Missions as an action layer on top of the pipeline
- company intelligence grounded in response and listing behavior rather than one-off anecdotes

Those choices made the system easier to explain and easier to trust.

## What I would change next

- add stronger feedback loops on ghost-job warnings and false positives
- decay learning signals by recency instead of treating old wins and recent wins equally
- expand multilingual keyword and pattern support
- continue breaking the larger subsystems into smaller modules with clearer ownership boundaries

## External references

These are representative public references that informed both the framing of the product and parts of the landing-page research layer. They are not the only inputs to the system, but they ground the paper in the same outside sources the app already points to.

### ATS and hiring-friction references

- [Jobscan, 2025 Applicant Tracking System (ATS) Usage Report](https://www.jobscan.co/blog/fortune-500-use-applicant-tracking-systems/)
  Used for ATS prevalence context, including Fortune 500 ATS adoption and common platform distribution.
- [Harvard Business School + Accenture, Hidden Workers: Untapped Talent](https://www.hbs.edu/managing-the-future-of-work/research/Pages/hidden-workers-untapped-talent.aspx)
  Useful background on how hiring processes and screening systems systematically exclude qualified candidates.
- [U.S. Bureau of Labor Statistics, JOLTS](https://www.bls.gov/jlt/)
  Baseline labor-market reference for openings, hires, quits, and broader market movement.

### Additional market context used in the app's benchmark layer

- [Greenhouse, 2024 State of Job Hunting report](https://www.greenhouse.com/blog/greenhouse-2024-state-of-job-hunting-report)
  Used for candidate-experience context around ghosting, recruiter overload, application flooding, and ghost jobs.
- [LinkedIn Economic Graph, January 2024 Workforce Report](https://economicgraph.linkedin.com/resources/linkedin-workforce-report-january-2024)
  Used for hiring-rate and labor-market context, especially month-over-month improvement versus year-over-year softness.
- [iCIMS, January 2025 Workforce Report](https://www.icims.com/company/newsroom/januaryinsights2025/)
  Used for applicants-per-opening and application-volume pressure in the current hiring market.

Where the product makes stronger operational claims than these public sources support on their own, those claims come from the app's own deterministic logic, telemetry, and user outcome data rather than from third-party research alone.

## Running it

P.A.T.H.O.S. is live at [yourpathos.app](https://yourpathos.app). The deterministic scoring engine is in `src/utils/atsScoring.js`, prompt logic in `src/utils/aiPrompts.js`, ghost-job detection in `src/utils/ghostJobDetection.js`, and email sync in `supabase/functions/ghost-listener/index.ts`.
