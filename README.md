# Broad Institute Obesity Challenge — CrunchDAO (2026)

> Predicting single-cell RNA-seq transcriptomic responses to unseen gene perturbations in human adipose tissue.

**Competition:** [Broad Obesity Challenge Crunch 1](https://hub.crunchdao.com/competitions/broad-obesity-1)  
**Platform:** CrunchDAO  
**Alias:** noble-usagi 
**Predictive Model Alias:** vanga-wakamanya  
**Submission date:** February 28, 2026  

---

## Results

| Metric | Rank | Score |
|---|---|---|
| L1 Distance | **23rd** | 0.109 |
| Maximum Mean Discrepancy (MMD) | 39th | 1.338 |
| Pearson Delta | 73rd | 0.063 |

2 out of 2 runs successful. Valid submission on first cloud run.

---

## Problem

Given single-cell RNA-seq data from 123 training gene perturbation experiments (CRISPR-Cas9 knockdowns) in human pre-adipocyte cells, predict the transcriptomic response for 2,863 **unseen** gene perturbations.

**Output required:**
- `prediction.h5ad` — 286,300 cells x 10,238 genes (100 synthetic cells per perturbation)
- `predict_program_proportion.csv` — cell-state proportions (pre_adipo, adipo, lipo, other) per perturbation

**Evaluation metric:** Pearson Delta — the Pearson correlation between predicted and true perturbation effects (relative to a perturbed centroid baseline), averaged across perturbations.

---

## Approach

### Why not deep learning?

The dataset has approximately 122 labelled perturbations against an 11,046-dimensional gene expression output space. In this extreme high-dimensional / low-sample regime, VAEs and MLPs massively overfit — we validated this empirically, with both architectures producing near-zero validation Pearson Delta despite strong training performance. Classical regularised regression is the right tool here.

### Pipeline

```
Training data (44,846 cells, 123 conditions)
        |
        v
Pseudo-bulk aggregation
(mean expression per perturbation -> 123 x 11,046 matrix)
        |
        v
PCA (50 components, 78.9% variance explained)
(123 x 11,046 -> 123 x 50 fingerprints)
        |
        v
Ridge Regression (alpha=10)
(maps 50-dim PCA coords -> 11,046-dim gene expression)
        |
        v
Inference on 2,863 unseen perturbations
(knockdown proxy: set target gene = 0 in control profile, project through PCA)
        |
        v
Synthetic cell expansion
(100 cells per perturbation via gene-wise Gaussian noise)
```

### Key design decisions

**Pseudo-bulk aggregation** reduces the noisy single-cell data to a stable perturbation-level signal before modelling, avoiding overfitting to cell-level stochasticity.

**PCA before Ridge** compresses the 11,046-gene input space to 50 principal components, further regularising the problem and ensuring the Ridge regression operates in a tractable subspace.

**Knockdown proxy for unseen genes** — for each of the 2,863 unseen target genes, we simulate the perturbation by taking the control (NC) mean expression profile and zeroing the target gene's expression, then projecting through the fitted PCA. This is mechanistically motivated: CRISPR-Cas9 knockdowns deplete the target gene.

**Noise injection** — the predicted perturbation mean is expanded to 100 cells by adding gene-wise Gaussian noise (sigma estimated from cross-perturbation variability in training). This ensures the output distribution has realistic spread rather than being a degenerate point mass, which matters for the MMD metric.

**Program proportions** — cell-state proportions (pre_adipo, adipo, lipo, other) are predicted via a second Ridge model in PCA space, with post-processing to enforce sum-to-1 and lipo <= adipo constraints.

---

## What worked and what did not

| Approach | Result |
|---|---|
| Ridge regression + PCA (submitted) | L1 rank 23, valid submission |
| MLP / neural network | Near-zero validation Pearson Delta, overfit |
| VAE | Near-zero validation Pearson Delta, overfit |
| Knockdown proxy for unseen genes | Weak Pearson signal (low PCA shift per gene) |

The knockdown proxy produced very small PCA embedding shifts for most unseen genes (max shift ~0.002-0.16), meaning the model predicted near-identical expression for most unseen perturbations. This is the primary reason Pearson Delta rank is lower than L1 rank. A meaningful improvement would require gene co-expression network priors or pre-trained biological embeddings (scGPT, Geneformer) to build richer representations for unseen genes.

---

## Repository structure

```
.
├── ridge_submission_v2.ipynb    # Full submission notebook
├── README.md                    # This file
└── results/
    └── leaderboard_screenshot.png
```

---

## Data

Data is provided by the Broad Institute via CrunchDAO and is not included in this repository. To reproduce:

1. Register at https://hub.crunchdao.com
2. Join the Broad Obesity Challenge
3. Run Cell 1 of the notebook with your token to download data

---

## Requirements

```
crunch-cli
anndata
scanpy
scikit-learn
numpy
pandas
scipy
h5py
psutil
tqdm
```

---

## Relevance to precision oncology

This competition is directly relevant to precision medicine research. The core challenge — predicting cellular responses to genetic interventions from limited training data in high-dimensional biological spaces — mirrors key problems in cancer genomics: predicting drug response, modelling gene regulatory networks, and understanding tumour heterogeneity at single-cell resolution. The regularisation strategies and dimensionality reduction approaches developed here are transferable to oncology contexts where labelled patient data is scarce.

---

## License

Code is released under MIT License. Competition data belongs to the Broad Institute / CrunchDAO.
