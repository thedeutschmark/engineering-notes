# Inbound Email Sync for Job Search Pipelines

How I built a system that automatically detects recruiter responses — rejections, interview invitations, offers — from forwarded emails and updates a job pipeline without the user touching anything. No AI for 80% of cases.

## The problem

You apply to 50 jobs. Responses trickle in over weeks. Some are rejections buried in promotions tabs. Some are interview requests you almost miss. Some companies never respond at all. Manually tracking which jobs are alive, dead, or ghosting you is a full-time job on top of your actual job search.

I wanted something simple: forward your recruiter emails to an alias, and the system figures out what the email means, matches it to the right job in your pipeline, and updates the status automatically. No manual sorting, no daily inbox archaeology.

The hard part isn't parsing emails. It's doing it accurately enough that you trust it to change your pipeline state without asking first.

## Architecture

```
Gmail/Outlook/iCloud
    │
    │  email forwarding rule
    ▼
track_{hash}@sync.yourpathos.app
    │
    │  Resend webhook
    ▼
Supabase Edge Function (Deno)
    │
    ├─ Svix HMAC-SHA256 verification
    ├─ User lookup by alias
    ├─ Rate limiting (daily + hourly)
    ├─ Deduplication (5-min window)
    ├─ Deterministic domain matching
    ├─ Deterministic keyword classification
    ├─ AI fallback (Gemini 2.5 Flash, ~20% of cases)
    ├─ Guardian confidence recalibration
    ├─ Role-title disambiguation
    ├─ Hard guardrails
    └─ Status update with undo support
    │
    ▼
jobs table (status updated)
inbound_email_logs (audit trail)
```

The entire pipeline is a single 1,600-line Edge Function. Every decision is logged. Raw email bodies are never persisted.

## Webhook authentication

Resend delivers inbound emails via webhook. The payload is signed with Svix HMAC-SHA256:

```typescript
async function verifySvixSignature(rawBody: string, headers): Promise<boolean> {
  const svixId = headers.get("svix-id");
  const svixTimestamp = headers.get("svix-timestamp");
  const svixSignature = headers.get("svix-signature");

  // Replay protection: reject if timestamp > 5 minutes old
  if (Math.abs(Date.now() / 1000 - parseInt(svixTimestamp)) > 300) return false;

  // Signed content = id.timestamp.body
  const signedContent = `${svixId}.${svixTimestamp}.${rawBody}`;

  // Decode secret (format: whsec_<base64>)
  const keyBytes = base64Decode(secret.replace("whsec_", ""));
  const key = await crypto.subtle.importKey("raw", keyBytes, { name: "HMAC", hash: "SHA-256" }, false, ["sign"]);
  const expected = base64Encode(await crypto.subtle.sign("HMAC", key, encode(signedContent)));

  // Svix sends multiple signatures — match against any
  return svixSignature.split(" ").some(sig => sig.split(",")[1] === expected);
}
```

The multi-signature support matters. Svix rotates signing keys and sends both old and new signatures during rotation windows. If you only check the first one, you'll reject valid webhooks during key rotation.

A fallback `INBOUND_INGEST_KEY` header exists for custom email providers that don't support Svix.

## Deterministic domain matching

The first classification pass uses zero AI. Extract the sender's email domain, normalize it, and match against the user's job pipeline.

### Domain normalization

```typescript
// Multi-part TLD handling
const MULTI_PART_TLDS = new Set([
  "co.uk", "co.jp", "co.kr", "com.au", "com.br", "com.cn",
  "com.mx", "com.sg", "org.uk", "ac.uk", "gov.uk", "edu.au",
  // ... 55 total
]);

function extractBrand(email: string): string {
  const domain = email.split("@")[1];
  const parts = domain.split(".");
  const lastTwo = parts.slice(-2).join(".");

  // amazon.co.uk → ["amazon", "co", "uk"] → brand = "amazon"
  // google.com   → ["google", "com"]       → brand = "google"
  if (MULTI_PART_TLDS.has(lastTwo)) {
    return parts[parts.length - 3];
  }
  return parts[parts.length - 2];
}
```

Company names are also normalized — strip `Inc`, `LLC`, `Ltd`, `Corp`, `Technologies`, `Solutions`, etc., remove non-alphanumeric characters, lowercase. `"Google Inc."` and `"google.com"` both become `"google"`.

### Match confidence

```
Exact domain match:   1.0    recruiting@amazon.co.uk → "amazon" == job.company "Amazon"
Exact name match:     1.0    normalized company == sender brand
Contains match:       0.95   sender brand ⊆ normalized company (brand ≥ 4 chars)
Reverse contains:     0.95   normalized company ⊆ sender brand (company ≥ 4 chars)
```

Generic email domains (Gmail, Yahoo, Outlook, iCloud, ProtonMail, 15 total) are skipped entirely. A rejection from `hr@gmail.com` doesn't match anything.

This handles ~80% of cases with zero API calls. Most companies email from their corporate domain.

## Keyword classification

Once domain matching identifies the company, keyword classification determines what the email means. Again, fully deterministic — regex against known patterns.

### Rejection patterns (21 core)

```
"unfortunately"
"not moving forward"
"decided not to proceed"
"another candidate"
"position has been filled"
"pursue other candidates"
"backgrounds align more closely"
"we regret to inform"
"after careful consideration"
"we have decided to go with"
"not selected"
"no longer considering"
...
```

### Interview patterns (14)

```
"schedule an interview"
"interview invitation"
"next step in the process"
"would love to chat"
"phone screen"
"technical assessment"
"meet the team"
"availability for"
"calendly.com"
"goodtime.io"
...
```

### Offer patterns (10)

```
"job offer"
"offer letter"
"compensation package"
"pleased to offer you"
"welcome to the team"
...
```

### Receipt patterns (6)

```
"application received"
"thank you for applying"
"we received your application"
"application confirmation"
...
```

The system also classifies rejection *reason* — `position_filled`, `pursuing_others`, `qualifications`, `keep_on_file`, `automated`, or `personal` — based on secondary regex. This feeds back into company intelligence (some companies always send personal rejections, some always send boilerplate).

### Conflict detection

What happens when an email matches both rejection AND interview patterns? Or rejection AND offer? This is more common than you'd think — "Unfortunately, we won't be moving forward with your application for the Senior Engineer role. However, we'd love to schedule an interview for the Staff Engineer position."

The system flags these as conflicts and forces manual review. No auto-application when signals contradict.

## AI fallback

~20% of emails can't be classified deterministically. The sender domain doesn't match any job (recruiter used personal email, company uses a third-party hiring platform like Greenhouse), or the email language is ambiguous.

For these cases, Gemini 2.5 Flash extracts company name, role title, and classification. But raw LLM confidence is unreliable, so a Guardian system recalibrates:

```typescript
// Guardian feature scoring (80% of final confidence)
let score = 0;
if (features.lexical_match)    score += 0.30;  // Standard recruiting boilerplate
if (features.semantic_match)   score += 0.30;  // Clear intent detected
if (features.structure_match)  score += 0.20;  // Professional email layout
if (features.logic_consistency) score += 0.20;  // No conflicting signals

// Penalty per uncertainty flag (max 0.35 total)
score = Math.max(0, score - uncertaintyFlags.length * 0.05);

// Blend: 80% Guardian, 20% raw LLM
final = score * 0.8 + Math.min(rawConfidence, 0.95) * 0.2;
```

The Guardian features are boolean checks the LLM performs on the email content. They're easier for the model to get right than a single confidence float. A rejection email that has standard boilerplate (lexical), clear "we won't be moving forward" language (semantic), professional formatting (structure), and no contradictory signals (logic) scores high on all four. The math is predictable. A raw 0.7 confidence from the LLM becomes 0.82 or 0.94 after Guardian scoring depending on the features.

### Confidence tiers

```
verified:   ≥ 0.90    Auto-apply eligible
probable:   0.70-0.89  Staged for user review
uncertain:  < 0.70     Logged but no action
```

Auto-application requires ≥ 0.95 confidence AND zero guardrail triggers. The bar is deliberately high — a false auto-apply (marking a live job as rejected) is worse than requiring a manual confirmation.

## Role-title disambiguation

When multiple jobs match the same company domain (you applied to 3 roles at Google), the system needs to figure out which one the email is about.

```typescript
function disambiguateByRole(jobs, extractedRole): Job {
  for (const job of jobs) {
    // Exact match
    if (job.title.toLowerCase() === extractedRole.toLowerCase()) return job;
  }
  for (const job of jobs) {
    // Contains match
    if (job.title.toLowerCase().includes(extractedRole.toLowerCase()) ||
        extractedRole.toLowerCase().includes(job.title.toLowerCase())) {
      return { job, score: 0.9 };
    }
  }
  for (const job of jobs) {
    // Levenshtein fuzzy (threshold: 0.6)
    const sim = similarity(job.title, extractedRole);
    if (sim > 0.6) return { job, score: sim };
  }
  // Fallback: most recently created job at that company
  return jobs[0];
}
```

The role title comes from the AI extraction (if AI fallback was used) or isn't available (deterministic path only). When it's not available and multiple jobs match, the system falls back to the most recent application.

## Receipt memory

Application receipt emails ("Thank you for applying") are low-value on their own but useful later. When a rejection arrives from the same domain weeks later, the receipt helps narrow which job it's for.

The system queries the last 12 receipt logs from the same sender domain and scores them:

```
+3 points: same message thread (subject line matches)
+2 points: role title similarity ≥ 0.8
+1 point:  role title similarity ≥ 0.6 or no role/thread data
```

If receipt memory narrows the candidates to a single job, confidence is set to 0.93. This avoids an AI call for what's essentially a lookup.

## Hard guardrails

Some status transitions should never happen automatically:

```
Rejection signal + current status is "offer" or "accepted"
  → guardrail: protected_positive_status
  → reason: you don't auto-reject someone who has an offer

Rejection + offer signals in same email
  → guardrail: offer_signal_conflicts_rejection
  → reason: ambiguous, needs human judgment

Interview + offer signals in same email
  → guardrail: offer_signal_conflicts_interview
  → reason: contradictory, probably a complex email
```

Any guardrail trigger blocks auto-application entirely and stages the update for manual review with the specific guardrail flag visible in the UI.

## Undo support

Every auto-applied status change records the previous status:

```typescript
{
  matched_job_id: job.id,
  staged_status: "rejected",
  previous_job_status: "applied",
  status_auto_applied: true,
  status_auto_applied_at: new Date().toISOString(),
  staged_confirmed_at: null,  // null = within undo window
}
```

The job row gets stamped with `auto_apply_log_id` pointing back to the log entry. The frontend can revert within 5 minutes by restoring `previous_job_status`.

## Response latency intelligence

When a user forwards an email, the provider timestamp reflects the forward date, not the original send date. The system recovers the actual company response time by parsing forwarded message headers:

```typescript
function extractOriginalEmailDate(body: string): Date | null {
  // Gmail:   "---------- Forwarded message ----------\nDate: Thu, 3 Apr 2026 14:22:00 -0400"
  // Outlook: "From: recruiter@company.com\nSent: Thursday, April 3, 2026 2:22 PM"
  // Apple:   "Begin forwarded message:\n...\nDate: April 3, 2026 at 2:22:00 PM EDT"
  // Generic: "On Thu, Apr 3, 2026, recruiter@company.com wrote:"

  // Sanity: not in future (1-day clock drift tolerance), not > 180 days old
  // Original date must be before forward date
}
```

With the accurate send date, the system calculates response latency from application date:

```
Ultra-fast rejection:  < 1 hour    → probably automated ATS filter
Very fast rejection:   1-6 hours   → likely automated
Same-day rejection:    6-24 hours  → could be human, could be batch processing
Normal cycle:          1-3 weeks   → standard
```

This data aggregates across users (anonymized) into company intelligence — ghost rates, average response times, fast-rejection frequency. Useful signal before you spend time tailoring a resume for a company that auto-rejects 90% of applicants within an hour.

## Deduplication

5-minute window on `(user_id, sender, subject)`. Same email forwarded twice won't create duplicate log entries or trigger duplicate status changes.

Exception: forwarding confirmation emails (Gmail's "forwarding-noreply@google.com", subjects containing "forwarding confirmation"). These bypass dedup because the user might need to retry the confirmation link.

## Forwarding confirmation detection

The system detects email provider forwarding setup messages:

```
"gmail forwarding confirmation"
"has requested to automatically forward mail"
"finish setting up forwarding"           (iCloud)
"verify that you own this email address"  (Outlook)
"forwarding rule"                         (generic)
```

When detected, the system auto-activates the sync status for that user. The email is logged with `category: "forwarding"` and doesn't trigger any job pipeline changes.

## What's not stored

Privacy constraint: raw email bodies are never persisted. The `body_text` and `body_snippet` columns are always `NULL`. The log records:

- Sender email, subject line
- Captured category and rejection reason
- Matched job ID and confidence
- Match method (domain, ai_fallback, receipt_memory)
- Guardian features and AI output (for audit trail)
- Response timing metadata
- Guardrail flags

Enough to debug and audit every decision. Not enough to reconstruct the email.

## Results

The system processes emails in the order of single-digit seconds. Domain matching + keyword classification handles ~80% of cases with zero AI cost. The Guardian confidence system keeps false auto-application rate effectively zero — in practice, ambiguous cases get staged for review rather than auto-applied.

The hardest edge cases aren't technical — they're linguistic. Recruiters who write "We'd love to keep in touch for future opportunities" are sending a rejection dressed as a positive signal. The keyword list catches the common ones, but new phrasings appear constantly. The AI fallback exists precisely for these cases, and the Guardian system ensures that even when the LLM is uncertain, the pipeline state doesn't get corrupted.

## Running it

This is part of [P.A.T.H.O.S.](https://yourpathos.app). The Edge Function is at `supabase/functions/ghost-listener/index.ts`. The full system design context is in the [P.A.T.H.O.S. engineering note](https://github.com/thedeutschmark/engineering-notes/tree/main/resume-optimization).
