# Architectural and Graph Effects in Multi-Omics Autoencoders
Code base for the NetMAHIB paper: Disentangling Architectural and Graph Effects in Multi-Omics Autoencoders: A Controlled Ablation and Null-Graph Study in Breast Cancer

This repository contains the Jupyter notebooks used to train and evaluate every
model and control condition reported in the paper: the Baseline and Denoising
Autoencoders, the Graph-Regularised DAE (GR-DAE) on the real PPI graph, two
null-graph negative controls, two architectural ablations, a PCA baseline, and
the k-robustness sweeps.

## Paper summary

We perform a controlled comparison of autoencoder variants for unsupervised
multi-omics patient representation learning on the TCGA-BRCA cohort, and test
whether the stability and clinical-association gains commonly attributed to
PPI-based graph regularisation in this literature are specific to the
biological network used. Using null-graph controls and architectural
ablations not previously applied in this area, we find that these gains are
substantially explained by weight-tying in the encoder architecture and by
generic graph-Laplacian smoothing, and that a widely reportable clinical
association reflects a cohort-annotation artefact rather than genuine
metastatic biology.

## Repository structure

```
.
├── README.md
└── notebooks/
    ├── 01_baseline_autoencoder.ipynb
    ├── 02_denoising_autoencoder.ipynb
    ├── 03_grdae_real_graph_main.ipynb
    ├── 04_grdae_lambda0_and_mstatus_attribution.ipynb
    ├── 05_null_graph_controls.ipynb
    ├── 06_collapse_diagnostics_and_pca_baseline.ipynb
    ├── 07_ablation_untied_encoder.ipynb
    ├── 08_ablation_weight_space_penalty.ipynb
    ├── 09_ksweep_dae.ipynb
    └── 10_ksweep_grdae.ipynb
```

## Data access

Raw data is **not redistributed** in this repository, consistent with TCGA
and STRING's own data use terms.

| Source | Access | Notes |
|---|---|---|
| TCGA-BRCA multi-omics (RNA-seq, CNV, methylation, mutation) | [GDC Data Portal](https://portal.gdc.cancer.gov) | Accessed 01 February 2026 |
| STRING PPI network | [string-db.org](https://string-db.org) | v11.0, combined confidence score ≥ 0.85 |
| Clinical covariates and PAM50 labels | TCGA clinical portal | Used only for post-hoc evaluation, never for training |

The notebooks load two pre-processed datasets hosted on Kaggle. 

| Dataset | Kaggle link | Used by | Derived from |
|---|---|---|---|
| TCGA-BRCA multi-omics (gene-level RNA-seq, CNV, methylation and mutation matrices, with associated clinical data) | [tcga-brca-multiomics](https://www.kaggle.com/datasets/habibaelkerabysqu/tcga-brca-multiomics) | Notebooks 01–10 | GDC Data Portal, accessed 01 February 2026 |
| STRING PPI edges mapped to HGNC gene symbols | [stringdb-preprocessed](https://www.kaggle.com/datasets/habibaelkerabysqu/stringdb-preprocessed) | Notebooks 03–08 and 10 | STRING v11.0, combined confidence score ≥ 0.85 |

Preprocessing steps (gene-universe definition, normalisation, methylation
imputation, STRING mapping and thresholding) are described in Section 2 of the
paper.

**Paths.** Notebooks 01 and 02 mount the omics data at
`/kaggle/input/tcga-brca-multiomics`, while the later notebooks use
`/kaggle/input/datasets/<owner>/<dataset>/`. These refer to the same
dataset. If you run the notebooks outside Kaggle, set the data directory in the
configuration cell at the top of each notebook.

## Environment

Core dependencies: `torch`, `numpy`, `pandas`, `scipy`, `scikit-learn`,
`statsmodels`, `networkx`, `matplotlib`. 

**Note on paths:** these notebooks were developed and run on Kaggle, and
contain `/kaggle/input/...` paths for data loading. If running elsewhere,
update these paths to point at your own local copies of the datasets above.

## Notebooks

Notebooks 01–03 are the three models from the original submission. Notebooks
04–10 were added during the revision. Several of them reuse the data loading,
graph construction, and evaluation code from notebook 03, so they share its
structure.

### 01 — Baseline Autoencoder (`01_baseline_autoencoder.ipynb`)
Unstructured reference model. A flat MLP (52,004 → 1,024 → 256 → 64) with no
input corruption and no graph term. Covers a single reference training run,
convergence statistics, k-means clustering quality across k = 2–8, and the
five-seed restart-stability experiment. It serves mainly as a reference point;
the DAE is the primary unstructured baseline in the paper.

### 02 — Denoising Autoencoder (`02_denoising_autoencoder.ipynb`)
The same flat architecture as the BAE, with Gaussian input corruption
(σ = 0.3). This is the main baseline against which the other models are
compared. Covers the reference run, restart stability, latent geometry,
and clinical and PAM50 association analyses.

### 03 — GR-DAE, real PPI graph (`03_grdae_real_graph_main.ipynb`)
The Graph-Regularised DAE: a shared-weight gene-wise encoder, mean pooling
over genes, and a Laplacian smoothness penalty on the STRING PPI graph
(λ = 10⁻⁶). Includes graph loading and normalised-Laplacian construction,
the reference run with early stopping, the restart-stability experiment,
geometry and PCA analyses, cluster-level and per-dimension clinical
association, and PAM50 alignment. Other notebooks build on this one.

### 04 — GR-DAE with λ = 0 and M-status attribution (`04_grdae_lambda0_and_mstatus_attribution.ipynb`)
The architecture-only control: the same GR-DAE with the Laplacian term
switched off. Reruns the restart experiment at λ = 0, applies the clinical
association tests to both λ = 0 and the selected model, and plots the
penalty-to-reconstruction ratio across training epochs. Also contains the
metastatic-status confound analysis: the full cluster × M-status contingency
table, a permutation test for M0 vs. M1 with MX patients excluded, and the
per-dimension re-analysis with MX treated as missing.

### 05 — Null-graph controls (`05_null_graph_controls.ipynb`)
GR-DAE trained on a null graph in place of the real STRING graph. A
`GRAPH_MODE` setting at the top of the notebook selects one of three
conditions, and the notebook is run once per setting:
`"real"`, `"degree_preserving_null"` (Maslov–Sneppen double-edge swaps), or
`"erdos_renyi_null"` (random G(n, m) graph matched on edge count). Real STRING
edge weights are randomly permuted onto the null edges. Training and
evaluation are otherwise identical across conditions (λ = 10⁻⁶, five seeds).
The held-out reconstruction and neighbourhood-preservation diagnostics are
included in the same notebook.

### 06 — Collapse diagnostics and PCA baseline (`06_collapse_diagnostics_and_pca_baseline.ipynb`)
Real-graph reference run extended with held-out evaluation: per-modality
reconstruction R², trustworthiness and continuity on the validation split, and
within-patient gene-embedding variance. Also implements the PCA baseline
(3 and 64 components): held-out reconstruction, full-cohort clustering, and
clinical and PAM50 association.

### 07 — Untied gene-wise encoder (`07_ablation_untied_encoder.ipynb`)
Architectural ablation that separates weight-tying from mean pooling. Each
gene gets its own independent encoder and decoder weights (about 3.4 × 10⁸
parameters), implemented with batched matrix multiplication. Pooling and the
Laplacian term are identical to GR-DAE. A `TIE_WEIGHTS` setting switches between
the untied model (`False`) and the standard shared-weight GR-DAE (`True`), which
is useful as a sanity check. Includes explicit GPU-memory cleanup between seeds.

### 08 — Weight-space Laplacian penalty on the flat DAE (`08_ablation_weight_space_penalty.ipynb`)
Applies a Laplacian penalty to a flat DAE with no architectural change. Because
the flat model has no per-patient gene-level activation, the penalty acts on the
first-layer weight matrix instead. Its columns are reshaped by gene and modality
and averaged across modalities to give a gene-indexed matrix. The penalty
coefficient is calibrated from the penalty magnitude at initialisation. Covers
the restart experiment, geometry stability, and a k-sweep. This notebook was
built from a truncated copy of notebook 03, so it rebuilds the scaled input
from `X_surv` rather than reusing the gene tensor.

### 09 — k-robustness sweep, DAE (`09_ksweep_dae.ipynb`)
Saves the embeddings from all five restart seeds, then recomputes restart
ARI/NMI across k = 2–8 by re-clustering them. No retraining is needed, because
k affects only the post-hoc clustering step.

### 10 — k-robustness sweep, GR-DAE (`10_ksweep_grdae.ipynb`)
The same k-sweep procedure applied to the real-graph GR-DAE.

## How the notebooks were used

These notebooks are parameterised templates, not one-shot scripts. Each
experimental condition in the paper was produced by setting the configuration
values at the top of a notebook (for example `GRAPH_MODE`, `TIE_WEIGHTS`,
`lambda_graph`, or `k`), running the full notebook, and saving the result. Where
one notebook covers several conditions, it was run once per setting.

The outputs stored in a notebook therefore reflect whichever configuration it
was last run with, and are not necessarily the ones reported in the paper. The
configuration for each reported condition is given in the notebook descriptions
below. To reproduce a result, set the configuration accordingly and run the
notebook from top to bottom.

## Contact
Dr. Hamza Zidoum — zidoum@squ.edu.om

Habiba El Keraby — s147635@student.squ.edu.om
