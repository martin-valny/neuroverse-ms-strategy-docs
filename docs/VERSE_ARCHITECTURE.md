# Verse Architecture: Orbitals, Multiverse Warping, and Geodesics (Condition-Agnostic)

This document is the **canonical architecture reference** for any Verse-based project (MS-first, AD, immune, etc.), with domain-specific details living in separate docs. The architecture is condition-agnostic: "condition" is any label that partitions the data and may change cell-state structure.

---

## 1. Core Thesis

Verse is built around three ideas:

1. There is a **reference latent space** anchored to **healthy tissue**, where each cell type/state occupies a probabilistic "orbital" in a pathway/program space.
2. Each **condition** (disease, treatment, timepoint, niche, genotype, etc.) defines a **warp field** that bends this reference space, shifting and deforming orbitals relative to the healthy anchor.
3. **Trajectories** (e.g., disease progression, treatment response) can be represented as paths through warped spaces; deviations from expected paths encode resistance or atypical behavior.

### Reference chain (MS example)
```
Healthy tissue (fixed reference)
  └── NAWM           (mild warp w0: subtle pre-lesion shifts)
        ├── Rim       (spatial warp w_spatial + activity warp w_activity)
        └── Core      (stronger spatial warp + activity component)
                          └── Interactions where data support (≥3 donors/cell)
```

### Warp factorization
The total warp for any sample is a linear combination of orthogonal condition components. For MS:
- **w_spatial**: compartment effect (NAWM / rim / core), shared across lesion activity states
- **w_activity**: lesion activity effect (chronic-active vs inactive), shared across spatial positions
- **w_interaction**: spatial × activity interaction (e.g., chronic-active rim has a foamy-microglia/CD8-T niche not seen in inactive rim)
- **w_dataset**: dataset/protocol fixed effect, absorbed as a non-biological covariate

This factorization allows geodesics to be traced along either axis independently.

### Conservative modeling philosophy
v0 implements: probabilistic pseudobulk → two-channel programs → two-level PCA → low-rank + diagonal covariance orbitals → factorized linear warps.

The following are **v2+ extensions** — do not add these ad hoc in v0:
- Non-linear warp fields (kernel warps, neural warp fields)
- Full covariance orbitals
- Rotation and inter-orbital coupling terms
- Transformer imprint and geodesic trajectory analysis

---

## 2. Inputs

- Single-cell or single-nucleus expression data (optionally multi-omics).
- Per-cell metadata: sample/donor ID, cell family label, condition label (disease, niche, treatment, etc.), technical covariates (batch, protocol, chemistry).
- **Required metadata columns** (at pseudobulk level): `reference_flag` (bool, True for healthy controls), `spatial_label`, `activity_label`, `dataset_id`.

---

## 3. Layer 1 — Probabilistic Pseudobulk

### Concept
Aggregate single-cell/nucleus data by **Sample × CellFamily** into probabilistic pseudobulk. For each group and each gene: mean expression **µ**, standard deviation **σ**, number of cells **n**, confidence score.

### Uncertainty model (NB variance, not 1/√n)
Single-cell/nucleus RNA-seq has gene-specific overdispersion that simple Poisson models underestimate. Use:

```
σ²_gene = µ + µ² / r_gene
```

where **r_gene** is a gene-specific overdispersion parameter. Apply **DESeq2-style empirical Bayes dispersion shrinkage** to stabilize estimates for lowly expressed or sparse genes by sharing information across genes of similar expression level.

Confidence is then a function of both n and gene-specific overdispersion — not just sampling depth.

### Hard threshold
- **min_cells = 50**: groups with fewer than 50 cells are excluded from modeling. These groups receive a **MISSING** flag with **zero confidence** and are excluded from Layer 5 token construction.
- Groups with 50–150 cells are included but down-weighted proportionally by their confidence score.

### Why pseudobulk
Respects donor/replicate structure (prevents pseudoreplication), captures biological heterogeneity, and propagates uncertainty downstream to programs, orbitals, and warp fitting.

---

## 4. Layer 2 — Two-Channel Program Encoder

### Concept
Map gene-level pseudobulk distributions into a **two-channel program/pathway space**:

### Channel 1 — Curated programs (~180 total, interpretable)
Scored against pseudobulk µ/σ using gene-set scoring methods (e.g., UCell or ssGSEA on pseudobulk). Recommended v0 library:

| Source | Programs | Selected content |
|--------|----------|-----------------|
| Reactome CNS | ~50 | Myelination, neuroinflammation, complement cascade, synaptic signaling |
| MSigDB Hallmarks | ~50 | IFN-γ/α, hypoxia, oxidative phosphorylation, TNF/NF-κB, apoptosis, fatty acid metabolism |
| GO:BP modules | ~30 | Myelination, lipid/cholesterol metabolism, ROS, phagocytosis, complement activation |
| DoRothEA TF regulons | ~50 | NF-κB, STAT1, STAT3, HIF1A, PPARG, RUNX1, IRF regulons |

### Channel 2 — Residual/NMF programs (data-driven, novelty capture)
1. Regress each gene's pseudobulk expression on curated program scores (linear model); take residuals.
2. Apply **consensus NMF** to residuals per cell family:
   - Run NMF 20× with different random seeds on 80% subsamples.
   - Select K via **cophenetic correlation** (threshold ≥ 0.95); K floor = 5, K cap = 20. Default K = 10 per major cell family.
   - Extract consensus W (gene-program) and H (pseudobulk-program) matrices.
   - Threshold: retain top-50 genes per component (by W weight) for naming.
   - Name programs by pathway overlap (Fisher's exact, FDR-corrected).

### Preprocessing
- **Z-score** each program score (per program, across all pseudobulk units) before PCA to prevent high-variance programs from dominating.

### Two-level PCA
| Level | Scope | Purpose |
|-------|-------|---------|
| Global PCA | All cell families combined, 20–30 components | Cross-family visualization, coarse alignment, patient/sample embedding |
| Per-family PCA | Each cell family separately | Orbital fitting, warp estimation, novelty scoring |

Per-family PCA prevents abundant families (microglia, astrocytes) from dominating the embedding at the expense of sparse families (T cells, endothelial).

### Back-projection (interpretability recovery)
After fitting in per-family PCA space, project warp shift vectors Δ_PCA back to the original program space:
```
Δ_program = W_family^T × Δ_PCA
```
This recovers named program contributions (e.g., "IFN-γ and lipid metabolism programs drive the rim warp in microglia").

### Shift vs identity
Always report **within-cell-type condition contrasts** (Δ_program vectors and warp magnitude) — not absolute pathway levels. Identity programs (e.g., "microglia-ness") are stable within a cell family and are not the target signal.

---

## 5. Layer 3 — Orbital Layer (L1/L2 Hierarchy)

### Hierarchy
- **L1 orbitals** = cell family identity, defined from healthy reference tissue. Stable across conditions.
- **L2 orbitals** = canonical states within each cell family. Warp fields operate at this level.

### Covariance model: low-rank + diagonal
Orbitals are modeled as **low-rank + diagonal covariance Gaussians** (factor analysis style):
```
Σ_orbital = Λ Λ^T + Ψ
```
where Λ is a low-rank matrix capturing correlated program directions (e.g., IFN-high × lipid-high in foamy microglia), and Ψ is diagonal noise. This is more honest than strict diagonal covariance for biologically correlated states.

**Fallback to strict diagonal**: if a cell family/state has fewer than 3 × n_PCA_components pseudobulk groups, fall back to diagonal covariance.

### Orbital initialization: anchor table (all families)

| Cell family | L2 canonical states | Anchor source |
|-------------|--------------------|--------------------|
| Microglia | homeostatic, DAM-like, IFN-responsive, foamy/ABCA1-G1-low, iron-accumulating | Published signatures; GSE301908 foamy/CD8 niche |
| Astrocytes | homeostatic, A1 neuroinflammatory, A2 neuroprotective, pan-reactive, scar-associated | A1/A2 literature; Lerma-Martin 2024 |
| OL-lineage | myelinating OL, pre-myelinating OL, stress/injured OL | Lerma-Martin 2024; Absinta et al. |
| OPC | quiescent OPC, activated OPC (PPARG+, differentiation-impaired) | MS lesion literature |
| T cells | CD8+ Trm (rim niche), CD4+ Th17-like, Treg | GSE301908 CD8 rim niche |
| Neurons | baseline (minimal warp expected — **negative control**) | Healthy reference |
| Endothelial | homeostatic, activated/inflamed | General vascular literature |
| Any other family | State-1 … State-N (NMF K=3–5 from healthy tissue) | Data-driven fallback |

Initialize L2 orbitals from healthy-reference donors. **Anchoring criterion**: ≥ 80% of healthy-reference cells must fall inside reference orbitals at 99% confidence before proceeding.

### Novelty scoring
```
novelty_score(x) = 1 − max_k P(x | orbital_k)
```
where P(x | orbital_k) uses the **combined variance** σ²_sample + σ²_orbital (uncertainty from the observed pseudobulk group + uncertainty of the orbital estimate).

**Two novelty bands**:
| Band | Threshold | Interpretation |
|------|-----------|----------------|
| Extreme known state | Outside 95% but inside 99% of best orbital | "Extreme activation of [state]" — report as tail, not novel |
| Truly novel | Outside 99% of all orbitals | Promote only if replicated across ≥2 donors |

**Guardrails against over-calling novelty**:
1. Groups with confidence < 0.5 excluded; 0.5–0.8 down-weighted. Low-confidence groups cannot generate novelty calls.
2. Families with < 100 pseudobulk groups across the dataset are flagged **"uncertainty-limited"**: novelty scores reported but not promoted.
3. **In vitro / query domains** (e.g., iPSC microglia): apply trajectory distance toward target tissue orbitals instead of absolute novelty scoring. iPSC microglia lack yolk-sac lineage imprinting and will score as high novelty for non-biological reasons.
4. **Technical negative controls**: always run batch/protocol as pseudo-conditions through the same novelty pipeline. Novelty calls driven by technical factors → revise integration.

---

## 6. Layer 4 — Factorized Multiverse Warping

### Reference
**Healthy tissue** (reference_flag = True) defines fixed orbital means and covariances. These are held fixed during warp fitting.

### Warp model (per cell family, per-family PCA space)
```
center_condition = center_healthy + w_spatial·X_spatial + w_activity·X_activity
                  + w_interaction·(X_spatial ⊗ X_activity) + w_dataset·X_dataset
```

| Parameter | Role |
|-----------|------|
| w_spatial | Spatial compartment effect (NAWM / rim / core) |
| w_activity | Lesion activity effect (chronic-active / inactive) |
| w_interaction | Spatial × activity (fit only if ≥3 donors per condition cell) |
| w_dataset | Dataset/protocol fixed effect (non-biological covariate) |
| sex, disease_duration | Additional demographic covariates |

### Collinearity check (required before fitting)
1. Build the full spatial × activity × dataset_id cross-table.
2. Compute the design matrix condition number. If effectively rank-1 (condition number > 30 or near-singular), spatial and activity axes cannot be separated in this dataset.
3. **Collapse** to a single **probabilistic lesion axis** (one composite warp vector for the dominant disease gradient). Document this explicitly as a limitation — it does not invalidate the model, but does limit mechanistic claims.

### Interaction sparsity rule
Fit w_interaction only if ≥ 3 donors exist per condition cell (each spatial × activity combination). Otherwise drop the interaction term and log all skipped cells.

### L2 regularization
Warp fitting minimizes:
```
L = reconstruction_error + λ · ||w||²
```
Select λ by leave-one-donor-out cross-validation.

### Warp magnitude (exact definition)
```
warp_magnitude = ||Δ_PCA||₂   (L2 norm of orbital center translation in per-family PCA space)
```
Scale changes (covariance shifts) are reported separately as percentage change in orbital variance.

### Back-projection + FDR
1. Project Δ_PCA via W_family^T to recover named program contributions Δ_program.
2. Rank programs by |Δ_program| (effect size).
3. Apply **Benjamini-Hochberg FDR correction** across all programs when labeling programs as significantly warp-driving.

### Batch constraint (validation)
Batch/protocol warp magnitude must be **< 50% of the biological (spatial or activity) warp magnitude** for the same cell family. Failure → revise integration before interpreting warps.

---

## 7. Layer 5 — Token Construction

Tokens combine per Sample × CellFamily: program scores (Layer 2), orbital membership + novelty (Layer 3), warped orbital positions (Layer 4), uncertainty and metadata.

**Confidence-based weighting**:
| Confidence | Token treatment |
|------------|----------------|
| < 0.5 | Excluded from token set |
| 0.5–0.8 | Weight = confidence (downweighted) |
| > 0.8 | Full weight = 1.0 |

---

## 8. Layer 6 — Cell Type Transformer (v2+)

Apply a transformer over the token set for each sample/patient. Attention over tokens learns which cell states matter for the condition or outcome. Outputs: patient "imprint" embedding + attention weights as interpretable cell-type importance.

*Deferred to v2+. Only implement if required for a specific near-term task.*

---

## 9. Layer 7 — Imprint Space & Multi-Task Heads (v2+)

Patient imprint used for contrastive objectives (condition A vs B, responder vs non-responder) and supervised prediction heads. *Deferred to v2+.*

---

## 10. Geodesics & Discovery Modules (v2+)

**Geodesic Analyzer**: expected trajectories in warped space between conditions; measures deviation of actual movement from geodesic.

**Warp Field Analyzer**: which orbitals are most warped; which programs drive shifts; human-readable mechanism summaries.

**Discovery tools**: novel state detection; biomarker discovery from warp-driven program changes.

*All deferred to v2+. Warps in v0 are already a mechanistic readout without geodesics.*

---

## 11. Batch Correction and Integration Strategy

### Cell level
Apply **Harmony** (or scVI) on the cell embedding before clustering and cell type annotation. Ensures cell family assignments are not confounded by chemistry (10X v2 vs v3, fresh-frozen vs fixed).

### Pseudobulk/program level
**Do not batch-correct pseudobulk expression matrices.** Include `dataset_id` and `protocol` as **fixed-effect covariates** in:
- Pseudobulk → program scoring
- Warp fitting design matrix (as w_dataset)

Rationale: batch-correcting pseudobulk silently removes signal when batch is confounded with biological condition. Retaining batch as a covariate lets the model estimate and separate technical from biological effects, and lets you detect when batch drives warps.

### Composition confound check
Before fitting warps, verify that cell-type proportions are not significantly predicted by `dataset_id` in a way that reflects processing artifacts. Run permutation tests per cell family. Flag families where proportion is significantly predicted by dataset_id.

---

## 12. Evaluation (Quantitative)

These are **aspirational v0 targets**. If a metric falls below threshold, log it with root-cause analysis — small n in MS datasets makes some targets aspirational rather than hard gates.

| Metric | Target | Per family? |
|--------|--------|------------|
| LODO: Spearman ρ of warp direction vectors | ≥ 0.7 | Yes |
| LOSO: Jaccard of top-10 warp-driving programs | ≥ 0.5 | Yes |
| Batch warp magnitude / biological warp magnitude | < 50% | Yes |
| Healthy-reference cells inside orbitals at 99% CI | ≥ 80% | Yes |
| Neurons warp magnitude / microglia warp magnitude | < 25% | Global |

Evaluate per cell family. Interpret relative to sampling depth — microglia warps will be more stable than T-cell warps given sparse T-cell sampling in white matter tissue.

---

## 13. Cross-Reference

| Topic | See also |
|-------|----------|
| Operational implementation steps | `ENGINEERING_CHECKLIST_MS_FIRST.md` |
| Architecture → checklist mapping | `ARCHITECTURE_IMPLEMENTATION_MAP.md` |
| MS-specific scientific strategy | `STRATEGY_MEMO_MS_FIRST.md` |
| Visual pipeline diagram | `VERSE_ARCHITECTURE_DIAGRAM.md` |
| One-page partner summary | `VERSE_FOR_MS_ONE_PAGE.md` |
