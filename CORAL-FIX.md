5/22/26 11:32AM
Key problem: domain adaptation is meant to handle distribution differences, but CORAL is a relatively simple method that only matches broad statistical properties, not deeper class-specific shifts like changed label frequencies or changed class structure.

# CORAL Improvement Plan (Domain Adaptation PoC)

## Current Observation

- Source test accuracy: 0.8989
- Target test accuracy (no CORAL): 0.6474
- Target test accuracy (with CORAL): 0.5903

CORAL lowered plain target accuracy in this run, but it also shifted class behavior (some attack recalls improved while majority-class behavior degraded).

## Why This Can Happen

CORAL aligns global second-order feature statistics (mean/covariance) between domains. It does not explicitly align class-conditional distributions or class priors. If source/target class proportions and intra-class geometry differ substantially, global alignment can over-correct some regions of feature space.

In this project, source and target class mixtures differ a lot (especially Benign vs multiple attack groups), so a pure global CORAL transform can trade off overall accuracy for different class-level behavior.

## Proposed Improvements

### 1) Improve covariance estimation robustness

Replace plain covariance estimation (`np.cov`) with shrinkage covariance:

- `sklearn.covariance.LedoitWolf` (recommended first)
- optional comparison with `OAS`

Why:

- Better-conditioned covariance under imbalance/noise
- More stable inverse square root in CORAL
- Less amplification of near-null directions

Implementation sketch:

1. Fit shrinkage covariance on source train feature matrix and target train feature matrix.
2. Store estimator covariance outputs in `coral_source_stats.joblib` and `coral_target_stats.joblib`.
3. Record condition number and min/max eigenvalues for diagnostics.

### 2) Increase spectral floor for inverse square root

Current floor (`eps=1e-6`) allows large amplification in low-variance directions.

Test floors:

- `1e-4`
- `1e-3`

Why:

- Reduces instability from tiny eigenvalues
- Prevents over-aggressive whitening of target covariance

### 3) Add CORAL damping (partial transform)

Use blended adaptation instead of full transform:

`X_adapted = X_target + lambda * (X_coral - X_target)`

Grid:

- `lambda in [0.1, 0.25, 0.5, 0.75, 1.0]`

Why:

- Controls over-correction
- Often improves accuracy under partial alignability

### 4) Add pre/post adaptation diagnostics

For each run, log:

- source/target covariance distance (Frobenius norm) before and after
- condition numbers of source and target covariance matrices
- min/median/max eigenvalues
- per-feature mean shift magnitude

Why:

- Confirms whether alignment is numerically meaningful
- Helps distinguish "bad transform" from "bad objective metric"

### 5) Evaluate with multiple metrics, not only accuracy

Report all of:

- accuracy
- macro F1
- weighted F1
- balanced accuracy
- per-class recall

Why:

- Domain adaptation can improve minority attack detection while reducing overall accuracy
- Single-metric optimization (accuracy only) can hide useful transfer behavior

### 6) Add non-adapted model baselines

Compare CORAL + linear model against:

- tuned `SGDClassifier` (hinge and log-loss)
- `HistGradientBoostingClassifier` (no CORAL baseline)

Why:

- Checks whether model capacity, not just adaptation, is the bottleneck
- Provides realistic fallback if CORAL does not help this domain pair

## Suggested Experiment Matrix

1. Baseline: no CORAL + current SGD.
2. CORAL full (`lambda=1.0`) + shrinkage covariance + eps sweep.
3. CORAL damped (`lambda` sweep) + best eps.
4. No CORAL + HistGradientBoosting baseline.
5. Compare metrics and confusion patterns, then select by project objective:
	- If objective is max overall accuracy, choose highest accuracy model.
	- If objective is attack coverage, weigh macro/balanced metrics more.

## Answer: "Isn't prior shift exactly what DA is for?"

Yes, partially. Domain adaptation is meant to address distribution shift, including covariate shift and sometimes mild prior shift. But not all DA methods handle all shift types equally well.

Important nuance:

- CORAL primarily addresses global feature distribution mismatch (second-order moments).
- CORAL does not explicitly correct label shift / class-prior shift.
- CORAL also does not model class-conditional multimodality.

So prior differences are not inherently a problem for DA in general, but they can be too severe for plain global CORAL when combined with strong class-conditional differences.

For this dataset pair, the mismatch appears strong enough that plain CORAL is likely under-specified unless regularized (shrinkage + damping) and evaluated with objective-aligned metrics.

## Practical Recommendation

Start with this minimal upgrade path:

1. Shrinkage covariance (LedoitWolf)
2. Increase spectral floor to `1e-4` or `1e-3`
3. Add damping (`lambda` sweep)
4. Select model by both accuracy and macro/balanced metrics

If CORAL still underperforms, keep no-CORAL baseline and consider class-conditional/domain-adversarial methods as next step.