# Inbound Email Sync for Job Search Pipelines

How I built a system that reads forwarded recruiter emails, classifies what they mean, matches them to the right job, and updates the pipeline with confidence gates and undo support. Most of the work is deterministic. The model is the fallback, not the default.

## The problem

Job-search tracking breaks down at the inbox layer. Responses arrive over weeks, often from different senders, with inconsistent subject lines and wildly different phrasing. Some are obvious rejections. Some are interview requests. Some are boilerplate acknowledgments that only become useful later.

I wanted the workflow to be simple: forward the email to a private alias and let the system decide whether it should update the pipeline, stage a review, or do nothing.

The challenge is not parsing email bodies. It is being accurate enough that a user is comfortable letting the system touch state on their behalf.

## Architecture

```text
Gmail / Outlook / iCloud
    |
    | forwarding rule
    v
track_{hash}@sync.yourpathos.app
    |
    | Resend webhook
    v
Supabase Edge Function
    |
    |- Svix signature verification
    |- user lookup by alias
    |- rate limiting
    |- deduplication
    |- deterministic domain matching
    |- deterministic keyword classification
    |- model fallback for ambiguous cases
    |- Guardian confidence recalibration
    |- role disambiguation
    |- guardrails
    '- status update + undo window
```

The first version lived in a single large Edge Function. That kept shipping simple, but if I were splitting it today I would break out the classifier, guardrails, and provider-specific parsing paths into separate modules.

## Webhook authentication

Inbound mail arrives through Resend, signed with Svix HMAC-SHA256:

```typescript
async function verifySvixSignature(rawBody: string, headers): Promise<boolean> {
  const svixId = headers.get("svix-id");
  const svixTimestamp = headers.get("svix-timestamp");
  const svixSignature = headers.get("svix-signature");

  if (Math.abs(Date.now() / 1000 - parseInt(svixTimestamp)) > 300) return false;

  const signedContent = `${svixId}.${svixTimestamp}.${rawBody}`;
  const keyBytes = base64Decode(secret.replace("whsec_", ""));
  const key = await crypto.subtle.importKey(
    "raw",
    keyBytes,
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"]
  );
  const expected = base64Encode(
    await crypto.subtle.sign("HMAC", key, encode(signedContent))
  );

  return svixSignature
    .split(" ")
    .some(sig => sig.split(",")[1] === expected);
}
```

Two details mattered here:

- Timestamp validation blocks replay attempts.
- Svix can send multiple signatures during key rotation, so verification has to accept any valid signature in the header rather than assuming a single current key path.

I also left a fallback `INBOUND_INGEST_KEY` path for providers that cannot emit Svix-compatible signatures.

## Deterministic company matching

The first pass tries to identify the company without any model call.

### Domain normalization

```typescript
const MULTI_PART_TLDS = new Set([
  "co.uk", "co.jp", "co.kr", "com.au", "com.br", "com.cn",
  "com.mx", "com.sg", "org.uk", "ac.uk", "gov.uk", "edu.au",
]);

function extractBrand(email: string): string {
  const domain = email.split("@")[1];
  const parts = domain.split(".");
  const lastTwo = parts.slice(-2).join(".");

  if (MULTI_PART_TLDS.has(lastTwo)) {
    return parts[parts.length - 3];
  }
  return parts[parts.length - 2];
}
```

Company names go through a parallel normalization path: lowercase, strip legal suffixes, remove punctuation, collapse spacing. That lets `"Google LLC"` and `recruiting@google.com` resolve to the same brand token.

### Match scoring

```text
exact domain/name match       -> 1.00
contains match                -> 0.95
reverse contains match        -> 0.95
generic mailbox domain        -> skipped
```

Generic mailbox domains such as Gmail, Yahoo, Outlook, iCloud, and ProtonMail are excluded from this path entirely. If the sender is `hr@gmail.com`, domain matching is not trustworthy enough to act on.

This deterministic pass handled most production traffic and kept inference cost low.

## Deterministic message classification

Once the company is known, the next question is what the email means.

I used regex- and phrase-based classification first:

- Rejection patterns
- Interview patterns
- Offer patterns
- Application-receipt patterns

Examples:

```text
rejection:
  "unfortunately"
  "not moving forward"
  "decided not to proceed"
  "another candidate"
  "position has been filled"

interview:
  "schedule an interview"
  "next step in the process"
  "phone screen"
  "technical assessment"
  "availability for"

offer:
  "job offer"
  "offer letter"
  "pleased to offer you"

receipt:
  "application received"
  "thank you for applying"
  "application confirmation"
```

The goal was not perfect language coverage. It was high precision on common cases. If the email was still ambiguous after deterministic checks, it moved to the fallback path instead of forcing a guess.

## Conflict detection

Some emails contain multiple valid signals:

```text
"We won't be moving forward with your application for role A.
However, we'd love to interview you for role B."
```

Those should not auto-apply anything. The system flags them as conflicts and stages them for manual review.

That decision matters because the expensive mistake is not "failed to automate." It is "automated the wrong state transition."

## Model fallback

Ambiguous cases use a fast model to extract:

- company
- role title
- message category
- raw confidence

This path is intentionally narrow. The model is not deciding everything on its own. It is filling in the cases where deterministic rules cannot get to a safe answer.

## Guardian confidence recalibration

Raw model confidence is not stable enough to trust by itself, so I recalibrated it with a second pass I called Guardian:

```typescript
let score = 0;
if (features.lexical_match) score += 0.30;
if (features.semantic_match) score += 0.30;
if (features.structure_match) score += 0.20;
if (features.logic_consistency) score += 0.20;

score = Math.max(0, score - uncertaintyFlags.length * 0.05);

final = score * 0.8 + Math.min(rawConfidence, 0.95) * 0.2;
```

The key idea was to ask the model for easier-to-verify signals instead of one big "trust me" number. Boolean or near-boolean checks such as "does this contain standard rejection language?" are easier to reason about and easier to audit later.

From there I mapped confidence into operational tiers:

```text
verified    >= 0.90
probable    0.70 - 0.89
uncertain   < 0.70
```

Auto-application had a stricter bar than "verified." It required very high confidence and zero guardrail violations.

## Role disambiguation

One company can map to several active applications, so the system has to decide which record the email belongs to.

The resolver used a staged approach:

1. Exact role-title match
2. Substring containment
3. Fuzzy similarity
4. Fallback to the most recent job at that company

Illustrative logic:

```typescript
function disambiguateByRole(jobs, extractedRole) {
  for (const job of jobs) {
    if (job.title.toLowerCase() === extractedRole.toLowerCase()) {
      return { job, score: 1.0 };
    }
  }

  for (const job of jobs) {
    if (
      job.title.toLowerCase().includes(extractedRole.toLowerCase()) ||
      extractedRole.toLowerCase().includes(job.title.toLowerCase())
    ) {
      return { job, score: 0.9 };
    }
  }

  for (const job of jobs) {
    const sim = similarity(job.title, extractedRole);
    if (sim > 0.6) return { job, score: sim };
  }

  return { job: jobs[0], score: 0.5 };
}
```

The fallback is intentionally weak. If the resolver reaches that branch without other supporting evidence, the result usually gets staged instead of applied.

## Receipt memory

Receipt emails are low-value in isolation but valuable later. If a rejection comes in from the same domain weeks after an application receipt, the receipt log can narrow the candidate jobs.

I used a lightweight scoring scheme:

```text
+3 same thread
+2 strong role-title similarity
+1 weak role-title similarity or missing role/thread data
```

If that memory reduced the candidate set to a single job, the system could often avoid a model call entirely.

## Hard guardrails

Some transitions are too risky to automate:

```text
rejection signal when current status is offer or accepted
offer and rejection signals in the same email
interview and offer signals in the same email
```

In those cases the system records the proposed interpretation but blocks the automatic update.

This was one of the more important product decisions. A system like this does not earn trust by being aggressive. It earns trust by being conservative in the right places.

## Undo support

Every auto-applied update stores the previous job status and an audit record:

```typescript
{
  matched_job_id: job.id,
  staged_status: "rejected",
  previous_job_status: "applied",
  status_auto_applied: true,
  status_auto_applied_at: new Date().toISOString(),
  staged_confirmed_at: null,
}
```

That made it possible to expose a short undo window in the UI. If the classifier got something wrong, rollback was explicit and cheap.

## Recovering the original response date

Forwarded emails are annoying because the provider timestamp reflects the forward, not the original message. To recover real response latency, I parsed provider-specific forwarding formats:

```typescript
function extractOriginalEmailDate(body: string): Date | null {
  // Gmail, Outlook, Apple Mail, generic "On Thu, Apr 3..." formats
  // Original date must precede the forward date
  // Reject obvious future dates and implausibly old timestamps
}
```

That let the system compute actual company response time relative to application date rather than forward time, which was much more useful for downstream analytics.

## Deduplication

I used a five-minute dedup window on `(user_id, sender, subject)` so the same forwarded message would not create duplicate state transitions.

The only major exception was forwarding-confirmation mail from providers. Those need to bypass normal dedup and classification logic because they are part of setup rather than job-state changes.

## Privacy

Raw email bodies are not persisted. The log stores only what is needed to audit decisions:

- sender
- subject
- category
- matched job
- confidence
- match method
- Guardian features
- guardrail flags
- timing metadata

That was enough to debug classifier behavior without turning the audit log into a second mailbox.

## Results

In practice, deterministic matching plus keyword classification handled the majority of emails. The fallback model mostly existed for personal-email recruiters, hiring platforms, and phrasing that did not fit the rule sets cleanly.

More importantly, the system failed in the right direction. When it was unsure, it staged. When it was very sure and the guardrails stayed clear, it applied. That balance mattered more than squeezing out a few more automated updates.

## What I would change next

- Split the large Edge Function into smaller classifier and provider modules
- Expand multilingual phrase support
- Add better recruiter-platform normalization for domains like Greenhouse and Lever
- Track calibration drift on the Guardian thresholds over time

## Running it

This is part of [P.A.T.H.O.S.](https://yourpathos.app). The ingestion function lives at `supabase/functions/ghost-listener/index.ts`, and the broader product context is in the [P.A.T.H.O.S. engineering note](https://github.com/thedeutschmark/engineering-notes/tree/main/how-i-built-pathos).
