# Domain Adaptation (CORAL) Proof of Concept

A controlled, offline experiment to verify the feasibility and effectiveness of the prospective domain adaptation feature.

## Overview

Feature space and label space alignment across source and target datasets must be done for CORAL domain adaptation. Label space alignment involves collapsing each dataset's specific attack classes into broader meta-categories representing a shared label space.

In the app, the implemented DA feature will consist of a target-data collection mechanism and will perform preprocessing and covariance calculation in the online app (as opposed to in an offline notebook environment). In this offline test, all procedures and computation will be performed in a notebook environment; the target-domain data (captured lab data) will be processed and feature-mapped in advance alongside the source-domain data (CICIDS2017), and CORAL calculation, model inference, and results extraction will be done in the notebooks as well.

## New Version Info

This iteration uses captured lab data as the target dataset. The data captured from the lab was scaled by the IDS capture pipeline; lab data from the json files are pre-scaled. The same scaler object used during capture is imported and used to scale the source data in src_preprocessing.ipynb to ensure consistency.

The lab collected dataset is substantially smaller than the previous target (csecicids2018) and source (cicids2017) datasets in number of rows, and reduced feature set. This is because of the nfstream to cicflowmeter capture technology discrepancy resulting in dropped columns in the IDS app's feature mapping stage. To account for this, the shared schema artifact shared_feature_space.json was reduced from the original 77 feature schema to the 70 feature structure of the captured data. 

Attack samples from csecicids2018 were injected into the benign-only lab data at a benign/attack proportion of roughly 75/25. Injection took place before train/test splitting; attack samples were present in the train split from which coral statistics were extracted and in the test split which the classifier was evaluated on with and without the coral transform applied.

The decision to include attacks in the train split which coral "learns" from was a choice influenced by a previous experiment's attack recall collapse caused by benign-only learning. However the most recent experiment (coral learning from a benign+attack train split) also resulted in recall collapse, likely caused by insufficient data size.

Using a preliminary linear classifier (SGD hinge) for evaluation, coral improved headline accuracy but collapsed recall. The coral transformation essentially made the classifier a benign-only predicter.

| Evaluation | Accuracy |
|---|---|
| Source test split | 0.8996 |
| Target test split (no CORAL) | 0.6705 |
| Target test split (with CORAL, best config) | 0.7425 |

Implementing 2 improvements to the data processing procedures, ie (1) standardizing source data benign/attack proportion to match that of target (75/25), and (2) changing the coral parameter sweep best-configuration selection metric from accuracy to macro recall, did not improve coral results. In fact they, further elucidated the negative impact of coral by unveiling the inflated accuracy  results caused by benign-prediction domination and demonstrating direct recall collapse caused by using coral.

## Datasets

| Role | Dataset |
|------|---------|
| Source | CICIDS2017 |
| Target | captured lab data |

## Shared Artifacts

The CORAL framework assumes source and target datasets are represented in the same feature and label spaces before adaptation is applied. To enforce this consistently, shared external contract files are used so both pipelines consume the same canonical definitions.

This project uses shared contract artifacts under `data/processed/`:

- `shared_feature_space.json`: canonical ordered list of shared features.
- `shared_label_space.json`: canonical ordered list of shared labels.
- `source_label_map.json`: raw source labels --> shared labels

Both datasets are mapped into the same canonical label space before encoding so label
integers stay consistent across source/target training and evaluation.

- The shared feature-space JSON contains the canonical ordered list of shared features.
- Both preprocessing notebooks consume this file directly.
- Feature alignment is strict by default:
   - Columns not in the contract are dropped.
   - Missing contract columns raise an error.
- Preserving column order is required so reused scalers and downstream models
   receive the same input schema across source and target domains.

## Steps

1. **Prepare CICIDS2017 and lab data**
   - Apply same preprocessing and scaling
   - Normalize feature space
   - Extract covariance statistics
2. **Train model on CICIDS2017**
3. **Calculate CORAL transform**
4. **Perform model inference on lab data**
   - With CORAL
   - Without CORAL
5. **Compare performance** using ground truth labels

## Preprocessing Notebook Steps

The tables below reflect the current non-diagnostic cell-title steps from both preprocessing notebooks.

| src_preprocessing.ipynb | trg_preprocessing.ipynb |
|---|---|
| 1. Imports | 1. Imports |
| 2. Import and concatenate CSVs | 2. Import and parse JSON files |
| 3. Data sanitization | 3. Data sanitization |
| 4. Feature-space alignment | 4. Inject attack samples into data |
| 5. Label-space alignment | 5. Feature-space alignment |
| 6. Train/Test Split | 6. Label-space alignment |
| 7. Scaling (apply imported scaler) | 7. Train/Test Split |
| 8. Label encoding (save encoder for target data processing) | 8. Label encoding (reuse encoder of shared label space) |
| 9. Calculate and export covariance and mean statistics | 9. Calculate and export covariance and mean statistics |
| 10. Export processed data | 10. Export processed data |

## Execution Order

Run notebooks in this order so all required artifacts exist before training and CORAL
evaluation:

1. `src_preprocessing.ipynb`
2. `trg_preprocessing.ipynb`
3. `training_and_eval.ipynb`

## Pipeline Outputs

After preprocessing and before evaluation, these outputs should exist:

- `data/processed/source/train.csv`
- `data/processed/source/test.csv`
- `data/processed/target/train.csv`
- `data/processed/target/test.csv`
- `models/source_scaler.joblib`
- `models/label_encoder.joblib`
- `models/coral_source_stats.joblib`
- `models/coral_target_stats.joblib`

The `training_and_eval.ipynb` notebook trains on processed source train data,
evaluates on source test and target test, and reports target performance with and
without CORAL adaptation.

## Setup: Virtual Environment & Dependencies

To ensure a clean Python environment and install all required dependencies:

1. **Create a virtual environment (venv):**
   
   On Windows:
   ```sh
   python -m venv venv
   venv\Scripts\activate
   ```
   On macOS/Linux:
   ```sh
   python3 -m venv venv
   source venv/bin/activate
   ```
2. **Install dependencies:**
   ```sh
   pip install -r requirements.txt
   ```

This will install all necessary packages as specified in requirements.txt.

## Implemented CORAL Calculation Improvements

The current notebooks include three stability-focused upgrades to the CORAL computation path. Together, they make the adaptation step more numerically stable and less likely to over-correct under a large source-target domain gap.

1. Shrinkage covariance estimation (`LedoitWolf`) replaces plain sample covariance for both source and target covariance statistics.
   - Why this matters: in high-dimensional IDS data, plain covariance can be noisy or ill-conditioned.
   - Shrinkage improves conditioning, which makes matrix square-root and inverse square-root operations more reliable.

2. Spectral floor sweep for CORAL matrix power operations.
   - The inverse square-root and square-root steps in CORAL use an eigenvalue floor (`eps`) to avoid unstable behavior around very small or negative eigenvalues.
   - The notebook evaluates multiple floor strengths: `eps in [1e-4, 1e-3]`.

3. CORAL damping (partial adaptation) via convex blending.
   - Full CORAL can be too aggressive when domains are far apart, so the adapted features are blended with the original target features:
   - `X_adapted = X_target + lambda * (X_coral_full - X_target)`
   - The notebook evaluates `lambda in [0.1, 0.25, 0.5, 0.75, 1.0]`, where:
     - smaller lambda = lighter adaptation,
     - `lambda=1.0` = full CORAL adaptation.

### How the config sweep works in evaluation

The with-CORAL evaluation does not create one adapted target dataset. It creates multiple adapted versions of the same target test split and scores all of them.

1. For each `eps` value, a numerically stabilized CORAL transform is computed.
2. That transform is applied to the target test features to produce a full-CORAL version.
3. For each `lambda`, a damped version is generated from that full-CORAL output.
4. The classifier predicts on each generated version.
5. Accuracy is recorded for each `(eps, lambda)` pair, and the best-performing configuration is selected and reported.

With the current grid, this yields `2 x 5 = 10` evaluated target-test feature variants.

This sweep gives a practical way to choose a stable adaptation strength instead of assuming full CORAL is always optimal.
