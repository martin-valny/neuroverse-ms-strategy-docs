# MS-first Engineering Checklist (Verse v0)

> **Prerequisite**: Read `VERSE_ARCHITECTURE.md` before implementing any step. This checklist operationalizes that spec. Cross-references to architecture sections are noted inline.

---

## 0) Condition Schema — Hard Prerequisite

Before any data is loaded, define and enforce the canonical metadata schema. **All downstream steps assume this schema. Runs that violate it must fail explicitly.**

Every pseudobulk row must contain these columns:

| Column | Type | Values | Notes |
|--------|------|--------|-------|
| `donor_id` | str | any | Per-individual identifier |
| `dataset_id` | str | {GSE180759, GSE301908, lerma_martin_2024} | Source dataset |
| `reference_flag` | bool | True/False | True only for healthy control donors |
| `spatial_label` | str | {healthy, NAWM, rim_active, rim_inactive, core_active, core_inactive, unknown} | Spatial + activity combined label |
| `activity_label` | str | {healthy, chronic_active, chronic_inactive, unknown} | Lesion activity status |
| `cell_family` | str | {microglia, astrocyte, OL_lineage, OPC, T_cell, neuron, endothelial, other} | Stable cell family vocabulary |
| `n_cells` | int | ≥ 50 (after filtering) | Cell count in this pseudobulk group |
| `confidence` | float | [0, 1] | NB-derived confidence score |

- [ ] Write a schema validation function that reads in AnnData `.obs` (or pseudobulk DataFrame) and raises an error if any required column is missing or contains out-of-vocabulary values.
- [ ] Build the **condition cross-table** (spatial_label × activity_label × dataset_id) and log row counts before any modeling. This is required for the collinearity check in Layer 4.
- [ ] Tag `reference_flag = True` for all healthy control samples (GSE180759 healthy controls; Lerma-Martin healthy controls). These anchor the fixed reference orbitals.

---

## A) Repo Scaffolding & Provenance

- [ ] `docs/PROVENANCE.md` (URL, access date, license, checksum per file).
- [ ] `scripts/download_manifest.tsv` includes:
  - GSE180759 raw + processed files
  - GSE301908 raw + processed files (snRNA-seq component)
  - Lerma-Martin 2024 data portal links (snRNA + spatial)
- [ ] Pinned environment file (`environment.yml` or lockfile). Include key versions: scanpy, scvi-tools, harmonypy, sklearn, decoupler/UCell, NMF library.
- [ ] Global random seed: set `SEED = 42` in config. All stochastic steps (Harmony, NMF, clustering, data splits) use this seed. Log seed in run metadata.
- [ ] One command runs the full demo: `make demo_ms` or `just demo_ms`. **No manual editing between steps.** All steps must be scriptable; notebooks are for exploration only and must not be required for the reproducible pipeline.

---

## B) Data Ingestion (Exact Datasets)

### GSE180759 (primary training cohort + healthy controls)
- [ ] Download and load into AnnData.
- [ ] Harmonize metadata fields:
  - `donor_id`, `lesion_id`
  - `spatial_label`: map `periplaque/NAWM → NAWM`, `edge/rim → rim_active or rim_inactive`, `core → core_active or core_inactive` (use `lesion_type` to disambiguate activity)
  - `activity_label`: map `chronic active → chronic_active`, `inactive → chronic_inactive`
  - `reference_flag = True` for healthy controls (non-MS donors)
  - `dataset_id = GSE180759`
- [ ] Validate against condition schema (Section 0); fail if any required column is missing.

### GSE301908 (chronic-active rim anchor + external validation)
- [ ] Download and load snRNA-seq component (Immunity 2025 paper; MERFISH component not used in v0).
- [ ] Harmonize donor/lesion metadata:
  - All samples from chronic-active lesions: `activity_label = chronic_active`
  - Assign `spatial_label` based on provided compartment annotations; mark as `unknown` where ambiguous
  - `reference_flag = False` (no healthy controls in this dataset)
  - `dataset_id = GSE301908`
- [ ] Designate as **external validation** (held out from orbital + warp fitting; used for LOSO evaluation).
- [ ] Validate against condition schema.

### Lerma-Martin 2024 (spatial anchor + paired spatial validation)
- [ ] Retrieve snRNA-seq component + spatial transcriptomics metadata.
- [ ] Harmonize niche labels → `spatial_label`. Collapse to: rim (active/inactive), core (active/inactive), NAWM, healthy/control. Store original fine-grained niche labels as `niche_original` for later expansion.
- [ ] For spatial component: store spot x/y coordinates, lesion boundary distances (if available), and histological annotations (iron rim, lipid, etc.) in `.obsm["spatial"]`.
- [ ] `reference_flag = True` for healthy/control samples.
- [ ] `dataset_id = lerma_martin_2024`
- [ ] Validate against condition schema.

---

## C) QC & Preprocessing

- [ ] Per-cell QC: n_genes, total UMIs, pct_mito (if available), ambient RNA markers.
- [ ] Flag likely doublets (DoubletFinder or Scrublet); remove confident doublets, flag borderline.
- [ ] Ensure consistent gene naming (HGNC symbols) across all three datasets. Use a gene name harmonization table.
- [ ] **Pseudobulk construction**: aggregate per donor × cell_family. Apply `min_cells = 50` threshold — groups below 50 cells receive `MISSING` flag and zero confidence. Document removal rate per cell family per dataset.
- [ ] Compute NB variance + DESeq2-style dispersion shrinkage per gene. Store per-gene overdispersion parameter `r_gene`. (Ref: `VERSE_ARCHITECTURE.md`, Layer 1.)
- [ ] **Composition confound check**: for each cell family, run a permutation test of cell-type proportion (fraction of all cells) against `dataset_id`, `spatial_label`, and `activity_label`. Flag families where proportion is significantly predicted by `dataset_id` (p < 0.05, permutation). These families require careful interpretation in warp analysis.

---

## D) Integration & Batch Correction

- [ ] **Cell level**: apply **Harmony** on the cell PCA embedding (before clustering/annotation). This corrects for dataset/chemistry effects in the cell embedding only. Do not Harmony-correct pseudobulk expression matrices.
- [ ] **Pseudobulk level**: add `dataset_id` and `protocol` as fixed-effect covariates in all downstream linear models (program scoring, warp fitting). Do not regress them out of the expression matrix itself.
- [ ] Verify that Harmony-corrected embeddings do not over-mix biological groups: compute silhouette score for `spatial_label` before and after Harmony. Harmony should reduce `dataset_id` silhouette while preserving `spatial_label` silhouette.
- [ ] Document final batch correction strategy in run metadata.

---

## E) Cell Type Annotation

- [ ] Use dataset-provided labels where available (GSE180759, Lerma-Martin have published annotations).
- [ ] For unlabeled or ambiguous cells: run conservative annotation against major classes using marker gene scoring (microglia, astrocyte, OL-lineage, OPC, neuron, endothelial, T cell, other).
- [ ] Store `cell_family` using the stable vocabulary defined in Section 0.
- [ ] Verify that Harmony-corrected UMAP shows expected anatomical/cell-family groupings.

---

## F) Program / Pathway Layer

### F1) Curated program library
- [ ] Assemble v0 program library (~180 programs total):
  - Reactome CNS pathways: download from Reactome API, filter to CNS-relevant pathways (~50)
  - MSigDB Hallmarks: download H collection (~50 gene sets)
  - GO:BP: myelination, lipid metabolism, phagocytosis, complement, ROS (~30 curated subsets)
  - DoRothEA TF regulons: NF-κB, STAT1, STAT3, HIF1A, PPARG, RUNX1, IRF family (~50 regulons; use confidence level A/B only)
- [ ] Score curated programs against pseudobulk µ per cell family. Use UCell or ssGSEA applied to pseudobulk mean expression. Store in `.uns["program_scores_curated"]`.

### F2) Residual NMF programs
- [ ] **Residualization**: for each cell family separately, regress each gene's pseudobulk expression on the curated program scores (OLS); take residuals as input for NMF.
- [ ] **Consensus NMF**:
  - Run NMF 20× with different random seeds on 80% random subsamples of pseudobulk groups.
  - Select K via cophenetic correlation (threshold ≥ 0.95). If no K satisfies this threshold, use K = 10. K floor = 5, K cap = 20.
  - Extract consensus W (gene-program) and H (pseudobulk-program) matrices.
  - Threshold: retain top-50 genes per NMF component (by W weight).
  - Name each NMF program by significant pathway overlap (Fisher's exact test, FDR < 0.1); assign "Unknown-N" if no enrichment.
- [ ] Store NMF programs in `.uns["program_scores_nmf"]`.

### F3) Combined program matrix
- [ ] Concatenate curated + NMF program scores into a single program matrix.
- [ ] **Z-score** each program across all pseudobulk units (per program, across all samples and cell families).
- [ ] Store normalized combined matrix in `.obsm["X_prog"]`.

### F4) Two-level PCA
- [ ] **Global PCA**: fit PCA on all pseudobulk units (all cell families), 20–30 components. Used for: cross-family UMAP, patient-level visualization, sanity checks.
- [ ] **Per-family PCA**: for each cell family separately, fit PCA on that family's pseudobulk program scores. Select number of components by elbow criterion (typically 10–20). These PCAs are used for orbital fitting and warp estimation.
- [ ] Store per-family PCA models and loading matrices W_family for later back-projection.

---

## G) Orbital Fitting

### G1) Initialization (all cell families)
For each cell family, initialize L2 orbitals from **healthy-reference donors only** (reference_flag = True):

| Cell family | L2 states to initialize | Method |
|-------------|------------------------|--------|
| Microglia | homeostatic, DAM-like, IFN-responsive, foamy/ABCA1-low, iron-accumulating | Marker gene scoring → cluster → fit orbital |
| Astrocytes | homeostatic, A1/neuroinflammatory, A2/neuroprotective, pan-reactive, scar | Marker gene scoring → cluster → fit orbital |
| OL-lineage | myelinating OL, pre-myelinating OL, stress/injured OL | Marker gene scoring → cluster → fit orbital |
| OPC | quiescent, activated (PPARG+) | Marker gene scoring → cluster → fit orbital |
| T cells | CD8+ Trm, CD4+ Th17-like, Treg | Marker gene scoring → cluster → fit orbital |
| Neurons | baseline (single orbital; expect minimal warp — **negative control**) | Single Gaussian on healthy neurons |
| Endothelial | homeostatic, activated | Marker gene scoring → cluster → fit orbital |
| Any other | NMF K=3–5 from healthy tissue, labeled State-1..N | Data-driven fallback |

- [ ] For each L2 state, fit a **low-rank + diagonal covariance Gaussian** (Σ = Λ Λ^T + Ψ) in per-family PCA space.
- [ ] **Fallback to diagonal**: if a state has fewer than 3 × n_PCA_components pseudobulk groups, use strict diagonal covariance Σ = diag(σ²).
- [ ] **Anchoring validation**: compute the fraction of healthy-reference cells inside each reference orbital at 99% confidence. This must be ≥ 80% per family before proceeding. If not met, inspect initialization and re-cluster.

### G2) Membership and novelty
- [ ] Compute L2 orbital membership probabilities for all pseudobulk groups (healthy + disease).
- [ ] Compute novelty score using combined variance σ²_sample + σ²_orbital.
- [ ] Apply two novelty bands (95–99%: "extreme known state"; >99%: "truly novel candidate").
- [ ] Flag families with < 100 pseudobulk groups as "uncertainty-limited" in novelty output.
- [ ] **Collinearity check (required here)**: build the spatial_label × activity_label × dataset_id cross-table. Compute design matrix condition number. If condition number > 30, document "spatial and activity axes not separable — collapse to probabilistic lesion axis" and switch to single-axis warp fitting.

---

## H) Warp Fitting

- [ ] For each cell family separately, fit the factorized warp model in per-family PCA space:
  ```
  center_condition = center_healthy + w_spatial·X_spatial + w_activity·X_activity
                    + w_interaction·(X_spatial ⊗ X_activity) + w_dataset·X_dataset
                    + w_sex·X_sex + w_disease_duration·X_disease_duration
  ```
- [ ] **Interaction sparsity rule**: build condition cross-table; fit w_interaction only if ≥ 3 donors per condition cell. Log all skipped interaction terms.
- [ ] **L2 regularization**: minimize reconstruction error + λ·||w||². Select λ by leave-one-donor-out CV.
- [ ] Compute **warp magnitude** = L2 norm of orbital center translation in per-family PCA space. Report scale changes separately.
- [ ] **Back-projection**: project Δ_PCA via W_family^T to recover Δ_program. Rank programs by |Δ_program|. Apply Benjamini-Hochberg FDR correction across programs. Report top-10 FDR-significant programs per warp axis per cell family.
- [ ] **Batch constraint check**: for each cell family, verify that warp magnitude for w_dataset < 50% of warp magnitude for w_spatial or w_activity. If this fails, integration must be revised before interpreting warps.

---

## I) Spatial Validation (Lerma-Martin 2024)

This subsection applies only when spatial transcriptomics data is available.

- [ ] Score each spatial spot/cell with L2 orbital membership (from the trained orbital model).
- [ ] Visualize dominant L2 state per spot in tissue coordinate space. The dominant microglia state should transition from homeostatic (NAWM) → IFN/foamy (rim) → scar/absent (core).
- [ ] Compute **Moran's I** for spatial autocorrelation of orbital membership scores. High Moran's I indicates spatially coherent orbital structure (expected for lesion niches).
- [ ] For foamy/ABCA1-low microglia orbital: correlate orbital density with **distance to histological lesion rim boundary** (from iron staining / histological annotations if available). Foamy microglia density should peak near the rim.
- [ ] For astrocyte scar orbital: density should peak in lesion core. Verify this directional prediction.
- [ ] **Expected high-novelty spots at the rim**: Disease-Associated Radial Glia-like Cells (DARGs — Park & Nicaise, Neuron 2025) are enriched at lesion boundaries and were confirmed in Lerma-Martin re-analysis. Spots near the active rim with `flag_novel = TRUE` and no strong L2 orbital membership may correspond to DARGs. This is **expected correct model behavior** (reference atlas predates DARG characterization), not an artifact. Log the count and location of such spots in the spatial validation report; include a note in the warp report PDF explaining this expected novelty class.
- [ ] Store spatial validation results in `reports/<run_id>/spatial_validation/`.

---

## J) Evaluation Suite (Automated, Per Cell Family)

- [ ] **Leave-one-donor-out (LODO)**:
  - For each fold: train orbital + warp on N-1 donors, apply to held-out donor.
  - Metric: Spearman ρ of warp direction vectors (Δ_PCA per condition) across folds.
  - **Target**: ρ ≥ 0.7 per cell family. If below target, log and analyze root cause (small n, donor outlier, etc.).

- [ ] **Leave-one-dataset-out (LOSO)**:
  - Train on 2 datasets, test on the 3rd (rotate across all three combos).
  - Metric: Jaccard similarity of top-10 warp-driving programs across train/test.
  - **Target**: Jaccard ≥ 0.5 per cell family. Cross-dataset agreement = "structured overlap", not exact identity.

- [ ] **Batch/protocol negative control warp**:
  - Treat `dataset_id` as a pseudo-condition; fit the same warp model.
  - Metric: warp magnitude of dataset_id warp vs biological (spatial/activity) warp.
  - **Target**: batch warp magnitude < 50% of biological warp magnitude. Failure → revise integration.

- [ ] **Anchoring criterion**:
  - **Target**: ≥ 80% of healthy-reference cells inside reference orbitals at 99% confidence (per family).

- [ ] **Neuron negative control**:
  - Compute neuron warp magnitude.
  - **Target**: neuron warp magnitude < 25% of microglia warp magnitude (neurons in WM lesions should show minimal disease-related shift).

- [ ] **Per-family interpretation**: report all metrics stratified by cell family. Interpret relative to sampling depth per family. Tag small families (T cells, endothelial) as "low-power" where n < 100 pseudobulk groups.

---

## K) Output Artifacts (Pilot-Ready)

- [ ] `reports/<run_id>/condition_schema_check.json` — schema validation + cross-table counts
- [ ] `reports/<run_id>/qc_summary.html` — QC metrics and doublet rates per dataset
- [ ] `reports/<run_id>/composition_confound.csv` — per-family permutation test results
- [ ] `reports/<run_id>/program_library.json` — curated program list with gene counts; NMF program names and top genes
- [ ] `reports/<run_id>/orbital_fit.pkl` — fitted orbital parameters (all families)
- [ ] `reports/<run_id>/anchoring_check.csv` — fraction of healthy cells inside orbitals per family
- [ ] `reports/<run_id>/universe_scores.csv` — per sample/donor: dominant_universe, confidence_tier, full posterior vector, flag_novel
- [ ] `reports/<run_id>/orbital_membership.csv` — per pseudobulk group: L2 orbital membership probabilities + novelty score + novelty_band
- [ ] `reports/<run_id>/warp_report.csv` — per cell family × warp axis: warp magnitude, top-10 FDR-significant programs, batch constraint check result
- [ ] `reports/<run_id>/collinearity_check.txt` — condition cross-table + design matrix condition number + collapse decision
- [ ] `reports/<run_id>/eval/lodo_results.csv`, `loso_results.csv`, `negative_control_warps.csv`
- [ ] `reports/<run_id>/spatial_validation/` — spatial maps, Moran's I results, rim distance correlation plots
- [ ] `reports/<run_id>/figures/` — UMAP (global + per-family), orbital plots, warp vector visualizations
- [ ] `reports/<run_id>/ms_warp_report.pdf` — 10–20 slide summary report
