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

**Inter-orbital coupling** (v2+): in v0 each cell family's orbital is fit and warped independently. In reality, cell families co-vary — IFN-high microglia tend to co-occur with IFN-responsive astrocytes; foamy microglia expansion correlates with CD8+ Trm T cell infiltration at the active rim. The sample imprint (§13) captures this at the sample level via concatenation. The v2+ extension adds explicit coupling parameters: a cross-family covariance matrix **C** where C_{AB} captures the degree to which orbital shift in family A predicts orbital shift in family B. This enables joint inference and cross-family mechanistic hypotheses. See `FUTURE_VERSIONS_ROADMAP.md §v2+` for the design spec.

---

## 2. Inputs

- Single-cell or single-nucleus expression data (optionally multi-omics).
- Per-cell metadata: sample/donor ID, cell family label, condition label (disease, niche, treatment, etc.), technical covariates (batch, protocol, chemistry).
- **Required metadata columns** (at pseudobulk level): `reference_flag` (bool, True for healthy controls), `spatial_label`, `activity_label`, `dataset_id`.

**Schema rules for spatial_label and activity_label** (fix: do not encode both in a single field):
- `spatial_label` encodes **spatial position only**: `{healthy, NAWM, rim, core, unknown}`. Do NOT use combined values like `rim_active` or `rim_inactive`.
- `activity_label` encodes **lesion activity only**: `{healthy, chronic_active, chronic_inactive, unknown}`.
- The intersection (e.g., chronic-active rim) is derived as `spatial_label=rim` + `activity_label=chronic_active`. Encoding both in `spatial_label` creates redundancy and can produce schema inconsistencies.
- Schema validator must reject any `spatial_label` value containing an activity suffix.

---

## 3. Layer 1 — Probabilistic Pseudobulk

### Concept
Aggregate single-cell/nucleus data by **Sample × CellFamily × SpatialLabel** into probabilistic pseudobulk. Each pseudobulk row = one donor × one cell family × one spatial compartment. For each group and each gene: mean expression **µ**, standard deviation **σ**, number of cells **n**, confidence score.

> **Note**: The correct aggregation granularity is Sample × CellFamily × SpatialLabel. Aggregating only by Sample × CellFamily merges cells from different spatial compartments (e.g., NAWM and rim) into a single group, which eliminates spatial resolution and makes the warp fitting meaningless.

### Uncertainty model (NB variance, not 1/√n)
Single-cell/nucleus RNA-seq has gene-specific overdispersion that simple Poisson models underestimate. Use:

```
σ²_gene = µ + µ² / r_gene
```

where **r_gene** is a gene-specific overdispersion parameter. Apply **DESeq2-style empirical Bayes dispersion shrinkage** to stabilize estimates for lowly expressed or sparse genes by sharing information across genes of similar expression level.

Confidence is then a function of both n and gene-specific overdispersion — not just sampling depth.

### Hard threshold
- **min_cells = 50** (default): groups with fewer cells are excluded and receive a **MISSING** flag with **zero confidence**.
- **Per-family threshold overrides** for sparse families:

  | Cell family | min_cells |
  |-------------|-----------|
  | T_cell | 20 |
  | endothelial | 20 |
  | neuron | 30 |
  | all others | 50 |

  T cells in white matter are sparse — often 50–200 cells per donor across the entire tissue. Aggregated at donor × T_cell × spatial_label, most groups will not reach 50 cells. A lower threshold (20) is used with proportionally lower confidence scores. T cell warp analysis is **best-effort**: report only if ≥ 5 donors have ≥ 20 T cells per compartment; otherwise report cell counts only and exclude from warp analysis.
- Groups with cells between min_cells and 150 are included but down-weighted proportionally by their confidence score.

### Why pseudobulk
Respects donor/replicate structure (prevents pseudoreplication), captures biological heterogeneity, and propagates uncertainty downstream to programs, orbitals, and warp fitting.

---

## 4. Layer 2 — Two-Channel Program Encoder

### Concept
Map gene-level pseudobulk distributions into a **two-channel program/pathway space**:

### Channel 1 — Curated programs (~195 total, interpretable)
Scored against pseudobulk µ/σ using gene-set scoring methods (e.g., UCell or ssGSEA on pseudobulk). Recommended v0 library:

| Source | Programs | Selected content |
|--------|----------|-----------------|
| Reactome CNS | ~50 | Myelination, neuroinflammation, complement cascade, synaptic signaling |
| MSigDB Hallmarks | ~50 | IFN-γ/α, hypoxia, oxidative phosphorylation, TNF/NF-κB, apoptosis, fatty acid metabolism |
| GO:BP modules | ~30 | Myelination, lipid/cholesterol metabolism, ROS, phagocytosis, complement activation |
| DoRothEA TF regulons | ~50 | NF-κB, STAT1, STAT3, HIF1A, PPARG, RUNX1, IRF regulons |
| PROGENy | 14 | JAK-STAT, NFkB, TGFb, MAPK, PI3K, VEGF, WNT, EGFR, Androgen, Estrogen, Hypoxia, p53, Trail, TNF |

**Why PROGENy**: PROGENy is perturbation-calibrated — trained on transcriptomic responses to experimentally validated pathway perturbations. It is more reliable than GO:BP for detecting active signaling (vs. gene co-expression), especially for JAK-STAT and TNF/NFkB which are central to MS lesion biology. Use `decoupleR` (R) or the `progeny` Python port.

> **v1 addition — CNS cell-state signatures**: 20–30 domain-specific gene sets derived from the three anchor datasets (GSE180759, GSE301908, Lerma-Martin 2024) and MS literature. These capture MS-specific cell states that general pathway databases underrepresent (foamy microglia ABCA1-low, iron-accumulating microglia, CD8+ Trm niche, DAM signature). See `FUTURE_VERSIONS_ROADMAP.md §v1` for the planned signature table.

### Channel 2 — Residual/NMF programs (data-driven, novelty capture)
1. **Residualization (ridge regression, not OLS)**: regress each gene's pseudobulk expression on curated program scores using **L2-penalized (ridge) regression**; take residuals. Rationale: OLS with ~180 predictors on sparse families (T cells, endothelial, neurons may have only 15–40 pseudobulk groups) is massively underdetermined — OLS overfits to zero residuals, leaving nothing for NMF. Ridge regression regularizes and prevents this.

   **Guard**: if `n_pseudobulk_groups < 2 × n_curated_programs` for a cell family, **skip residualization entirely** and run NMF directly on the curated program score matrix. Log a WARNING.

2. Apply **consensus NMF** to residuals (or directly to the curated program score matrix if guard triggered) per cell family:
   - Run NMF 20× with different random seeds on 80% subsamples.
   - Select K via **cophenetic correlation**. Note: a threshold of 0.95 is unrealistically strict for n < 200 (typical of v0 datasets). Use tiered criteria: ≥ 0.95 (excellent) → ≥ 0.85 (satisfactory) → fallback K=10 with WARNING logged if max cophenetic < 0.85. K floor = 5, K cap = 20.
   - **Expected in v0 datasets** (n ≈ 32–160 per family): residualization guard will fire (n < 2p = 390), so NMF runs directly on curated program scores. Max cophenetic will likely be 0.80–0.90. K=10 fallback is expected and valid.
   - Extract consensus W (gene-program) and H (pseudobulk-program) matrices.
   - Threshold: retain top-50 genes per component (by W weight) for naming.
   - Name programs by pathway overlap (Fisher's exact, FDR-corrected).

### Preprocessing: Z-scoring (two separate operations)
Z-scoring must be done at two distinct scopes — **do not use a single global z-score for both purposes**:

| PCA level | Z-scoring scope | Rationale |
|-----------|-----------------|-----------|
| Global PCA (all families) | Z-score across **all pseudobulk units globally** (per program, all families pooled) | Appropriate: normalizes scale differences across all groups |
| Per-family PCA | Z-score **within each cell family** separately (per program, only that family's pseudobulk subset) | Required: global z-scoring before per-family PCA lets cell-family identity dominate the embedding, masking within-family condition variation |

> **Do NOT feed the global z-scored matrix into per-family PCA.** Re-compute z-scores within each family's pseudobulk subset before per-family PCA.

### Two-level PCA
| Level | Scope | Purpose |
|-------|-------|---------|
| Global PCA | All cell families combined, 20–30 components | Cross-family visualization, coarse alignment, patient/sample embedding |
| Per-family PCA | Each cell family separately (within-family z-scored) | Orbital fitting, warp estimation, novelty scoring |

Per-family PCA prevents abundant families (microglia, astrocytes) from dominating the embedding at the expense of sparse families (T cells, endothelial).

### Back-projection (interpretability recovery)
After fitting in per-family PCA space, project warp shift vectors Δ_PCA back to the original program space:
```
Δ_program = W_family^T × Δ_PCA
```
This recovers named program contributions (e.g., "IFN-γ and lipid metabolism programs drive the rim warp in microglia").

### Program uncertainty (σ_program) — v0 limitation, v1 spec
Layer 1 produces per-gene µ (mean) and σ² (NB variance) for each pseudobulk group. In v0, program scores are computed from µ only (UCell/ssGSEA). **σ_program is not propagated through UCell/ssGSEA in v0.** Downstream novelty scoring therefore uses σ²_orbital only, not σ²_sample + σ²_orbital. This is an acknowledged limitation.

**v1 implementation — delta method propagation**: for any program P scored as a weighted sum of gene expression:
```
score_P = Σ_g  w_g · µ_gene_g

σ²_program = Σ_g  w²_g · σ²_gene_g        (delta method, first-order approximation)

σ_program = sqrt(σ²_program)
```
Where `w_g` are the UCell/ssGSEA gene weights and `σ²_gene_g` is the NB variance from Layer 1. This propagates gene-level uncertainty into program-level uncertainty without requiring new sampling.

**v1 pipeline changes** (three additions):
1. Layer 2: compute `σ²_program` alongside `score_P` using the formula above.
2. Layer 3 novelty: replace `P(x | orbital_k)` with `σ²_total = σ²_orbital + σ²_program` as the combined variance term.
3. Warp report: add per-program uncertainty bands (score_P ± 1.96·σ_program) to `warp_report.csv`.

**Alternative (non-linear scorers)**: Monte Carlo sampling — draw N=200 gene expression vectors from the NB distribution per pseudobulk group, score each, and take the empirical standard deviation. More accurate for non-linear scorers but ~200× slower. See `FUTURE_VERSIONS_ROADMAP.md §v1` for implementation details.

### Shift vs identity
Always report **within-cell-type condition contrasts** (Δ_program vectors and warp magnitude) — not absolute pathway levels. Identity programs (e.g., "microglia-ness") are stable within a cell family and are not the target signal.

---

## 5. Layer 3 — Orbital Layer (L1/L2 Hierarchy)

### Hierarchy
- **L1 orbitals** = cell family identity, defined from healthy reference tissue. Stable across conditions.
- **L2 orbitals** = canonical states within each cell family. Warp fields operate at this level.

### Covariance model: diagonal (v0); low-rank + diagonal (v1+)
For v0, orbitals are modeled as **diagonal covariance Gaussians** per cell family. Rationale: with 4–8 healthy donors available, the low-rank + diagonal model (Σ = Λ Λ^T + Ψ) almost always falls back to strict diagonal anyway (< 3 × n_PCA_components pseudobulk groups per state). Making diagonal the explicit v0 default is more honest. Low-rank covariance fitting (capturing correlated program directions, e.g., IFN-high × lipid-high) is a **v1 feature** requiring more healthy reference donors.

### Reference orbital fitting: data-driven from healthy tissue
**Do not pre-define disease states as initialization inputs.** The reference orbital for each cell family is fit directly and exclusively from healthy-reference pseudobulk groups:

1. Take all pseudobulk groups where `reference_flag=True` for a given cell family.
2. Check for substructure in healthy data: if ≥ 2 well-separated clusters emerge (gap statistic or silhouette criterion on within-family PCA space), fit K Gaussians (K determined by the data, not assumed). Otherwise fit a single Gaussian.
3. **That is the reference orbital** — no pre-labeling required. The orbital is defined entirely by what healthy tissue looks like.
4. Disease pseudobulk groups land wherever they land in program space. The shift from the reference orbital center IS the warp vector.

**Anchoring criterion**: ≥ 80% of healthy-reference pseudobulk groups must fall inside reference orbitals at 99% confidence. This is a validation check, not a fitting target.

### Disease state discovery (post-fitting, data-driven)
L2 disease states (DAM-like, IFN-responsive, foamy microglia, etc.) are **discovery outputs**, not initialization inputs. After warp estimation:

1. Cluster disease pseudobulk warp vectors (per cell family) using k-means or hierarchical clustering.
2. For each cluster, identify its top program signatures.
3. Annotate clusters against the **Expected Disease State Annotation Guide** below (post-hoc labeling).
4. Any cluster that does not match a known state is flagged as a **novelty candidate** (promote only if replicated ≥ 2 donors).

This architecture is more scientifically defensible: it measures confirmation rather than enforcing prior assumptions, and naturally handles the absence of disease-specific states in healthy reference tissue.

### Expected Disease State Annotation Guide (post-discovery labels, not initialization inputs)

> These are expected outputs of the discovery step — not inputs to orbital fitting. If a discovered cluster matches one of these states, apply the label. If it does not match, treat it as potentially novel.
>
> **Usage constraint**: Do not consult this guide before or during warp estimation, orbital fitting, or cluster discovery. Only use it after clusters are finalized and independently characterized (e.g., by top marker gene enrichment), as a reference for applying standard terminology. Consulting this guide during discovery would bias cluster interpretation toward expected outcomes.

| Cell family | Expected states (annotation guide) | Key markers |
|-------------|-------------------------------------|-------------|
| Microglia | homeostatic, DAM-like, IFN-responsive, foamy/ABCA1-G1-low, iron-accumulating | P2RY12/CX3CR1; TREM2/APOE; ISG15/MX1; ABCA1↓/PLIN2; FTH1/SLC40A1 |
| Astrocytes | homeostatic, pan-reactive, IFN-responsive, scar-associated | SLC1A2/ALDH1L1; GFAP-high/VIM/CLDN10; IFITM3/B2M/HLA-DRA; GFAP-high/CSPG4/TNC |
| OL-lineage | myelinating OL, pre-myelinating OL, stress/injured OL | MBP/MOG; BCAS1/ENPP6; HSPA1A/ATF3 |
| OPC | quiescent OPC, activated OPC (PPARG+, differentiation-impaired) | PDGFRA; PPARG/OLIG2 |
| T cells | CD8+ Trm (rim niche), CD4+ Th17-like, Treg | ITGAE/CXCR6; RORC/IL17A; FOXP3/IL2RA |
| Neurons | baseline (minimal warp expected — **negative control**) | RBFOX3/MAP2 |
| Endothelial | homeostatic, activated/inflamed | VWF/CDH5; ICAM1/SELE |

> **Astrocyte note**: A1/A2 binary classification (Liddelow et al. 2017) is not used. This binary framing has not replicated robustly in vivo and was characterized from in vitro LPS stimulation. The four continuous reactive states above (pan-reactive, IFN-responsive, scar-associated, homeostatic) have more reliable marker support in human MS tissue. If discovered clusters correspond to these continuous states, use these labels.

### Novelty scoring
```
novelty_score(x) = 1 − max_k P(x | orbital_k)
```
where P(x | orbital_k) uses **σ²_orbital** (orbital estimate uncertainty). Note: σ²_sample is not included in v0 because σ_program is not propagated through program scoring (see Layer 2 limitation note). This is a known v0 limitation.

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

### Orbital overlap check (Bhattacharyya distance)
After discovery of L2 disease states (§ "Disease state discovery"), check that discovered orbitals are sufficiently distinct. Two orbitals that largely overlap may indicate the clustering step selected too many K.

For diagonal Gaussian orbitals A and B:
```
Bhattacharyya distance:
  D_B(A, B) = (1/8)(µ_A − µ_B)^T Σ_avg^{−1} (µ_A − µ_B)
              + (1/2) ln(|Σ_avg| / sqrt(|Σ_A| · |Σ_B|))

  Σ_avg = (Σ_A + Σ_B) / 2

  Bhattacharyya coefficient (overlap similarity): BC = exp(−D_B)
  BC = 0 → fully disjoint; BC = 1 → identical orbitals
```

Compute pairwise D_B across all L2 discovered state orbitals within each cell family. **Flag any pair with BC > 0.5 as "potentially redundant states."** Redundant pairs should be reviewed: either merge the states or revisit the clustering K. Output: `orbital_overlap_matrix.csv` (pairwise BC per cell family). This matrix is also useful for presentations — it quantifies how biologically distinct the discovered states are.

---

## 6. Layer 4 — Factorized Multiverse Warping

### Reference
**Healthy tissue** (reference_flag = True) defines fixed orbital means and covariances. These are held fixed during warp fitting.

### Warp model (per cell family, per-family PCA space)
```
center_condition = center_healthy
                  + w_spatial · X_spatial
                  + w_activity · X_activity
                  + Σ_{i,j} w_{ij} · 1[spatial=i] · 1[activity=j]   ← interaction term
                  + w_dataset · X_dataset
                  + w_sex · X_sex
                  + w_disease_duration · X_disease_duration
```

> **Interaction term notation**: `Σ_{i,j} w_{ij} · 1[spatial=i] · 1[activity=j]` means a separate warp parameter per (spatial_bin, activity_bin) combination (e.g., w_rim_chronic_active, w_core_chronic_inactive). This is NOT a Kronecker product of the full design matrices — it is an element-wise indicator product for each pair.

| Parameter | Role |
|-----------|------|
| w_spatial | Spatial compartment effect (NAWM / rim / core) |
| w_activity | Lesion activity effect (chronic-active / inactive) |
| w_{ij} | Spatial × activity interaction per (i, j) pair (fit only if ≥3 donors per condition cell) |
| w_dataset | Dataset/protocol fixed effect (non-biological covariate) |
| w_sex, w_disease_duration | Additional demographic covariates |

**Known limitation — donor non-independence**: pseudobulk is aggregated at donor × cell_family × spatial_label, so the same donor contributes groups for NAWM, rim, and core. These groups are **not independent observations** — they share donor-level biological and technical variation. In v0, each pseudobulk group is treated as independent (standard OLS/ridge formulation). In v1, add `donor_id` as a random effect (mixed model) to properly account for within-donor correlation.

### Collinearity check (required before fitting)
1. Build the full spatial × activity × dataset_id cross-table.
2. **Primary criterion**: compute VIF (Variance Inflation Factor) for each warp axis. If any axis has VIF > 10, spatial and activity axes are not separable.
3. **Fast first check**: condition number of the design matrix. Condition number > 30 triggers a VIF computation (but condition number alone is scale-dependent and not the definitive criterion).
4. **Collapse** to a single **probabilistic lesion axis** when VIF > 10. Document this explicitly as a limitation — it does not invalidate the model, but does limit mechanistic claims.

### Interaction sparsity rule
Fit w_interaction only if ≥ 3 donors exist per condition cell (each spatial × activity combination). Otherwise drop the interaction term and log all skipped cells.

### L2 regularization and nested cross-validation
Warp fitting minimizes:
```
L = reconstruction_error + λ · ||w||²
```

**Nested cross-validation** (required — avoids data leakage):
- **Outer loop** (evaluation): leave-one-donor-out (LODO) — hold out donor D, evaluate warp direction ρ on D.
- **Inner loop** (λ selection): for each outer fold, select λ by cross-validation on the N-2 remaining donors.
- Do NOT select λ on the same LODO splits used for evaluation. Using the same splits for both hyperparameter tuning and evaluation produces optimistically biased ρ estimates.
- With small N (5–15 donors), the inner loop has very few folds. Document the exact nested CV scheme in run metadata.

### Warp magnitude and orbital scaling (exact definitions)
```
Center shift (warp magnitude):
  warp_magnitude = ||Δ_PCA||₂     (L2 norm of orbital center translation in per-family PCA space)

Orbital scaling warp (σ expansion/contraction):
  For each warp axis w_k and cell family f:
    scale_ratio_k = σ_condition / σ_reference
                    (per PCA dimension; summarize as geometric mean across dimensions)

  scale_ratio > 1 → orbital expansion  (more heterogeneous / dispersed state under condition k)
  scale_ratio < 1 → orbital compression (more uniform / "locked" state under condition k)
```

Both are first-class reported quantities per cell family per warp axis:
- `warp_magnitude`: orbital center shift (how far the distribution moved)
- `scale_ratio`: orbital size change (how the within-state heterogeneity changed)

**Why scale matters**: foamy microglia at the active rim are heterogeneous (broad orbital, scale_ratio > 1), reflecting a mixed lipid-loading state. Homeostatic microglia have a tight orbital. Scale compression (scale_ratio < 1) can indicate a "locked" disease state where cells have lost transcriptional flexibility. Center shift alone misses this dimension entirely.

### Back-projection + effect-size ranking
1. Project Δ_PCA via W_family^T to recover named program contributions Δ_program.
2. Rank programs by |Δ_program| (effect size magnitude). Report top-K programs as the interpretation of each warp axis.

> **Why not FDR**: back-projection is a deterministic linear transformation. It produces **effect sizes**, not p-values or test statistics. There is no null distribution and no sampling variability being quantified, so BH FDR correction is not statistically meaningful here.

**Statistical validation (v0+ extension)**: to assign significance to Δ_program values, either:
- Bootstrap warp estimation by resampling donors (N=1000); compute 95% CI for each Δ_program. Programs with CI excluding zero are reliably warp-driving.
- Permute donor condition labels (N=1000) to build a null distribution for each |Δ_program|; compute empirical p-values, then apply BH FDR.

For v0: rank-based reporting only. Bootstrap/permutation significance is a v0+ extension.

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

**[MISSING] token for absent cell families** (v2+): when a cell family is absent from a sample (MISSING flag, confidence < 0.5), the transformer receives a **learnable [MISSING] embedding** instead of a zero vector or silent exclusion. This is a per-family embedding of the same dimension as a normal token, initialized randomly and trained jointly with the transformer. It encodes "this cell type was undetectable in this sample" as a first-class signal. This matters because absence can itself be biological — T cells are absent in NAWM but present at the active rim; the transformer should be able to learn this pattern. See `FUTURE_VERSIONS_ROADMAP.md §v2+` for initialization strategy and attention behavior.

**Masked cell type prediction pre-training objective** (v2+): BERT-style self-supervised pre-training. During pre-training, randomly mask 15% of cell-family tokens (replace with a learnable [MASK] token). Task: predict the masked token's program score vector from the remaining cell-family context. This forces the attention mechanism to learn inter-cell-type dependencies (e.g., the T-cell token can be predicted from the microglia + astrocyte tokens in co-occurring disease states) rather than within-family patterns alone. Pre-trained weights are fine-tuned on the supervised warp prediction and imprint task. See `FUTURE_VERSIONS_ROADMAP.md §v2+` for training recipe.

*Deferred to v2+. Only implement if required for a specific near-term task.*

---

## 9. Layer 7 — Imprint Space & Multi-Task Heads (v2+)

Patient imprint used for contrastive objectives and supervised prediction heads. Two contrastive training objectives are specified for v2+:

**1. Supervised contrastive loss** (condition labels): samples from the same condition (e.g., chronic-active rim) are pulled together in imprint space; samples from different conditions are pushed apart. Uses the standard SupCon loss (Khosla et al. 2020). This ensures the imprint space is organized by disease condition, not by batch or technical variation.

**2. Trajectory contrastive loss** (spatial/temporal ordering): if sample B is known to be "further along" the disease progression axis than sample A (e.g., rim > NAWM in spatial progression), the imprint distance should reflect this ordering. Uses a triplet/ranking loss:
```
  ||imprint(A) − imprint(B)||₂ < ||imprint(A) − imprint(C)||₂
  when B is spatially/temporally between A and C on the disease axis
```
For the MS case, the known ordering is: healthy → NAWM → rim → core. This loss encourages the imprint space to be a smooth manifold aligned with disease progression geometry.

See `FUTURE_VERSIONS_ROADMAP.md §v2+` for training recipe and loss weighting.

*Deferred to v2+.*

---

## 10. Geodesics & Discovery Modules (v2+)

**Geodesic Analyzer**: expected trajectories in warped space between conditions; measures deviation of actual movement from geodesic.

**Warp Field Analyzer**: which orbitals are most warped; which programs drive shifts; human-readable mechanism summaries.

**Discovery tools**: novel state detection; biomarker discovery from warp-driven program changes.

**Biomarker discovery interface** (v1+): two-group comparison of orbital membership scores and per-program scores between sample groups. Given groups A and B (e.g., responders vs non-responders, treated vs untreated):
- For each cell family × program: compute Mann-Whitney U test (non-parametric, no normality assumption) on the program scores across samples in group A vs B.
- Output: Δ_score (group mean difference), U-statistic, p-value, BH FDR q-value.
- Also compare orbital membership fractions per cell family (proportion in each L2 state) between groups.

> **Why Mann-Whitney U here (not earlier)**: in back-projection, the FDR correction was invalid because Δ_program comes from a deterministic transformation of a single warp vector — there is no sampling null. Here we ARE comparing N_A vs N_B independent samples, so statistical testing is appropriate.

*Geodesics, warp field analyzer, and transformer-based discovery deferred to v2+. Biomarker interface is v1+. Warps in v0 are already a mechanistic readout without these modules.*

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
| LODO: Spearman ρ of warp direction vectors | ≥ 0.65 (aspirational; realistic range 0.50–0.65 with N≈20) | Yes (primary families) |
| LOSO: Jaccard of top-10 warp-driving programs | ≥ 0.5 in ≥ 2 of 3 folds | Yes (primary families) |
| Batch warp magnitude / biological warp magnitude | < 50% | Yes |
| Healthy-reference cells inside orbitals at 99% CI | ≥ 80% | Yes |
| Neurons warp magnitude / microglia warp magnitude | < 25% | Global |

**Primary vs best-effort family tiers**:
| Tier | Families | Notes |
|------|----------|-------|
| Primary | microglia, astrocyte, OL_lineage, OPC | Abundant in WM; all metrics apply |
| Best-effort | T_cell, endothelial, neuron | Sparse; LODO/LOSO reported if data permit; not hard gates |

**LOSO caveat**: LOSO with 3 datasets produces exactly 3 fold evaluations. This is **not sufficient for statistical claims about generalization**. Interpret LOSO Jaccard ≥ 0.5 as a qualitative consistency check — a sanity check that top programs replicate, not a rigorous generalization test. Report all 3 fold Jaccard values individually; do not collapse to a single pass/fail.

Evaluate per cell family. Interpret relative to sampling depth — microglia warps will be more stable than T-cell warps given sparse T-cell sampling in white matter tissue.

**Known v0 evaluation limitation**: LODO is used for both λ selection (inner loop) and evaluation (outer loop) via nested CV. Document the exact nested CV scheme in metadata. With very small N (5–10 donors in some families), even nested CV is underpowered — treat LODO ρ as directional evidence, not a definitive benchmark.

---

## 13. Sample Imprint — Holistic Ecosystem Signature (v0.5, no transformer required)

Per-family warp vectors capture individual cell-type shifts. But MS lesion biology is fundamentally about the **ecosystem**: IFN-high microglia co-occur with IFN-responsive astrocytes; foamy microglia co-occur with CD8+ Trm T cells. Analyzing each family in isolation misses this cross-cell-type co-variation.

**Sample imprint**: concatenate per-family warp vectors across all families for a given sample:
```
imprint(s) = [Δ_microglia | Δ_astrocyte | Δ_OL | Δ_OPC | Δ_T_cell | Δ_neuron | Δ_endothelial]
```

Cluster or embed sample imprints (UMAP / k-means) → sample-level disease signature capturing the full cellular ecosystem state. This gives holistic U0/U1/U2 universe assignment based on the complete ecosystem, rather than aggregating per-family assignments.

**Why this is more powerful**: a per-family warp sees "microglia shifted toward foamy state." The imprint sees "microglia shifted AND astrocytes shifted AND T cells appeared" — which is the actual chronic-active rim niche, not a coincidence.

**This does NOT require the transformer (Layer 6/7)**. Simple concatenation + clustering is sufficient for v0.5 and provides the "patient imprint" concept without the full transformer overhead.

**Placement in roadmap**: v0 = per-family warps + per-family analysis; v0.5 = sample imprints from concatenated warps; v2 = transformer imprint + geodesic trajectories.

### Warp interpolation — trajectory visualization (v0.5)
Given two samples A and B in imprint space, linear interpolation traces a path between their ecosystem states:
```
interp(t) = (1 − t) · imprint(A) + t · imprint(B),   t ∈ [0, 1]
```
Stepping t from 0 → 1 visualizes the "path" between sample A's ecosystem state and sample B's ecosystem state through the program space.

**Use cases**:
- Visualize the disease progression axis: interpolate from a healthy donor imprint toward a chronic-active rim imprint → shows the sequence of cellular changes along that axis
- Show treatment response: shift from a disease-state imprint toward healthy
- Generate synthetic intermediate states for presentations or pedagogical figures

**Important caveats**: this is a visualization and hypothesis-generation tool, not a quantitative inference procedure. Linear interpolation in a high-dimensional program space is only interpretable as a "path" if the manifold is approximately linear in that region (which may not hold). Do not use interpolated points as inputs to orbital membership or warp estimation. Valid only between samples measured in the same reference orbital coordinate system.

---

## 14. Cross-Reference

| Topic | See also |
|-------|----------|
| Operational implementation steps | `ENGINEERING_CHECKLIST_MS_FIRST.md` |
| Architecture → checklist mapping | `ARCHITECTURE_IMPLEMENTATION_MAP.md` |
| MS-specific scientific strategy | `STRATEGY_MEMO_MS_FIRST.md` |
| Visual pipeline diagram | `VERSE_ARCHITECTURE_DIAGRAM.md` |
| One-page partner summary | `VERSE_FOR_MS_ONE_PAGE.md` |
| v1/v2+ roadmap and concrete specs | `FUTURE_VERSIONS_ROADMAP.md` |

---

*Version: v0.5 — last updated 2026-04-13. Key changes from v0.4: PROGENy added to curated library (14 programs → total ~195); σ_program delta method formula specified (v1 concrete spec); CNS cell-state signatures planned for v1; Bhattacharyya orbital overlap check added with formula; orbital scaling warp (scale_ratio) made a first-class v0 output; inter-orbital coupling documented as v2+ with design intent; [MISSING] token and masked cell type prediction pre-training added to Layer 6 spec; contrastive learning objectives (supervised + trajectory) specified in Layer 7; biomarker discovery interface specified in §10 (v1+); warp interpolation added to §13 sample imprint section; FUTURE_VERSIONS_ROADMAP.md created for all v1/v2+ items.*
