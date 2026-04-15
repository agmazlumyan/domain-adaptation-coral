Domain adaptation (CORAL) proof of concept project

A controlled, offline experiment to verify the feasibility and effectiveness
of the prospective domain adaptation feature. 

Feature space and label space alignment across source and target datasets must
be done for CORAL domain adaptation. Label space alignment involves collapsing
each dataset's specific attack classes into broader meta-categories representing
a shared label space.

In the app, the implemented DA feature will consist of a target-data collection
mechanism and will perform preprocessing and covariance calculation in the online
app (as opposed to in an offline ipynb env). In this offline test, all procedures
computation will be performed in a ipynb environment; the target-domain data
(CIC_ToN_IoT) will be processed and feature-mapped in advance alongside the source-
domain data (CICIDS2017), and CORAL calculation, model inference, and results 
extraction will be done in ipynb as well.

Source: CICIDS2017
Target: CIC_ToN_IoT

Steps:
- Prepare CICIDS2017 and CIC_ToN_IoT
    - same preprocessing and scaling
    - normalize feature space
    - extract covariance statistics
- Train model on CICIDS2017
- Calculate CORAL transform
- perform model inference on CIC_ToN_IoT
    - with CORAL
    - without CORAL
- compare performance (use ground truth labels)
