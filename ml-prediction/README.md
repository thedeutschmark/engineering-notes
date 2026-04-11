# Machine Learning in a Job Search Tool

*Adding a learned prediction layer on top of a deterministic scoring engine without giving up transparency or control.*

I added a learned prediction layer to P.A.T.H.O.S. because the deterministic scoring engine was useful but predictably limited.

A hand-built score can be transparent, stable, and easy to reason about. It also cannot discover much beyond the interactions I explicitly encoded into it.

The ML layer exists to pick up some of that missing structure **without replacing** the deterministic system underneath it.

---

## The problem

The deterministic Apply Confidence score already did a lot of work:

- ATS alignment
- gap severity
- compensation fit
- company intelligence
- compliance and review gates

That made it explainable, which I cared about. But it also meant the weights were ultimately hand-tuned. If real outcomes suggested the product was underestimating or overestimating certain combinations, the system could only improve as fast as I updated the rules.

That is where the learned layer became useful.

## Why I did not let the model take over

The product would have been worse if I had simply replaced the deterministic engine with a model score.

I did not want the user to lose the part they could actually inspect.

So the ML layer is deliberately **subordinate**. It can nudge the final score, but it does not erase the baseline logic. The blending is explicit: the deterministic score carries the majority weight, the model acts as a correction. That matters for rollout, for failure handling, and for user trust.

> The system is better when the learned layer behaves like a correction term rather than like a sovereign judge.

---

## The feature space

The model sees a structured feature vector derived from optimization runs and downstream outcomes. Not raw resumes and job descriptions.

That choice was partly about privacy and partly about control. Feature-based training gives a cleaner interface between product logic and model training. It also makes it easier to reason about what the model is actually being allowed to learn from.

The features fall into a few categories:

**Optimization signals** - the direct output of the ATS engine: match quality before and after optimization, how much the match improved, how much original voice was preserved, hard and soft skill gap counts, and fragile keyword risk.

**Company intelligence** - aggregate data from the network: ghost rate, interview rate, and a confidence score reflecting how much data backs the intel for that company.

**User history** - per-user learning signals: how many tracked outcomes the user has, whether they have enough outcomes to be statistically meaningful, and how much their personal history is adjusting the score.

**Context signals** - job and application metadata: seniority level, work model flags, whether the user set compensation expectations, and whether the job meets them.

Every feature is normalized to a consistent range before training. Count features use fixed denominators rather than data-driven normalization so the feature space does not shift between training runs.

## Why logistic regression

Logistic regression was the right first model for this problem.

Not because it is glamorous. Because it **matched the maturity of the data and the product**.

The task is binary (positive outcome vs. negative). The feature space is structured. The dataset is meaningful but not massive. The rollout needs to be controlled. The output needs to be interpretable enough that I can compare it sensibly against the deterministic baseline.

Training uses standard gradient descent with learning rate decay and L2 regularization to keep the weights from overfitting to sparse features. Class weighting uses inverse frequency so the model does not just learn to predict the majority class.

A more expressive model might eventually beat it. That was not the point of the first deployment.

---

## Calibration mattered as much as ranking

I cared less about the model sounding sophisticated and more about whether its output **behaved responsibly** inside a product.

A model that sorts examples reasonably well but produces unstable probability estimates is awkward inside a user-facing score. If the model says 0.72 for one application and 0.74 for another, those numbers need to mean something stable enough to display.

The system uses **Platt scaling** on a held-out calibration pool that the logistic regression never saw during training. The data is split chronologically into three pools: training, calibration, and holdout evaluation, always ordered oldest to most recent.

Platt scaling fits a two-parameter transform on the raw logits using the calibration pool. The result is a calibrated probability that better reflects the actual observed outcome rate at each confidence level.

That requirement pushed the system toward a more careful pipeline than I would have needed for a purely internal ranking model.

## Time-aware evaluation

One thing I cared about early was evaluating the model in a way that actually resembled the product.

> The real task is not "predict a random held-out row." It is "predict future outcomes from past data."

That is why the split is chronological rather than random. The holdout set is always the most recent slice of labeled data, so evaluation metrics reflect the model's ability to generalize forward in time rather than interpolate within a shuffled dataset.

The publication gate requires the model to beat the deterministic baseline on multiple evaluation metrics without materially degrading discrimination. If it does not clear that bar, the product keeps using the deterministic path. **No model is better than a bad model.**

## Publishing and rollback

Another important decision was treating model promotion as a **gate** instead of an automatic consequence of training.

A new model should not become active just because it exists. It should have to outperform the baseline in a way that is actually useful to the product.

Publication is an atomic operation with rollback protection. If the new model fails to activate, the system restores the previous model. The product is never left without an active model.

That discipline matters even more in a system like this because the deterministic engine is already usable. The learned layer has to justify itself. If it does not, the product is not blocked. It simply keeps using the stronger baseline.

---

## Consent and privacy

This was one of the places where I wanted the architecture to **reflect the product promise** rather than decorate it.

Shared-model training is opt-in. The training export function only returns rows where the user has consented and the outcome has been labeled. It strips company identifiers before export. Prediction is available to all users, but training contribution requires explicit consent.

The system is designed around structured features instead of raw personal text as the main training unit. Prediction and training eligibility are tracked in a way that can be audited later instead of reconstructed from vibes.

That does not make the ML layer magically pure. It just means the product is trying to be deliberate about what it learns from and why.

## Public intelligence versus personal learning

One useful distinction in the system is between:

- **aggregate intelligence** from many users
- **user-specific patterns** from one person's history

Those are different kinds of learning and they should not be treated as the same thing.

The aggregate layer is useful for company and market behavior. The personal layer is useful for figuring out what seems to work unusually well or poorly for one user over time.

Keeping those ideas separate made the prediction system easier to reason about and harder to overclaim.

---

## Why the system works

The prediction layer works because it does not try to do everything.

It sits on top of an explainable baseline, learns from outcome data, and changes the final score only within a controlled boundary. If inference fails or the active model is not good enough, the product still has a sensible deterministic path underneath it.

That is the right balance for this stage of the system.

## What I would change next

- add a more explicit validation-and-early-stopping path for training
- test a stronger model family once the dataset is materially larger
- keep improving interaction and temporal features without making the feature space opaque
- tighten monitoring around calibration drift as the product evolves

I still think the first model was the right one. The point was not to build the fanciest predictor. The point was to add learned correction without giving up control.

## Running it

This is part of [P.A.T.H.O.S.](https://yourpathos.app). The broader product context is in [How I built P.A.T.H.O.S.](../how-i-built-pathos/).
