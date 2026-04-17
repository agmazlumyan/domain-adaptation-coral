# Domain Adaptation (CORAL) Proof of Concept

A controlled, offline experiment to verify the feasibility and effectiveness of the prospective domain adaptation feature.

## Overview

Feature space and label space alignment across source and target datasets must be done for CORAL domain adaptation. Label space alignment involves collapsing each dataset's specific attack classes into broader meta-categories representing a shared label space.

In the app, the implemented DA feature will consist of a target-data collection mechanism and will perform preprocessing and covariance calculation in the online app (as opposed to in an offline notebook environment). In this offline test, all procedures and computation will be performed in a notebook environment; the target-domain data (CIC-ToN-IoT) will be processed and feature-mapped in advance alongside the source-domain data (CICIDS2017), and CORAL calculation, model inference, and results extraction will be done in the notebooks as well.

## Datasets

| Role | Dataset |
|------|---------|
| Source | CICIDS2017 |
| Target | CIC-ToN-IoT |

## Shared Artifacts

The CORAL framework assumes source and target datasets are represented in the same feature and label spaces before adaptation is applied. To enforce this consistently, shared external contract files are used so both pipelines consume the same canonical definitions.

This project uses a fixed external feature-space contract at
`data/processed/shared_feature_space.json` to prevents source/target schema drift.

For label-space alignment, the project uses shared label artifacts:
- `data/processed/shared_label_space.json`: canonical ordered list of common labels.
- `data/processed/source_label_map.json`: maps source raw classes to canonical labels.
- `data/processed/target_label_map.json`: maps target raw classes to canonical labels.

Both datasets are mapped into the same canonical label space before encoding so label
integers stay consistent across source/target training and evaluation.

- The JSON file contains the canonical ordered list of shared features.
- Both preprocessing notebooks consume this file directly.
- Feature alignment is strict by default:
   - Columns not in the contract are dropped.
   - Missing contract columns raise an error.
- Preserving column order is required so reused scalers and downstream models
   receive the same input schema across source and target domains.

## Steps

1. **Prepare CICIDS2017 and CIC-ToN-IoT**
   - Apply same preprocessing and scaling
   - Normalize feature space
   - Extract covariance statistics
2. **Train model on CICIDS2017**
3. **Calculate CORAL transform**
4. **Perform model inference on CIC-ToN-IoT**
   - With CORAL
   - Without CORAL
5. **Compare performance** using ground truth labels

## Preprocessing Notebook Steps

The tables below reflect the current cell-title steps from both preprocessing notebooks.

| src_preprocessing.ipynb | trg_preprocessing.ipynb |
|---|---|
| 1. Imports | 1. Imports |
| 2. Import and concatenate CSVs | 2. Import CSV |
| 3. Data sanitization | 3. Data sanitization |
| 4. Feature-space alignment | 4. Feature-space alignment (align features according to predetermined shared feature space) |
| 5. Label-space alignment | 5. Label-space alignment (align labels according to predetermined shared label space) |
| 6. Train/Val/Test Split | 6. Scaling (reuse scaler of source dataset) |
| 7. Scaling (save scaler for CIC_ToN_IoT) | 7. Label encoding (reuse encoder of shared label space) |
| 8. Label encoding (save encoder for CIC_ToN_IoT) | 8. Calculate and export covariance and mean statistics |
| 9. Calculate and export covariance and mean statistics | 9. Export processed data |
| 10. Export processed data |  |

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
