# Machine Learning in a Job Search Tool

How I added a learned prediction layer to a job-search product without giving up explainability: logistic regression trained on consented outcome data, calibrated after training, and blended with a deterministic scoring engine instead of replacing it.

## The prediction problem

P.A.T.H.O.S. already had a deterministic Apply Confidence score. It combined ATS match, skill gaps, compensation alignment, and company intelligence into a single number that users could inspect and understand.

That score was useful, but it had a predictable limitation: the weights were hand-tuned. It could not learn that certain feature combinations mattered more than I originally expected unless I encoded those interactions manually.

The ML layer exists to correct for that. It does not replace the deterministic engine. It supplements it.

```text
final score = 60% deterministic + 40% model
```

If the model is unavailable, the deterministic score passes through unchanged.

## Two-layer data design

The system uses two information layers:

```text
Public corpus layer
  anonymous aggregates such as company ghost rates,
  interview rates, and response timing

Per-user layer
  private outcome patterns such as stronger titles,
  skills, and industry alignment for one user
```

The public layer improves shared company intelligence. The per-user layer improves personalization. Both can generate features, but neither requires storing raw resume text or raw job descriptions in training data.

## The 30-feature snapshot

Each optimization run produces a numeric feature snapshot. Exactly 30 features, normalized to `[0, 1]` or `[-1, 1]`:

| # | Feature | Type | Normalization |
|---|---|---|---|
| 1 | `deterministicApplyConfidenceScore` | continuous | `/100` |
| 2 | `matchOptimized` | continuous | `/100` |
| 3 | `matchDelta` | signed | `/50` |
| 4 | `preservationScore` | continuous | `/100` |
| 5 | `criticalGapCount` | count | `/6` |
| 6 | `stretchGapCount` | count | `/6` |
| 7 | `reciprocalStrengthCount` | count | `/5` |
| 8 | `applicationInstructionCount` | count | `/10` |
| 9 | `cleanupReplacementCount` | count | `/10` |
| 10 | `unsupportedKeywordRisk` | continuous | `/100` |
| 11 | `antiAiPolicy` | boolean | `{0,1}` |
| 12 | `reviewRequired` | boolean | `{0,1}` |
| 13 | `compliancePassed` | boolean | `{0,1}` |
| 14 | `companyIntelAvailable` | boolean | `{0,1}` |
| 15 | `companyGhostRate` | continuous | `/100` |
| 16 | `companyInterviewRate` | continuous | `/100` |
| 17 | `learningTotalOutcomes` | log count | `log1p(n)/log1p(100)` |
| 18 | `learningCalibrated` | boolean | `{0,1}` |
| 19 | `hasSalaryTarget` | boolean | `{0,1}` |
| 20 | `salaryAboveTargetFloor` | boolean | `{0,1}` |
| 21 | `compensationScoreDelta` | signed | `/12` |
| 22 | `learningScoreDelta` | signed | `/10` |
| 23 | `networkScoreDelta` | signed | `/10` |
| 24 | `roleLevelScore` | ordinal | `/8` |
| 25 | `workModelRemote` | boolean | `{0,1}` |
| 26 | `workModelHybrid` | boolean | `{0,1}` |
| 27 | `workModelInPerson` | boolean | `{0,1}` |
| 28 | `companyIntelConfidenceScore` | ordinal | `/3` |
| 29 | `includeCoverLetter` | boolean | `{0,1}` |
| 30 | `acceptedSkillsProvided` | boolean | `{0,1}` |

The important constraint is what is not there:

- no raw resume text
- no raw job descriptions
- no company names in the exported training payload
- no personal identifiers

At runtime the system may still carry a normalized company string for display or debugging, but the export path strips it before training:

```sql
mp.feature_snapshot - 'companyNormalized' AS feature_snapshot
```

## Why logistic regression

The task is binary: does this application eventually produce a positive outcome or not?

For the dataset size and feature set I had, logistic regression was the right starting point:

- interpretable coefficients
- calibrated probability output after post-processing
- fast training and cheap inference
- strong baseline behavior with a modest number of numeric features

I could have reached for a more expressive model, but the cost would have been weaker interpretability and a less controlled first deployment.

## Training from scratch

The model is implemented in plain JavaScript in `scripts/lib/mlPipeline.js`.

Training uses full-batch gradient descent with L2 regularization:

```javascript
trainLogisticRegression({
  X,
  y,
  epochs: 1600,
  learningRate: 0.08,
  l2: 0.001,
})
```

There are a few practical safeguards in the training loop:

- class balancing so the model does not collapse toward the majority negative class
- logit clamping to avoid sigmoid overflow
- probability clipping to avoid `log(0)`
- L2 regularization on weights only, not the intercept
- learning-rate decay every 400 epochs

Class balancing in particular mattered. In job-search data, negative outcomes are common enough that an unbalanced model can achieve passable accuracy by being pessimistic all the time. That is not useful product behavior.

## Platt scaling

Raw logistic-regression output is not guaranteed to be well calibrated even when ranking is decent, so I fit a second-stage calibration layer with Platt scaling:

```javascript
fitPlattScaling({
  logits,
  labels,
  epochs: 800,
  learningRate: 0.05,
  l2: 0.0001,
})

// inference
calibratedLogit = rawLogit * slope + intercept
probability = sigmoid(calibratedLogit)
```

The important part is the split: Platt scaling is fit on a calibration set, not the same rows used to train the base model.

## Time-aware data split

The data split is chronological:

```text
oldest 60% -> train
next 20%   -> calibration
newest 20% -> holdout
```

That matters more than it sounds. A random split can leak temporal structure and make the model look better than it really is. In production, the task is always "predict future outcomes from past outcomes," so the evaluation split should mirror that.

## Feature stability filtering

Before training, features with near-zero variance are removed. Constant columns add noise and can create numerical instability without carrying information.

That filter became useful as the feature set evolved. Some signals are common in one product phase and nonexistent in another. The pipeline handles that automatically instead of assuming every column is always meaningful.

## The deterministic engine underneath the blend

The model only owns 40% of the final score because the deterministic engine carries the explanation layer.

At a high level, the deterministic score combines:

- ATS match
- critical and stretch skill gaps
- reciprocal strengths
- compensation alignment
- learning history when sample size is sufficient
- company and network intelligence
- review and compliance gates

The result is a score users can inspect:

```text
strong ATS match            +22
2 critical skill gaps       -28
company ghosts frequently    -8
final deterministic score    62
```

That is the piece the user can reason about directly. The model is there to nudge the score when historical outcomes suggest the hand-tuned weights are missing something.

## The 60:40 blend

The blend is intentionally simple:

```javascript
const blendedScore = Math.round(
  deterministicScore * 0.6 + modelScore * 0.4
);
```

If the deterministic score says `72` and the model score says `58`, the user sees `66`. The learned layer can pull the result up or down, but it cannot erase the explainable baseline.

That design also made rollout safer. If inference fails, the product still behaves in a familiar way.

## Inference path

There are two inference routes and one graceful fallback:

```text
client request
  -> Edge Function inference path
  -> direct RPC fallback
  -> deterministic-only fallback if neither is available
```

The database RPC handles the actual dot product, calibration transform, and sigmoid evaluation inside Postgres. That keeps the prediction surface small and makes it easier to use the same active model artifact from more than one runtime path.

## Publication gate

A trained model does not go live automatically. It has to clear a promotion gate relative to the deterministic baseline:

```javascript
const signals = [
  comparison.aucRocDelta >= 0.01,
  comparison.brierImprovement > 0.003,
  comparison.logLossImprovement > 0.01,
  comparison.eceImprovement > 0,
];

const recommended =
  signals.filter(Boolean).length >= 2 &&
  comparison.aucRocDelta >= -0.01;
```

That gate asks for improvement on at least two useful dimensions while rejecting models that materially degrade ranking quality.

The metrics are:

- `AUC-ROC` for ranking quality
- `Brier score` for probability quality
- `log loss` for penalizing confident mistakes
- `ECE` for calibration quality

## Evaluation

AUC-ROC is computed exactly with pairwise ranking rather than an approximation. At the dataset sizes I had, that was still cheap and removed one source of noise from evaluation.

ECE is computed across ten equal-width probability bins. It is not perfect, but it is a practical sanity check for whether predicted probabilities line up with observed outcomes.

I cared about calibration because this score is surfaced to users as advice. If the model says 0.7, it should mean something close to 70% under the conditions where it is used.

## Publishing and rollback

Publishing is done through a script:

```bash
node scripts/publishApplyConfidenceModel.js --artifact latest-training-report.json
```

The publish flow snapshots the current active model, inserts the candidate artifact, switches activation, and restores the previous model if activation fails. The goal is not a perfect ceremony. It is a practical deployment path where a bad activation does not strand the system without an active model.

## Consent and privacy

Shared-model training is explicit opt-in:

```sql
ALTER TABLE profiles
  ADD COLUMN shared_training_consent boolean NOT NULL DEFAULT false,
  ADD COLUMN shared_training_consent_version text,
  ADD COLUMN shared_training_consent_updated_at timestamptz;
```

Training exports only rows that are:

- explicitly consented
- labeled with an outcome
- non-empty in feature space

And again, the export path strips company identifiers from the feature payload.

Prediction logs also capture the consent snapshot at prediction time so training eligibility can be audited later instead of inferred retroactively.

## Company intelligence

The public corpus feeds daily company and role aggregates such as:

- ghost rate
- interview rate
- response timing

Those aggregates come with eligibility filters:

- the user must contribute to intelligence
- the account must be old enough to reduce poisoning risk
- the application needs enough age and metadata to be meaningful

### k-anonymity and surfacing thresholds

Exports enforce a minimum sample size of 10, with a default threshold of 20 for most views. In practice I treated the confidence bands like this:

| Band | Samples | Surfaced |
|---|---|---|
| high | 25+ | yes |
| medium | 10-24 | yes |
| none | 0-9 | no |

That keeps the public layer useful without pretending tiny samples are stable.

### Company normalization

Company names are normalized and, where helpful, mapped through an alias layer:

```text
"Google LLC"          -> "google"
"Alphabet"            -> "google"
"DeepMind"            -> "google"
"Amazon Web Services" -> "amazon"
"GitHub"              -> "microsoft"
"Slack"               -> "salesforce"
```

That was necessary because otherwise the aggregate layer fragments into several near-duplicate entities and loses value.

## Drift detection

The monitoring layer tracks:

- recent vs training-baseline means for several key features
- inference failure rate
- average predicted probability vs observed positive rate
- obvious miss patterns such as high-score negatives and low-score positives

The point is not advanced MLOps. It is enough monitoring to tell when the model is becoming misaligned with the live product.

## Benchmark support

I also wired in a Kaggle resume dataset for benchmark and fixture generation. That does not replace real outcome data, but it is useful for stress-testing extraction quality, deterministic scoring paths, and regression behavior without waiting for live traffic to accumulate.

## What I would change next

- add a validation split with early stopping instead of fixed epochs
- introduce a small set of explicit interaction features
- test a tree-based model once the dataset is materially larger
- add temporal features for hiring seasonality and macro conditions

I still think logistic regression was the right first model. It matched the maturity of the data and the need for controlled rollout.

## Running it

This is part of [P.A.T.H.O.S.](https://yourpathos.app). The training and inference code lives in `scripts/lib/mlPipeline.js`, `scripts/trainApplyConfidenceModel.js`, `supabase/functions/ml-inference/index.ts`, `src/utils/applyConfidence.js`, and `src/utils/mlTelemetryPayload.js`.
