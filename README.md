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
