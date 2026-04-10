# Machine Learning in a Job Search Tool

How I built a prediction system that tells users whether a job application is worth their time — using logistic regression trained on opt-in outcome data, blended with a deterministic scoring engine, with zero external ML libraries.

## The prediction problem

P.A.T.H.O.S. already had a deterministic Apply Confidence score — a formula that combines ATS match, skill gaps, compensation alignment, and company intelligence into a 0-100 number. It works. But it has blind spots. The formula weights are hand-tuned. It can't learn that certain feature combinations predict success better than the weights suggest. It doesn't know that a 75 ATS match with zero critical gaps outperforms a 90 match with three critical gaps, unless I manually encode that interaction.

The ML layer exists to catch what the formula misses. Not replace it — supplement it. The final score is a 60:40 blend: 60% deterministic (explainable, auditable), 40% model (learned from outcomes). If the model is unavailable, the deterministic score stands alone. The system never depends on ML being up.

## Two-layer architecture

```
┌─────────────────────────────────────────────────────┐
│  Public Corpus Layer (anonymous, aggregated)         │
│                                                     │
│  Company ghost rates, interview rates, response     │
│  times. Computed daily from opt-in users.            │
│  No individual data crosses user boundaries.         │
│  k-anonymity enforced (min 10-25 samples).          │
└──────────────────────┬──────────────────────────────┘
                       │ feeds into
┌──────────────────────▼──────────────────────────────┐
│  Per-User Learning Layer (private)                   │
│                                                     │
│  Winning keywords, preferred titles, top-performing  │
│  industries. Outcome-based calibration.              │
│  Never shared. Never exported.                       │
└─────────────────────────────────────────────────────┘
```

The public layer is a network effect — more users contributing outcomes means better company intelligence for everyone. The per-user layer is personal — your specific patterns, calibrated after 11+ tracked outcomes.

Both layers feed features into the ML model, but the model itself is trained on consented, labeled data with company names stripped. The two layers inform the prediction without leaking individual data into the shared model.

## The 30-feature spec

Every optimization run produces a feature snapshot. Exactly 30 features, all normalized to `[0, 1]` or `[-1, 1]`:

| # | Feature | Type | Normalization |
|---|---------|------|---------------|
| 1 | `deterministicApplyConfidenceScore` | Continuous | `/100` |
| 2 | `matchOptimized` | Continuous | `/100` |
| 3 | `matchDelta` | Signed | `/50 → [-1,1]` |
| 4 | `preservationScore` | Continuous | `/100` |
| 5 | `criticalGapCount` | Count | `/6` |
| 6 | `stretchGapCount` | Count | `/6` |
| 7 | `reciprocalStrengthCount` | Count | `/5` |
| 8 | `applicationInstructionCount` | Count | `/10` |
| 9 | `cleanupReplacementCount` | Count | `/10` |
| 10 | `unsupportedKeywordRisk` | Risk | `/100` |
| 11 | `antiAiPolicy` | Boolean | `{0,1}` |
| 12 | `reviewRequired` | Boolean | `{0,1}` |
| 13 | `compliancePassed` | Boolean | `{0,1}` |
| 14 | `companyIntelAvailable` | Boolean | `{0,1}` |
| 15 | `companyGhostRate` | Percent | `/100` |
| 16 | `companyInterviewRate` | Percent | `/100` |
| 17 | `learningTotalOutcomes` | Log count | `log1p(n)/log1p(100)` |
| 18 | `learningCalibrated` | Boolean | `{0,1}` |
| 19 | `hasSalaryTarget` | Boolean | `{0,1}` |
| 20 | `salaryAboveTargetFloor` | Boolean | `{0,1}` |
| 21 | `compensationScoreDelta` | Signed | `/12 → [-1,1]` |
| 22 | `learningScoreDelta` | Signed | `/10 → [-1,1]` |
| 23 | `networkScoreDelta` | Signed | `/10 → [-1,1]` |
| 24 | `roleLevelScore` | Ordinal | `/8` (intern=0, exec=8) |
| 25 | `workModelRemote` | Boolean | `{0,1}` |
| 26 | `workModelHybrid` | Boolean | `{0,1}` |
| 27 | `workModelInPerson` | Boolean | `{0,1}` |
| 28 | `companyIntelConfidenceScore` | Ordinal | `/3` (none=0, high=3) |
| 29 | `includeCoverLetter` | Boolean | `{0,1}` |
| 30 | `acceptedSkillsProvided` | Boolean | `{0,1}` |

No resume text, no job descriptions, no company names, no PII. Everything is a derived numeric signal. The `companyNormalized` field exists in the runtime snapshot for display but is explicitly stripped from training exports:

```sql
mp.feature_snapshot - 'companyNormalized' AS feature_snapshot
```

The normalization functions are pure — `normalizeScore`, `normalizeSigned`, `normalizeCount`, `normalizeLogCount`, `booleanToNumber`. No external dependencies. `log1p` for `learningTotalOutcomes` handles the skewed distribution (most users have 0-20 outcomes, a few have 100+).

## Model: logistic regression from scratch

Binary logistic regression with L2 regularization. No TensorFlow, no scikit-learn, no PyTorch. The entire model is implemented in ~200 lines of JavaScript in `scripts/lib/mlPipeline.js`.

### Why logistic regression

The prediction is binary: will this application result in a positive outcome (interview, offer, accepted) or a negative one (ghosted, rejected)? Logistic regression is the right tool — interpretable coefficients, calibrated probability output, works well with 30 features and modest training data. A neural network would overfit. A random forest would be harder to calibrate. Logistic regression gives me a probability I can trust and coefficients I can inspect.

### Training

```javascript
trainLogisticRegression({
  X,                    // feature matrix
  y,                    // binary labels
  epochs: 1600,
  learningRate: 0.08,   // decays by 0.85 every 400 epochs
  l2: 0.001,
})
```

Full-batch gradient descent with class balancing:

```javascript
positiveWeight = rowCount / (2 * positiveCount);
negativeWeight = rowCount / (2 * negativeCount);
```

Inverse-frequency weighting prevents the model from defaulting to "predict negative" when 70% of applications are ghosted or rejected. Without it, the model learns that predicting failure is almost always right, which is accurate but useless.

Learning rate decays by 0.85 every 400 epochs. Logit inputs are clamped to `[-20, 20]` to prevent numeric overflow in sigmoid. Loss function clips probabilities at `1e-6` to avoid `log(0)`. L2 regularization applies to weights only, not the intercept.

### Platt scaling

Raw logistic regression probabilities can be systematically miscalibrated — the model might output 0.7 when the true positive rate at that score is actually 0.55. Platt scaling corrects this with a second-stage linear transform:

```javascript
fitPlattScaling({
  logits,               // from training set predictions
  labels,               // calibration set ground truth
  epochs: 800,
  learningRate: 0.05,
  l2: 0.0001,
})

// At inference:
calibrated_logit = (raw_logit * slope) + intercept
probability = sigmoid(calibrated_logit)
```

The key: Platt scaling is fit on a separate calibration set, not the training set. If you calibrate on training data, you're just memorizing the training distribution.

### Chronological split

Data is split by time, not randomly:

```
Oldest 60%  →  Training set (fit logistic regression)
Next 20%    →  Calibration set (fit Platt scaling)
Newest 20%  →  Holdout set (evaluation only)
```

Random splits leak temporal information — a model trained on April data and evaluated on March data looks artificially good because it's "predicting" outcomes it's already seen patterns from. Chronological splits force the model to predict future outcomes from past data, which is what it actually has to do in production.

### Feature stability filter

Before training, features with variance below `1e-8` are dropped. If every row has `antiAiPolicy = 0`, that feature is constant and would cause numerical instability in gradient descent. The filter catches degenerate columns automatically.

## The deterministic engine the model supplements

The model doesn't start from zero. The deterministic Apply Confidence score is a hand-tuned signal aggregator:

```
Base: 50 + matchDelta

Match delta:
  88+ ATS match:  +28
  80-87:          +22
  72-79:          +15
  62-71:           +7
  52-61:           +0
  < 52:           -12

Layered signals:
  Critical gaps:           -14 per gap (cap -30)
  Stretch gaps:             -3 per gap (cap -10)
  Reciprocal strengths:     +4 per strength (cap +10)
  Compensation alignment:  -10 to +8
  Learning history:         -3 to +8 (requires 11+ outcomes)
  Network intelligence:    -12 to +10 (from company stats)
  Anti-AI policy:          -12
  Review gate:              -6

Final: clamped [0, 100]
```

Each signal maps to explainable factors shown to the user. "Your score is 62 because: strong ATS match (+22), but 2 critical skill gaps (-28), and this company ghosts 65% of applicants (-8)." The user can see exactly why the score is what it is and what they can do about it.

The ML model can't explain itself this way. That's why it's 40%, not 100%. The deterministic engine carries the explanation. The model carries the correction.

## The 60:40 blend

```javascript
const blendedScore = Math.round(
  (deterministicScore * 0.6) + (modelScore * 0.4)
);
```

If the deterministic engine says 72 and the model says 58, the blended score is `72 * 0.6 + 58 * 0.4 = 66.4 ≈ 66`. The model pulled it down — it learned something about this feature combination that the hand-tuned weights miss.

If the model is unavailable (Edge Function down, no active model, missing features), the blend doesn't happen. Deterministic score passes through unchanged. The user never sees "ML unavailable" — they just get the score they would have gotten before ML existed.

## Inference pipeline

Two paths, one fallback:

```
Client request
    │
    ├── Try: Edge Function (ml-inference)
    │     ├── Authenticate JWT
    │     ├── Fetch consent profile + agent run (parallel)
    │     ├── Call ml_predict_outcome RPC
    │     ├── Compute blend
    │     ├── Log prediction (with consent snapshot)
    │     └── Return
    │
    ├── Fallback: Direct RPC (ml_predict_outcome)
    │     ├── Load active model
    │     ├── Dot product + Platt scaling + sigmoid
    │     └── Return probability
    │
    └── Fallback: { available: false }
          └── Deterministic score used unchanged
```

The RPC function handles the math entirely in Postgres — loads coefficients from `ml_models`, iterates the feature vector, computes the dot product, applies Platt scaling, runs sigmoid. Supports nested JSONB paths via `string_to_array(feature_name, '.')` for features like `compensationDetails.salaryMidpoint`.

## Publication gate

A new model can't go live just because it finished training. It has to beat the deterministic baseline on at least 2 of 4 metrics:

```javascript
const signals = [
  comparison.aucRocDelta >= 0.01,       // AUC improvement ≥ 1%
  comparison.brierImprovement > 0.003,   // Brier score improvement
  comparison.logLossImprovement > 0.01,  // Log loss improvement
  comparison.eceImprovement > 0,         // Any ECE improvement
];

const recommended = signals.filter(Boolean).length >= 2
                 && comparison.aucRocDelta >= -0.01;  // no >1% AUC degradation
```

AUC-ROC measures discrimination (can the model rank positives above negatives?). Brier score measures calibration + discrimination combined. Log loss penalizes confident wrong predictions. ECE (Expected Calibration Error) measures whether predicted probabilities match observed rates across 10 bins.

The hard floor: AUC-ROC cannot degrade by more than 1% absolute. A model that improves calibration but can't rank outcomes is worse than the deterministic baseline.

### Evaluation metrics

AUC-ROC is computed via exact Wilcoxon-Mann-Whitney — for every (positive, negative) pair, check if the positive scored higher. No approximations, no binning. With training datasets in the hundreds to low thousands, exact computation is cheap and eliminates sampling noise.

Calibration is measured across 10 equal-width bins: for each bin, compare average predicted probability against average actual outcome rate. The weighted average of those gaps is the ECE. A well-calibrated model saying "0.7 probability" should see ~70% positive outcomes among all predictions in the 0.65-0.75 bin.

## Publishing a model

```bash
node scripts/publishApplyConfidenceModel.js --artifact latest-training-report.json
```

The publish script does an atomic swap:

1. Snapshot current active model ID (for rollback)
2. Insert new model with `is_active = false`
3. Deactivate all models for this task version
4. Activate new model
5. If step 4 fails, re-activate the snapshot

There is never a window with zero active models. If the new model insert succeeds but activation fails, the old model stays live. The publish is `--force`-able if the publication gate didn't recommend it, but the default requires passing the gate.

Model metadata includes training hyperparameters, evaluation metrics, calibration coefficients, feature baselines for drift detection, and the training report artifact path for auditability.

## Consent and privacy

Shared training consent is explicit opt-in, default false:

```sql
ALTER TABLE profiles
  ADD COLUMN shared_training_consent boolean NOT NULL DEFAULT false;
  ADD COLUMN shared_training_consent_version text;
  ADD COLUMN shared_training_consent_updated_at timestamptz;
```

Every prediction log captures the user's consent status at prediction time as an immutable field. If a user later revokes consent, their previously-consented rows remain in the training pool (they were consented when created), but new predictions won't be eligible.

The training export function enforces multiple constraints:

```sql
WHERE mp.consented_for_shared_training = true    -- explicit opt-in
  AND ar.outcome_status IS NOT NULL               -- must have a label
  AND mp.feature_snapshot != '{}'::jsonb           -- non-empty features
```

Plus the `companyNormalized` strip. In a small dataset, company name + role level + date could identify an individual. The model doesn't need the company name — it has `companyGhostRate` and `companyInterviewRate` as anonymized aggregates.

### Privilege separation

| Function | authenticated | service_role |
|---|---|---|
| `set_shared_training_consent` | GRANT | GRANT |
| `ml_predict_outcome` | GRANT | GRANT |
| `export_ml_training_data` | REVOKE | GRANT |
| `ml_training_stats` | REVOKE | GRANT |
| `ml_monitoring_snapshot` | REVOKE | GRANT |

Users can request predictions and manage their own consent. Training data export is service-role only — it runs in offline scripts, not in the browser.

## Company intelligence layer

The public corpus feeds two tables: `intel_company_stats` and `intel_role_stats`. A daily Edge Function runs at 03:00 UTC, reads from `intel_eligible_jobs` (a view with anti-poisoning filters), and upserts aggregates.

### Eligibility filters

- User must have `contributes_to_intelligence = true`
- Account age ≥ 7 days (anti-poisoning)
- Company name must be present and normalized
- Application age ≥ 24 hours
- `applied_date` must be non-null
- Auto-ghost: any application still `applied` after 30 days is reclassified as `ghosted`

The 7-day account age filter blocks throwaway accounts from poisoning aggregates. The test data exclusion (30+ apps from one user in a single day) catches automated testing.

### k-anonymity

Intelligence exports enforce minimum sample sizes:

```sql
IF p_min_sample < 10 THEN p_min_sample := 10; END IF;
```

Default minimum is 20 samples. Hard floor is 10, regardless of parameter. Companies below threshold are excluded entirely — not surfaced with a "low confidence" warning, just absent.

Confidence bands:

| Band | Samples | Surfaced |
|---|---|---|
| high | 25+ | Yes |
| medium | 10-24 | Yes |
| low | 5-9 | Yes |
| none | 0-4 | No |

### Company name normalization

`normalizeCompanyName()` strips legal suffixes, lowercases, and resolves aliases:

```
"Google LLC"          → "google"
"Alphabet"            → "google"
"DeepMind"            → "google"
"Amazon Web Services" → "amazon"
"GitHub"              → "microsoft"
"Slack"               → "salesforce"
```

Three iterative passes of suffix stripping catch nested patterns like "Acme Holdings Corp." → "acme".

## Drift detection

`ml_monitoring_snapshot` tracks feature distribution shifts over a 7-day rolling window:

For 6 key features (`deterministicApplyConfidenceScore`, `matchOptimized`, `preservationScore`, `companyGhostRate`, `companyInterviewRate`, `criticalGapCount`), it computes:

- **Recent mean** vs **baseline mean** (from training data, stored in model metadata)
- **Shift percentage**: `(recent - baseline) / |baseline|`

Plus system health:
- **Failure rate**: fraction of predictions returning `available: false`
- **Calibration gap**: `avg(predicted) - observed_positive_rate`
- **Bad prediction rate**: high-score negatives + low-score positives divided by total labeled predictions

"High-score negative" means the model said 65+ but the user got ghosted or rejected. "Low-score positive" means the model said <50 but the user got an interview. Both are model failures, and tracking their rate over time tells you when the model is degrading.

## Benchmark infrastructure

The training pipeline includes a Kaggle dataset integration (`snehaanbhawal/resume-dataset`, 2,484 resumes across 24 categories) for benchmark testing. The ingest script maps each category to a plausible job description, extracts resume sections, and generates fixture files matching the existing benchmark format.

This lets me test the deterministic scoring engine and ML pipeline against a large corpus of real resumes without waiting for production outcome data. The fixtures support deterministic benchmarks (ATS scoring accuracy), live benchmarks (LLM generation testing), and parse quality auditing (PDF extraction fidelity).

## What I'd do differently

**Use a proper train/validation/test split with early stopping.** Fixed epoch count with learning rate decay works, but early stopping on a validation set would adapt to dataset size automatically and prevent subtle overfitting.

**Add interaction features.** The model sees `criticalGapCount` and `matchOptimized` independently. A feature like `matchOptimized * (1 - criticalGapCount/6)` would let it learn that high match with gaps is different from high match without gaps, without needing the model to discover this from raw features alone.

**Explore gradient-boosted trees.** Logistic regression is right for now — interpretable, fast, works with small data. But as training data grows past 10k rows, a small XGBoost model would capture nonlinear interactions that logistic regression misses. The 60:40 blend architecture means swapping the model is straightforward — the deterministic engine stays, only the 40% changes.

**Better temporal features.** The model doesn't know what month it is, what the unemployment rate is, or whether it's hiring season. Seasonal hiring patterns are strong (January surge, summer slowdown). Adding a month-of-year feature and economic indicators would capture macro trends the current feature set misses entirely.

## Running it

This is part of [P.A.T.H.O.S.](https://yourpathos.app). The ML pipeline is at `scripts/lib/mlPipeline.js`, training at `scripts/trainApplyConfidenceModel.js`, inference at `supabase/functions/ml-inference/index.ts`, the deterministic engine at `src/utils/applyConfidence.js`, and the feature spec at `src/utils/mlTelemetryPayload.js`. The full architecture document is at `KNOWLEDGE_BASE/ML_ARCHITECTURE.md`.
