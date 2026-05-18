# Domain Adaptation (CORAL) Proof of Concept

A controlled, offline experiment to verify the feasibility and effectiveness of the prospective domain adaptation feature.

## Overview

Feature space and label space alignment across source and target datasets must be done for CORAL domain adaptation. Label space alignment involves collapsing each dataset's specific attack classes into broader meta-categories representing a shared label space.

In the app, the implemented DA feature will consist of a target-data collection mechanism and will perform preprocessing and covariance calculation in the online app (as opposed to in an offline notebook environment). In this offline test, all procedures and computation will be performed in a notebook environment; the target-domain data (CSE-CIC-IDS2018) will be processed and feature-mapped in advance alongside the source-domain data (CICIDS2017), and CORAL calculation, model inference, and results extraction will be done in the notebooks as well.

## New Version Info

This iteration replaces the previous target dataset with CSE-CIC-IDS2018 to address the severe domain gap between CICIDS2017 and CIC_ToN_IoT which caused poor results.

## Datasets

| Role | Dataset |
|------|---------|
| Source | CICIDS2017 |
| Target | CSE-CIC-IDS2018 |

## Shared Artifacts

The CORAL framework assumes source and target datasets are represented in the same feature and label spaces before adaptation is applied. To enforce this consistently, shared external contract files are used so both pipelines consume the same canonical definitions.

This project uses shared contract artifacts under `data/processed/`:

- `shared_feature_space.json`: canonical ordered list of shared features.
- `shared_label_space.json`: canonical ordered list of shared labels.
- `source_label_map.json`: raw source labels --> shared labels
- `target_label_map.json`: raw target labels --> shared labels
- `target_feature_map.json`: raw target features --> shared features

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

1. **Prepare CICIDS2017 and CSE-CIC-IDS2018**
   - Apply same preprocessing and scaling
   - Normalize feature space
   - Extract covariance statistics
2. **Train model on CICIDS2017**
3. **Calculate CORAL transform**
4. **Perform model inference on CSE-CIC-IDS2018**
   - With CORAL
   - Without CORAL
5. **Compare performance** using ground truth labels

## Preprocessing Notebook Steps

The tables below reflect the current cell-title steps from both preprocessing notebooks.

| src_preprocessing.ipynb | trg_preprocessing.ipynb |
|---|---|
| 1. Imports | 1. Imports |
| 2. Load and concatenate CSVs | 2. Load and concatenate CSVs |
| 3. Data sanitization | 3. Data reduction |
| 4. Feature-space alignment | 4. Data sanitization |
| 5. Label-space alignment | 5. Feature-space alignment |
| 6. Train/Test Split (80/20) | 6. Label-space alignment |
| 7. Scaling (fit to source and save) | 7. Train/Test Split (80/20) |
| 8. Label encoding (fit to source and save) | 8. Scaling (reuse source-fitted scaler) |
| 9. Calculate and export covariance and mean statistics | 9. Label encoding (reuse source-fitted encoder) |
| 10. Export processed data | 10. Calculate and export covariance and mean statistics |
|  | 11. Export processed data |

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
