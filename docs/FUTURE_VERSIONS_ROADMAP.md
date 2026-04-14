# NeuroVerse Future Versions Roadmap

This document captures all planned additions beyond v0, with enough detail to start implementation. It is organized by version tier. Items are derived from: (a) v0 known limitations documented in `VERSE_ARCHITECTURE.md`, and (b) conceptual gaps identified against the original ImmuneVerse specification.

Cross-references: `VERSE_ARCHITECTURE.md` (canonical spec) · `ENGINEERING_CHECKLIST_MS_FIRST.md` (v0 implementation steps) · `ARCHITECTURE_IMPLEMENTATION_MAP.md` (consistency map)

---

## v0.5 — No Transformer Required

### Warp Interpolation (trajectory visualization)

**Status**: Spec complete. Add to pipeline after v0 sample imprints are working.

**What it is**: given two samples A and B with fitted imprint vectors, linear interpolation traces a path between their ecosystem states in warp space:
```
interp(t) = (1 − t) · imprint(A) + t · imprint(B),   t ∈ [0, 1]
```

**Use cases**:
- Visualize the MS disease progression axis: interpolate from healthy donor imprint → NAWM → rim → core. Each step shows how the cellular ecosystem changes along that gradient.
- Show treatment response trajectory: shift from a disease-state imprint toward the healthy orbital center.
- Generate synthetic intermediate states for presentations.

**Implementation notes**:
- Compute imprint vectors per sample as described in `VERSE_ARCHITECTURE.md §13` (concatenated per-family warp vectors).
- Interpolation is a post-processing step on the fitted imprint vectors — no model changes needed.
- For each interpolated point `interp(t)`, split back into per-family blocks (same ordering as imprint concatenation) and project back to per-family program space via `W_family` for interpretation.
- Output: a DataFrame of interpolated points at t = 0, 0.1, 0.2, ... 1.0, with per-family block projections and a visualization notebook.

**Caveats**: interpolation is a visualization tool, not a quantitative inference procedure. Do not use interpolated points as inputs to orbital membership or warp estimation. Valid only between samples with imprints in the same reference orbital coordinate system (same cell family set, same PCA basis).

---

## v1 Additions

### σ_program Propagation via Delta Method

**Status**: Spec complete. Requires no new data — Layer 1 already outputs `(µ_gene, σ²_gene)`.

**Problem being solved**: v0 program scores are computed from pseudobulk µ only (UCell/ssGSEA). Gene-level uncertainty (σ²_gene from NB variance in Layer 1) is not propagated into program scores. Novelty scoring therefore uses only σ²_orbital, underestimating total uncertainty. Programs in sparse pseudobulk groups (small n, high overdispersion) appear as confident as programs in dense groups.

**Formula (delta method, first-order approximation)**:

For any program P scored as a weighted sum of gene expression:
```
score_P = Σ_g  w_g · µ_gene_g

σ²_program_P = Σ_g  w²_g · σ²_gene_g

σ_program_P = sqrt(σ²_program_P)
```

Where:
- `w_g` = UCell/ssGSEA gene weight for gene g in program P
- `µ_gene_g` = pseudobulk mean expression for gene g (Layer 1 output)
- `σ²_gene_g` = NB variance for gene g: `µ + µ²/r_gene` (Layer 1 output)

**Pipeline changes in v1 (three additions)**:

1. **Layer 2 — program scoring**: compute `σ²_program_P` alongside `score_P` for each program P and each pseudobulk group. Store in `.uns["program_score_variances"]`.

2. **Layer 3 — novelty scoring**: update the Gaussian likelihood computation to use combined variance:
   ```
   σ²_total = σ²_orbital + σ²_program    (combined uncertainty)
   ```
   Replace `σ²_orbital only` in the P(x | orbital_k) computation with `σ²_total`.

3. **Warp report output**: add per-program uncertainty bands (`score_P ± 1.96 · σ_program_P`) to `warp_report.csv` for each top-10 program per warp axis.

**Alternative implementation — Monte Carlo sampling**:
For non-linear scorers (e.g., ssGSEA with rank-based scoring), the delta method is approximate. As an alternative:
1. For each pseudobulk group, draw N=200 gene expression vectors by sampling independently from `NB(µ_gene_g, r_gene)` for each gene.
2. Score each of the 200 draws through the full UCell/ssGSEA pipeline.
3. Compute empirical standard deviation across the 200 scored vectors → `σ_program_P`.

This is ~200× slower than the delta method but more accurate for non-linear scorers. Recommended only if computational resources allow; delta method is the default.

**Dependency**: requires `r_gene` (NB dispersion parameter) per gene to be stored from Layer 1. Verify this is in the v0 output before implementing v1.

---

### CNS Cell-State Signature Library

**Status**: Gene sets need to be curated from published marker tables. No new data required.

**Problem being solved**: the v0 curated library (Reactome, Hallmarks, GO:BP, DoRothEA, PROGENy) covers general pathways but does not capture MS/CNS cell-state-specific programs. Foamy microglia (ABCA1↓/PLIN2), iron-accumulating microglia (FTH1/SLC40A1), and CD8+ Trm niche programs are not represented. These are the most scientifically interesting signals in the MS application.

**Planned v1 signature table** (~20–30 programs):

| Signature | Source | Key genes | Cell family |
|-----------|--------|-----------|-------------|
| Foamy microglia / lipid-accumulating | GSE301908 (Immunity 2025); Absinta/Maric | ABCA1↓, ABCG1↓, PLIN2, LIPA, CD68, TREM2↑ | Microglia |
| Iron-accumulating microglia | Absinta/Maric GSE180759; MS literature | FTH1, FTLL1, SLC40A1, HMOX1 | Microglia |
| IFN-responsive microglia | Absinta; Lerma-Martin | ISG15, MX1, IFIT1, IFIT2, IFIT3, OAS1 | Microglia |
| DAM (disease-associated microglia) | Keren-Shaul et al. 2017 (Nature) | TREM2, APOE, SPP1, LPL, GPNMB | Microglia |
| Homeostatic microglia | Multiple | P2RY12, CX3CR1, TMEM119, SALL1, OLFML3 | Microglia |
| CD8+ Trm (tissue-resident memory T cells) | GSE301908; Lerma-Martin | ITGAE, CXCR6, ZNF683, PDCD1, GZMB | T cells |
| CD8-Trm + foamy µglia co-signature | GSE301908 | ABCA1↓ + ITGAE (spatial co-localization marker set) | Microglia + T cells |
| Pan-reactive astrocyte | Multiple | GFAP (high), VIM, CLDN10, CD44, OSMR | Astrocytes |
| IFN-responsive astrocyte | Absinta; Lerma-Martin | IFITM3, B2M, HLA-DRA, STAT1, STAT3 | Astrocytes |
| Scar-associated astrocyte | MS literature | GFAP (high), CSPG4, TNC, CHI3L1, LCN2 | Astrocytes |
| Stress/injured oligodendrocyte | Absinta; MS literature | HSPA1A, HSPA1B, ATF3, DDIT3, GADD45B | OL-lineage |
| Remyelinating OPC | MS literature | BCAS1, ENPP6, MAL, PLP1 (low), PDGFRA (low) | OPC |
| OPC activation / differentiation-impaired | Absinta; GSE301908 | PPARG, OLIG2, ID4, SOX2, CCND1 | OPC |
| Activated endothelium / BBB disruption | Lerma-Martin; MS literature | ICAM1, SELE, VCAM1, CXCL10 | Endothelial |

**Implementation steps**:
1. For each signature above, extract gene lists from the published marker tables in the cited papers.
2. Apply the same quality filter as curated programs: minimum 15 genes per gene set after intersecting with the detected gene space.
3. Score using UCell against pseudobulk µ (same pipeline as other curated programs).
4. Store in `.uns["program_scores_curated"]` alongside the existing ~195 programs.

**Estimated total**: ~195 (v0) + ~25 new = ~220 curated programs in v1.

---

### Biomarker Discovery Interface

**Status**: Design complete. Implement as a standalone analysis module after core v0 pipeline is stable.

**Problem being solved**: v0 produces warp magnitudes and top programs per warp axis. These describe the population-level effect. The biomarker interface answers a different question: given two groups of samples (e.g., responders vs non-responders, treated vs untreated), which programs and orbital states are differentially activated?

**Design**:

Input: sample group assignments (column in the metadata table). Groups A vs B (can be binary or multi-class with one-vs-rest).

Per cell family, per program:
1. Collect program scores for all samples in group A and group B.
2. Run Mann-Whitney U test (non-parametric; no normality assumption).
3. Compute: U-statistic, p-value, effect size (rank-biserial correlation = 2U/(n_A · n_B) − 1), BH FDR q-value across all programs × cell families.

Per cell family, per L2 orbital state:
1. Compute orbital membership fraction (fraction of pseudobulk groups per sample assigned to each L2 state).
2. Compare fractions between groups A and B using Mann-Whitney U.
3. Report: Δ_fraction, p-value, BH FDR q-value.

> **Why BH FDR is valid here**: we are testing N_programs × N_families independent hypotheses across N_A + N_B independent samples. This is the classical multiple testing scenario where BH FDR is appropriate. This is distinct from back-projection, where Δ_program is a deterministic transformation of a single warp vector (no sampling, no null distribution).

Output files:
- `biomarker_program_scores.csv`: per cell family × program × group comparison: mean_A, mean_B, Δ_score, U-stat, p-value, FDR q-value
- `biomarker_orbital_membership.csv`: per cell family × L2 state × group comparison: fraction_A, fraction_B, Δ_fraction, p-value, FDR q-value

---

### Additional v1 Items (from v0.4 known limitations)

**Donor random effects / mixed model warp fitting**:
- v0 limitation: pseudobulk groups from the same donor across spatial compartments are not independent, but v0 treats them as independent.
- v1 fix: add `donor_id` as a random effect in the warp fitting model using a mixed-effects formulation. Candidate libraries: `lme4` (R) or `statsmodels` MixedLM (Python).
- Dependency: requires enough donors (≥ 10) for random effects to be estimable. With N < 10, random effects may not converge.

**Low-rank + diagonal covariance for orbitals**:
- v0 uses diagonal covariance Gaussians. This misses correlated program directions (e.g., IFN-high and lipid-high co-vary in foamy microglia).
- v1 model: Σ = Λ Λ^T + Ψ (low-rank + diagonal), where Λ has r << n_components columns.
- Requires: more healthy reference donors (≥ 15 per family per state for rank-2 covariance to be estimable). Check current healthy donor count before implementing.

---

## v2+ Additions

### Learnable [MISSING] Token for Absent Cell Families

**Status**: Design complete. Requires transformer (Layer 6) to be implemented first.

**Problem being solved**: when a cell family is absent from a sample (MISSING flag, confidence < 0.5), the v0 approach is to exclude the family token from the token set. This silently drops information. The transformer architecture should be able to learn that absence is itself a biological signal.

**Design**:

For each cell family f, maintain a learnable parameter vector `e_missing_f` of dimension d_token (same as a normal token embedding). When family f is absent in sample s:
- Replace the token for family f in sample s with `e_missing_f`.
- Do NOT replace with zero vector — zero is not a neutral signal in transformer attention.

**Per-family vs shared**: recommend per-family [MISSING] tokens (one per cell family). Shared [MISSING] tokens lose the family identity signal.

**Initialization**: random normal N(0, σ_token) where σ_token is the typical token embedding standard deviation at initialization.

**Training**: `e_missing_f` parameters are trained jointly with all other transformer parameters. No special learning rate or regularization needed.

**Attention behavior**: the [MISSING] token participates in cross-attention normally — other family tokens can attend to it and learn "T cells absent in this sample." This is the key feature: the model can learn cross-family absence patterns.

**Evaluation**: after training, inspect which families' [MISSING] tokens receive the most attention from other families. This reveals which absences are most informative for disease state prediction.

---

### Inter-Orbital Coupling Matrix

**Status**: Design complete. Requires: (a) enough samples for cross-family covariance estimation; (b) v0.5 sample imprints as validation baseline.

**Problem being solved**: v0 fits and warps each cell family's orbital independently. IFN-high microglia and IFN-responsive astrocytes co-occur at the active rim — this is captured indirectly by the sample imprint (concatenated warps), but there is no explicit structural parameter encoding the coupling.

**Design**:

For each pair of cell families (A, B), estimate the coupling coefficient C_{AB}:
```
C_{AB} = Cov(Δ_A, Δ_B) / (||Δ_A|| · ||Δ_B||)
```
where Δ_A and Δ_B are the per-sample warp vectors for families A and B (in their respective per-family PCA spaces, after projecting to a common low-dimensional summary, e.g., warp magnitude + top-3 program loadings).

The full coupling matrix C is N_families × N_families (symmetric). Entries near 1: families co-vary (co-activation); near −1: anti-correlated (one activates, other suppresses); near 0: independent.

**Estimation requirements**: need ≥ 2 × (N_families²) samples for the covariance matrix to be estimable with reasonable precision. For 7 families, that is ~100 samples minimum. With the three v0 datasets combined (~40–60 total samples), the coupling matrix will be noisy — report with large confidence intervals.

**Connection to sample imprint**: the sample imprint (v0.5) provides an empirical approximation of inter-family coupling via clustering. The coupling matrix is the parametric version of the same concept. Use v0.5 imprint clustering as a sanity check against the coupling matrix structure — they should agree on which families co-cluster.

**v2+ scope note**: inter-orbital coupling requires transformer-scale data to be estimated reliably. It is a v2+ feature, not a v1 target.

---

### Masked Cell Type Prediction Pre-Training Objective

**Status**: Design complete. Requires transformer (Layer 6) to be implemented first.

**What it is**: BERT-style self-supervised pre-training for the cell type transformer. Learns inter-cell-type dependencies without requiring condition labels.

**Training procedure**:
1. For each training sample, randomly select 15% of cell family tokens to mask.
2. Replace masked tokens with a learnable [MASK] token embedding (shared across families, distinct from [MISSING]).
3. Task: predict the program score vector of each masked token from the unmasked context via the transformer encoder.
4. Loss: mean squared error between predicted and actual program score vectors for masked tokens.

**Why this objective**: it forces the attention mechanism to learn cross-family dependencies. To predict the masked microglia token, the transformer must use astrocyte, T cell, and OL token context — learning which patterns co-occur. This is more informative than direct supervised training on condition labels alone.

**Training recipe**:
1. Pre-train on masked prediction using all available snRNA-seq samples (public + partner, condition-agnostic).
2. Fine-tune on warp prediction (supervised: given token set, predict warp axis assignments).
3. For imprint space (Layer 7): add supervised contrastive and trajectory contrastive objectives during fine-tuning.

**Hyperparameters** (starting point):
- Masking probability: 0.15
- Token dimension d_token: 128 (adjust based on n_families × n_programs)
- Transformer depth: 4–6 layers, 4–8 heads
- Pre-training epochs: 100 (or until plateau on masked prediction loss on held-out donors)

---

### Contrastive Learning for Imprint Space

**Status**: Design complete. Requires transformer (Layer 6) + imprint space (Layer 7) to be implemented first.

**Two objectives**:

**1. Supervised contrastive loss (condition labels)**:
Using the SupCon loss (Khosla et al. 2020, NeurIPS):
```
L_SupCon = Σ_i (−1 / |P(i)|) Σ_{p ∈ P(i)} log [exp(z_i · z_p / τ) / Σ_{a ∈ A(i)} exp(z_i · z_a / τ)]
```
Where z_i = L2-normalized imprint for sample i, P(i) = set of samples with same condition label, A(i) = all other samples, τ = temperature (default 0.07).

Pulls chronic-active rim samples together; pushes them away from chronic-inactive rim, NAWM, and healthy. Ensures the imprint space is organized by disease biology, not batch.

**2. Trajectory contrastive loss (spatial/temporal ordering)**:
For the MS application, the known spatial ordering is:
```
healthy → NAWM → rim → core
```
Use a margin-ranking triplet loss:
```
L_triplet = max(0, ||z_A − z_B||₂ − ||z_A − z_C||₂ + margin)
```
where B is "closer" to A on the disease axis than C. For MS: if A=NAWM, B=rim, C=core, then imprint(A) should be closer to imprint(B) than to imprint(C).

**Margin**: start with margin = 0.1 (relative to normalized unit sphere). Tune on LODO validation.

**Combined training objective**:
```
L_total = L_warp_supervised + α · L_SupCon + β · L_triplet
```
Default: α = 0.5, β = 0.25. Treat as hyperparameters; sweep during fine-tuning.

**Evaluation**: t-SNE/UMAP of the learned imprint space should show: (a) condition-organized clusters, (b) smooth manifold from healthy → NAWM → rim → core along the trajectory axis. Compare to v0.5 sample imprint clustering as baseline.

---

## Explicitly Out of Scope for Verse Core

**Druggability check** (gene → known drug target lookup): this is a downstream application layer, not a core Verse architecture feature. The relevant databases (ChEMBL, DGIdb, OpenTargets) and the CNS-specific constraints (BBB penetration, CNS-penetrant drug filtering) belong in a separate analysis layer built on top of Verse outputs. If needed, implement as a post-processing script that takes the top warp-driving genes from `warp_report.csv` and queries OpenTargets.

---

## Version Tracking

| File | Version | Last reviewed |
|------|---------|---------------|
| VERSE_ARCHITECTURE.md | v0.5 | 2026-04-13 |
| ENGINEERING_CHECKLIST_MS_FIRST.md | v0.5 | 2026-04-13 |
| FUTURE_VERSIONS_ROADMAP.md | v0.5 | 2026-04-13 |

When adding new planned features, add them to the appropriate version section here and create corresponding orphan entries in `ARCHITECTURE_IMPLEMENTATION_MAP.md`.

---

*Version: v0.5 — created 2026-04-13. Consolidates all v1/v2+ specs from: (a) v0 known limitations documented in VERSE_ARCHITECTURE.md and ENGINEERING_CHECKLIST_MS_FIRST.md; (b) 12-gap conceptual review against the original ImmuneVerse specification (σ_program delta method, [MISSING] token, Bhattacharyya overlap, orbital scaling warp, inter-orbital coupling, PROGENy, CNS signatures, warp interpolation, biomarker discovery interface, masked cell type prediction, contrastive learning objectives).*
