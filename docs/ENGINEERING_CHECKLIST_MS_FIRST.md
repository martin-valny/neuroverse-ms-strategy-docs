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
| `spatial_label` | str | {healthy, NAWM, rim, core, unknown} | **Spatial position only** — do NOT encode activity here |
| `activity_label` | str | {healthy, chronic_active, chronic_inactive, unknown} | **Lesion activity only** — intersection (e.g., chronic-active rim) is derived from spatial_label + activity_label |
| `cell_family` | str | {microglia, astrocyte, OL_lineage, OPC, T_cell, neuron, endothelial, other} | Stable cell family vocabulary |
| `n_cells` | int | ≥ 50 (after filtering) | Cell count in this pseudobulk group |
| `confidence` | float | [0, 1] | NB-derived confidence score |

- [ ] Write a schema validation function that reads in AnnData `.obs` (or pseudobulk DataFrame) and raises an error if any required column is missing or contains out-of-vocabulary values. Explicitly reject any `spatial_label` value containing an activity suffix (e.g., `rim_active`, `core_inactive`) — these are invalid under the v0.4 schema.
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
- [ ] Two pipeline targets:
  - `make demo_ms` — downsampled fast run: 1,000 cells/donor, microglia family only, skip LOSO, skip spatial validation. Should complete in ~30 min. Used for quick smoke-testing.
  - `make run_ms` — full production run: all families, all donors, full LODO + LOSO + spatial validation. Document expected wall time (~12–48h depending on hardware) in README.
  **No manual editing between steps.** All steps must be scriptable; notebooks are for exploration only.
- [ ] Orbital model versioning: store trained orbital models as **JSON + numpy arrays** (not pickle). Format: `orbital_fit_v{version}_{run_id}.json` + `orbital_weights_v{version}_{run_id}.npz`. Include model version, training dataset IDs, seed, and data checksums in the JSON header. `orbital_fit.pkl` is deprecated — pickle files are Python-version dependent and not portable.
- [ ] Add orbital model version field to `docs/PROVENANCE.md` template.

---

## B) Data Ingestion (Exact Datasets)

### GSE180759 (primary training cohort + healthy controls)
- [ ] Download and load into AnnData.
- [ ] Harmonize metadata fields:
  - `donor_id`, `lesion_id`
  - `spatial_label`: map `periplaque/NAWM → NAWM`, `edge/rim → rim`, `core → core` (spatial only — activity encoded separately)
  - `activity_label`: derive from `lesion_type` — chronic active → `chronic_active`, inactive → `chronic_inactive`
  - Confirm `activity_label` values match schema: `{healthy, chronic_active, chronic_inactive, unknown}`
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
- [ ] Harmonize niche labels → `spatial_label` (spatial only: `{healthy, NAWM, rim, core, unknown}`) and `activity_label` separately. Store original fine-grained niche labels as `niche_original` for later expansion.
- [ ] For spatial component: store spot x/y coordinates, lesion boundary distances (if available), and histological annotations (iron rim, lipid, etc.) in `.obsm["spatial"]`.
- [ ] `reference_flag = True` for healthy/control samples.
- [ ] `dataset_id = lerma_martin_2024`
- [ ] Validate against condition schema.

---

## C) QC & Preprocessing

- [ ] Per-cell QC: n_genes, total UMIs, pct_mito (if available), ambient RNA markers.
- [ ] Flag likely doublets (DoubletFinder or Scrublet); remove confident doublets, flag borderline.
- [ ] Ensure consistent gene naming (HGNC symbols) across all three datasets. Use a gene name harmonization table.
- [ ] **Pseudobulk construction**: aggregate per **donor × cell_family × spatial_label**. Each pseudobulk row = one donor × one cell family × one spatial compartment. Apply per-family `min_cells` thresholds:
  - default: 50 cells
  - T_cell, endothelial: 20 cells
  - neuron: 30 cells

  Groups below the threshold receive `MISSING` flag and zero confidence. Document removal rate per cell family per dataset. **T cell warp analysis is best-effort**: report only if ≥ 5 donors have ≥ 20 T cells per compartment; otherwise report cell counts and exclude from warp analysis.
- [ ] Compute NB variance + DESeq2-style dispersion shrinkage per gene. Store per-gene overdispersion parameter `r_gene` in `.uns["nb_dispersion"]` (per cell family). **v1 dependency**: σ_program propagation (delta method) requires `r_gene` — verify this artifact is present in v0 outputs before v1 development begins. (Ref: `VERSE_ARCHITECTURE.md`, Layer 1; `FUTURE_VERSIONS_ROADMAP.md` §v1.)
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
- [ ] **Cross-dataset label harmonization**: map each dataset's published cell-type annotations to the canonical `cell_family` vocabulary. Document the mapping explicitly:

  | Dataset | Published label examples | → `cell_family` |
  |---------|------------------------|-----------------|
  | GSE180759 | Microglia, MG, Mic | microglia |
  | GSE180759 | Astrocytes, Astro | astrocyte |
  | GSE180759 | Oligodendrocytes, Oligo, MOL | OL_lineage |
  | GSE180759 | OPC, COP | OPC |
  | GSE180759 | Neurons, ExN, InN | neuron |
  | GSE180759 | Endothelial, Endo | endothelial |
  | GSE180759 | T cells, CD8, CD4 | T_cell |
  | Lerma-Martin | (map published labels similarly) | ... |
  | GSE301908 | (map published labels similarly) | ... |

  **Note**: complete this table after downloading each dataset's published annotation vocabulary. Ambiguous labels (e.g., "myeloid" which could be microglia or macrophage) must be resolved using marker gene expression, not assumed.

- [ ] For unlabeled or ambiguous cells: run conservative annotation against major classes using marker gene scoring (microglia, astrocyte, OL-lineage, OPC, neuron, endothelial, T cell, other).
- [ ] Store `cell_family` using the stable vocabulary defined in Section 0.
- [ ] Verify that Harmony-corrected UMAP shows expected anatomical/cell-family groupings.

---

## F) Program / Pathway Layer

### F1) Curated program library
- [ ] Assemble v0 program library (~195 programs total):
  - Reactome CNS pathways: download from Reactome API, filter to CNS-relevant pathways (~50)
  - MSigDB Hallmarks: download H collection (~50 gene sets)
  - GO:BP: myelination, lipid metabolism, phagocytosis, complement, ROS (~30 curated subsets)
  - DoRothEA TF regulons: NF-κB, STAT1, STAT3, HIF1A, PPARG, RUNX1, IRF family (~50 regulons; use confidence level A/B only)
  - **PROGENy** (14 perturbation-calibrated signaling pathways): JAK-STAT, NFkB, TGFb, MAPK, PI3K, VEGF, WNT, EGFR, Androgen, Estrogen, Hypoxia, p53, Trail, TNF. Use `decoupleR` R package (`run_progeny()`) or the `progeny` Python port. PROGENy is preferred over GO:BP for detecting active signaling (perturbation-calibrated, not just gene co-expression).
- [ ] Score curated programs against pseudobulk µ per cell family. Use UCell or ssGSEA applied to pseudobulk mean expression. Store in `.uns["program_scores_curated"]`.
- [ ] *(v1 only, skip in v0)* CNS cell-state signatures (~20–30 domain-specific gene sets from GSE180759, GSE301908, Lerma-Martin 2024, MS literature). See `FUTURE_VERSIONS_ROADMAP.md §v1`.

### F2) Residual NMF programs
- [ ] **Residualization (ridge regression)**:
  - First check: if `n_pseudobulk_groups < 2 × n_curated_programs` for a cell family → **skip residualization**, run NMF directly on curated program score matrix. Log a WARNING.
  - **Expected behavior**: with ~195 curated programs, the guard triggers when n < 390 pseudobulk groups. With the three v0 datasets (GSE180759 ≈ 160 groups/family, Lerma-Martin ≈ 32, GSE301908 ≈ 18), **this guard will fire for essentially every cell family in every dataset**. This is expected and correct behavior for v0 — the curated program score matrix is a compact, meaningful input for NMF even without residualization.
  - Otherwise (future data with n ≥ 390): for each cell family separately, regress each gene's pseudobulk expression on the curated program scores using **L2-penalized (ridge) regression** (not OLS). Take residuals as input for NMF.
- [ ] **Consensus NMF**:
  - Run NMF 20× with different random seeds on 80% random subsamples of pseudobulk groups.
  - Select K via cophenetic correlation. Revised thresholds (0.95 is too strict for n < 200):
    - cophenetic ≥ 0.95: use K with highest cophenetic (very stable, clean signal)
    - cophenetic in [0.85, 0.95): use K with highest cophenetic ≥ 0.85 (satisfactory)
    - cophenetic < 0.85 for all K: use K=10 as fallback. Log WARNING with actual max cophenetic value. Note in warp report that NMF stability was low for this family.
  - K floor = 5, K cap = 20.
  - Extract consensus W (gene-program) and H (pseudobulk-program) matrices.
  - Threshold: retain top-50 genes per NMF component (by W weight).
  - Name each NMF program by significant pathway overlap (Fisher's exact test, FDR < 0.1); assign "Unknown-N" if no enrichment.
- [ ] Store NMF programs in `.uns["program_scores_nmf"]`.

### F3) Combined program matrix
- [ ] Concatenate curated + NMF program scores into a single program matrix.
- [ ] **Global z-score**: z-score each program across ALL pseudobulk units (per program, all families pooled). Store as `.obsm["X_prog_global"]`. Used only for global PCA.
- [ ] **Per-family z-score**: for each cell family, z-score each program within that family's pseudobulk subset only. Store per-family matrices separately. Used for per-family PCA. **Do NOT feed the global z-scored matrix into per-family PCA** — global z-scoring before per-family PCA allows cell-family identity to dominate within-family embeddings.
- [ ] Document in run metadata: both z-score scopes computed and stored separately.
- [ ] Also note: in v0, program scores are computed from pseudobulk µ only. σ_program is not propagated through UCell/ssGSEA. Novelty scoring downstream uses σ²_orbital only (acknowledged v0 limitation).

### F4) Two-level PCA
- [ ] **Global PCA**: fit PCA on `.obsm["X_prog_global"]` (all cell families, globally z-scored), 20–30 components. Used for: cross-family UMAP, patient-level visualization, sanity checks.
- [ ] **Per-family PCA**: for each cell family separately, fit PCA on that family's within-family z-scored program scores. Select number of components by **kneedle algorithm** (automatic knee/elbow detection) on the cumulative variance explained curve, with a floor of 10 and a cap of 20. If kneedle selects < 10 components, use 10; if > 20, cap at 20. As a fallback: use the smallest K that explains ≥ 90% cumulative variance. Document the chosen method and K per family in run metadata. These PCAs are used for orbital fitting and warp estimation.
- [ ] Store per-family PCA models and loading matrices W_family for later back-projection.

---

## G) Orbital Fitting

### G1) Reference orbital fitting (data-driven from healthy tissue)

**Do not pre-define disease states as initialization inputs.** Reference orbitals are fit purely from healthy data; disease states are discovered post-fitting.

For each cell family:
- [ ] Select all pseudobulk groups with `reference_flag=True` for this family.
- [ ] Check for substructure in healthy data: run gap statistic or silhouette criterion on within-family PCA space to determine if ≥ 2 well-separated clusters exist.
  - If yes (K ≥ 2 clusters): fit K diagonal-covariance Gaussians (K from data, not assumed).
  - If no (single cluster): fit a single diagonal-covariance Gaussian.
- [ ] **That is the reference orbital.** No pre-labeling of states required.
- [ ] **Covariance model (v0)**: diagonal covariance Σ = diag(σ²). Low-rank + diagonal (Σ = Λ Λ^T + Ψ) is a v1 extension requiring more healthy donors.
- [ ] **Anchoring validation**: compute the fraction of healthy-reference pseudobulk groups inside reference orbitals at 99% confidence. Must be ≥ 80% per family before proceeding. If not met, inspect the healthy data clustering and adjust K.

**Collinearity check (required here)**:
- [ ] Build the spatial_label × activity_label × dataset_id cross-table.
- [ ] Compute condition number of design matrix as fast first check. If > 30, proceed to VIF.
- [ ] Compute VIF for each warp axis (spatial, activity, dataset).
- [ ] **If any VIF > 10 → collapse protocol** (implement as follows):
  1. Fit a single ordered `w_lesion_stage` warp using the natural disease progression ordering: `healthy (0) < NAWM (1) < rim_inactive (2) < rim_active (3) < core_inactive (4) < core_active (5)`. Drop separate `w_spatial`, `w_activity`, and interaction terms.
  2. This is a univariate linear warp along the lesion axis — it captures disease severity but loses directional separation (spatial ≠ activity).
  3. Log `collapse_reason = "VIF > 10 for [axis]: spatial/activity axes not separable"` in `collinearity_check.txt`.
  4. Report collapsed warp magnitude and top programs per lesion_stage increment.
  5. Do NOT attempt to re-introduce interaction terms after collapse — they cannot be identified if the main axes are already collinear.
  6. **Mechanistic impact note**: document in the warp report that collapse means "we can describe *how much* cell states change along the lesion gradient, but cannot separate *location-specific* from *activity-specific* drivers."

### G1b) Disease state discovery (post-warp-fitting step — see §H)
After warp vectors are estimated (§H), cluster disease pseudobulk warp vectors per family, annotate clusters against the Expected Disease State Annotation Guide in `VERSE_ARCHITECTURE.md §5`. This is a post-fitting discovery step, not an input to orbital fitting.

- [ ] **Clustering algorithm**: use **k-means** with K selected via **gap statistic** (Tibshirani et al. 2001). K range: 2–8 per cell family. If the gap statistic plateau is ambiguous, use silhouette score as a tiebreaker (choose K that maximizes mean silhouette). Alternative: hierarchical clustering (Ward linkage) with dendrogram cut at silhouette-maximizing K. Document algorithm choice and K per family in run metadata.

### G1c) Orbital overlap check (Bhattacharyya distance)
After L2 state orbitals are fitted (or discovered post-fitting), verify they are sufficiently distinct:
- [ ] For each cell family: compute pairwise Bhattacharyya coefficient (BC) between all L2 orbital pairs:
  ```
  D_B(A, B) = (1/8)(µ_A − µ_B)^T Σ_avg^{−1} (µ_A − µ_B) + (1/2) ln(|Σ_avg| / sqrt(|Σ_A|·|Σ_B|))
  BC = exp(−D_B)
  ```
- [ ] **Flag any pair with BC > 0.5** as "potentially redundant states." Log to `orbital_overlap_matrix.csv` (per family). Apply the following decision rule:
  - **BC > 0.7** (substantial overlap): merge the two states → reduce K by 1 and re-fit. These states are not meaningfully distinct.
  - **0.5 < BC ≤ 0.7** (moderate overlap): keep separate but annotate in warp report: "States X and Y partially overlap (BC=0.NN) — may represent a continuum rather than discrete states." Do not re-fit.
  - **BC ≤ 0.5** (distinct): no action needed.
- [ ] Document final merge decisions in run metadata.
- [ ] Store: `reports/<run_id>/orbital_overlap_matrix.csv` — pairwise BC per cell family (rows/cols = discovered state labels).

### G2) Membership and novelty
- [ ] Compute orbital membership probabilities for all pseudobulk groups (healthy + disease).
- [ ] Compute novelty score using σ²_orbital only (v0 limitation: σ_program not propagated from Layer 2; see §F3 note).
- [ ] Apply two novelty bands (95–99%: "extreme known state"; >99%: "truly novel candidate"; promote only if replicated ≥ 2 donors).
- [ ] Flag families with < 100 pseudobulk groups as "uncertainty-limited" in novelty output.

---

## H) Warp Fitting

- [ ] For each cell family separately, fit the factorized warp model in per-family PCA space:
  ```
  center_condition = center_healthy
                    + w_spatial · X_spatial
                    + w_activity · X_activity
                    + Σ_{i,j} w_{ij} · 1[spatial=i] · 1[activity=j]   ← interaction
                    + w_dataset · X_dataset
                    + w_sex · X_sex
                    + w_disease_duration · X_disease_duration
  ```
  Note: the interaction term is an element-wise indicator product per (spatial_bin, activity_bin) pair — not a Kronecker product of the full design matrices.
- [ ] **Interaction sparsity rule**: fit interaction terms only if ≥ 3 donors per (spatial, activity) combination. Log all skipped terms. **Expected v0 outcome**: with N≈20 total and the ≥3 donors rule, most interaction terms will be skipped. v0 warp analysis will primarily capture main effects (spatial + activity); interaction analysis requires larger cohorts or partner data.
- [ ] **L2 regularization + nested CV**: minimize reconstruction error + λ·||w||². Select λ using **nested cross-validation**:
  - Outer loop: LODO — hold out donor D for evaluation.
  - Inner loop: for each outer fold, select λ by CV on the N-2 remaining donors.
  - Do NOT use the same LODO splits for both λ selection and evaluation.
- [ ] Compute **warp magnitude** = L2 norm of orbital center translation in per-family PCA space. Report scale changes separately.
- [ ] **Back-projection + effect-size ranking**: project Δ_PCA via W_family^T to recover Δ_program. Rank programs by |Δ_program| (effect size). Report top-10 programs per warp axis per cell family. **Do NOT apply BH FDR to these effect sizes** — back-projection produces effect sizes, not p-values; FDR is statistically invalid here. Statistical significance via bootstrap or permutation is a v0+ extension.
- [ ] **Orbital scaling warp (scale_ratio)**: for each cell family × warp axis, compute `scale_ratio = σ_condition / σ_reference` (geometric mean across PCA dimensions). scale_ratio > 1 = orbital expansion (more heterogeneous state); scale_ratio < 1 = orbital compression ("locked" state). Append `scale_ratio` as a column in `warp_report.csv` alongside `warp_magnitude`.
- [ ] **Donor non-independence note**: log in run metadata that pseudobulk groups from the same donor across different spatial compartments are not statistically independent. v0 treats them as independent (standard formulation); v1 will add donor_id as a random effect.
- [ ] **Batch constraint check**: for each cell family, verify that warp magnitude for w_dataset < 50% of warp magnitude for w_spatial or w_activity. Failure → revise integration before interpreting warps.
- [ ] **Batch-confounded program check**: for each top-10 warp-driving program per axis, compute Pearson r with batch covariates (dataset_id, protocol, chemistry). Flag programs with |r| > 0.3 as "potentially batch-confounded." Exclude flagged programs from the primary interpretation; report them separately in an annotation note. This ensures the final report's top programs are batch-resistant.
- [ ] **Interaction fit log**: output `interaction_fit_report.csv` — per cell family × (spatial_i, activity_j) pair: n_donors, fitted (Y/N), w_{ij} value (if fitted), reason skipped (if not). Partners and reviewers can see which interaction terms had sufficient data.
- [ ] **Post-fitting disease state discovery** (§G1b): cluster disease warp vectors per family; annotate clusters against the Expected Disease State Annotation Guide. **Run clustering before consulting the guide** (blinded clustering) — compare to guide only after cluster labels are assigned.

---

## I) Spatial Validation (Lerma-Martin 2024)

This subsection applies only when spatial transcriptomics data is available.

- [ ] Score each spatial spot/cell with L2 orbital membership (from the trained orbital model).
- [ ] Visualize dominant L2 state per spot in tissue coordinate space.
- [ ] **Spatial gradient check (primary validation target — structure only, state-agnostic)**:
  - For each discovered L2 orbital state per cell family, compute its **spatial profile**: mean orbital membership score per `spatial_label` (healthy, NAWM, rim, core).
  - Run a **Kruskal-Wallis test** on orbital membership scores across spatial labels. At least one discovered state per cell family should show p < 0.05 after the Kruskal-Wallis test. This confirms that discovered states are spatially organized — regardless of which state shows the gradient or in which direction.
  - The gradient direction (which spatial label has the highest membership score for a given state) is an **observation, not a requirement**. Do not pre-specify which state should peak at which spatial label; report what the data show.
  - If no discovered state shows a significant spatial gradient for a given family, flag as "spatially unorganized" and investigate: (a) insufficient cell counts per spatial compartment? (b) poor batch correction masking spatial signal? (c) genuine biology — this family's states are not spatially organized?
- [ ] **Moran's I spatial autocorrelation**: for each discovered L2 state per cell family, compute Moran's I on orbital membership scores using spatial coordinates. High Moran's I (> 0.1) indicates spatially coherent orbital structure (expected for lesion niches). Report Moran's I per state — this is a structural check independent of which state is present.
- [ ] **Rim distance correlation (structure only)**: if lesion boundary distance annotations are available (from iron staining or histological annotations in Lerma-Martin), correlate **each discovered L2 state's orbital density** with distance to lesion rim boundary. Report the correlation coefficients and their directions for each state as observations — do not pre-specify which state should be rim-proximal or rim-distal. Record which states show significant (p < 0.05) positive or negative correlations with rim proximity.
- [ ] **NAWM-in-orbit check**: compute what fraction of NAWM pseudobulk groups fall **inside the healthy reference orbital** at 99% confidence per cell family. Report this fraction separately from the anchoring check (which uses only healthy donors). If < 20% of NAWM falls inside the healthy orbit, NAWM is a disease state in this dataset — document this and adjust warp interpretation accordingly (NAWM is not a "mild warp" but a substantial disease shift). The specific threshold at which NAWM partially vs fully separates from healthy is an empirical finding to report, not a target to enforce.
- [ ] **Warp directionality validation (top-5 programs, structure only)**: for each top-5 warp-driving program per warp axis, overlay the raw program score heatmap on tissue coordinates. For each program, check:
  - Does this program show spatial coherence? (Moran's I > 0.1) ✓/✗
  - Is the gradient direction consistent with the warp direction? (programs driving the rim warp should score higher at rim than NAWM) ✓/✗
  - **Do NOT pre-specify which program names should appear.** Report what programs actually drive each warp axis by effect size. If any program shows an unexpected spatial pattern (e.g., a rim-warp-driving program scores highest at NAWM), investigate: (a) spatial label assignment reversed? (b) program scoring inverted? (c) unexpected biology? Document findings.
  - **Post-hoc only**: after the directionality check is complete, consult the Expected Disease State Annotation Guide (`VERSE_ARCHITECTURE.md §5`) to identify whether top warp-driving programs correspond to known MS biology. This mapping is done last — it annotates what was found, it does not define what should be found.
- [ ] **Expected high-novelty spots at the rim**: Disease-Associated Radial Glia-like Cells (DARGs — Park & Nicaise, Neuron 2025) are enriched at lesion boundaries and were confirmed in Lerma-Martin re-analysis. Spots near the active rim with `flag_novel = TRUE` and no strong L2 orbital membership may correspond to DARGs. This is **expected correct model behavior** (reference atlas predates DARG characterization), not an artifact. Log the count and location of such spots in the spatial validation report; include a note in the warp report PDF explaining this expected novelty class.
- [ ] Store spatial validation results in `reports/<run_id>/spatial_validation/`.

---

## J) Evaluation Suite (Automated, Per Cell Family)

**Family tiers for evaluation**:
| Tier | Families | Evaluation target applies? |
|------|----------|--------------------------|
| Primary | microglia, astrocyte, OL_lineage, OPC | Yes — all metrics are hard targets |
| Best-effort | T_cell, endothelial, neuron | Report if data permit; not hard gates |

- [ ] **Leave-one-donor-out (LODO)** — use nested CV (outer LODO for evaluation, inner CV for λ selection; see §H):
  - For each outer fold: train orbital + warp on N-1 donors, evaluate on held-out donor.
  - Metric: Spearman ρ of warp direction vectors (Δ_PCA per condition) across folds.
  - **Target (primary families)**: mean ρ ≥ 0.65 per cell family (aspirational). With N ≈ 20 total donors, warp estimates are inherently noisy — mean ρ ≈ 0.50–0.65 is the realistic expected range. ρ ≥ 0.65 is a good outcome, not a minimum requirement.
  - **Power note**: with N=20, each LODO fold trains on 19 donors. For a 6-dimensional warp space, this is modestly powered but not underpowered. Heterogeneous donors (atypical warp direction) will produce low ρ for those specific folds — report the distribution of ρ across folds, not just the mean.
  - If mean ρ < 0.50 for a primary family: log root cause — is the warp direction inconsistent across donors (biological heterogeneity), or is the warp signal too small to detect (effect size issue)?

- [ ] **Leave-one-dataset-out (LOSO)**:
  - Train on 2 datasets, test on the 3rd (rotate across all three combos).
  - Metric: Jaccard similarity of top-10 warp-driving programs across train/test.
  - **Expected (primary families)**: Jaccard ≥ 0.5 in ≥ 2 of 3 folds.
  - **Important**: LOSO produces 3 fold values — this is a **qualitative sanity check**, not a statistical test. Report all 3 fold Jaccard values individually. Do not collapse to a single pass/fail. Do not make generalization claims based on 3 folds.
  - **LOSO asymmetry**: GSE301908 contains only chronic-active lesions (no variance on the activity axis). When GSE301908 is the held-out fold, LOSO validates spatial-axis warp only — activity-axis programs cannot be tested in that fold. Annotate each fold's results with which warp axes were testable.

- [ ] **Batch/protocol negative control warp**:
  - Treat `dataset_id` as a pseudo-condition; fit the same warp model.
  - Metric: warp magnitude of dataset_id warp vs biological (spatial/activity) warp.
  - **Target**: batch warp magnitude < 50% of biological warp magnitude. Failure → revise integration.

- [ ] **Anchoring criterion**:
  - **Target**: ≥ 80% of healthy-reference pseudobulk groups inside reference orbitals at 99% confidence (per family).

- [ ] **Neuron negative control**:
  - Compute neuron warp magnitude per spatial compartment.
  - **Target (NAWM and rim)**: neuron warp magnitude < 25% of microglia warp magnitude. In white matter, axons dominate and surviving neuron cell bodies are sparse — near-zero warp is expected here.
  - **Core compartment**: do NOT apply the 25% target at the lesion core. Surviving neurons in the core may show stress signatures (HSPA1A, ATF3 upregulation, RBFOX3 downregulation) — a small warp at the core is biologically plausible, not a validation failure. Report core neuron warp separately without a hard target.

- [ ] **Sparse family realistic coverage note**: document in the run report the actual pseudobulk group counts per family per dataset after applying min_cells thresholds. Expected outcomes based on dataset composition:
  - T cells / endothelial: most pseudobulk groups will have < 20 cells per donor per spatial label in white matter (typical WM T-cell density ≈ 1–5 cells per mm²). Expect 30–60% MISSING flags. Warp analysis requires ≥ 5 donors meeting the threshold — document whether this is achieved.
  - Neurons: very sparse in WM; most groups will fall below min_cells=30. Primarily useful as a negative control, not a mechanistic readout.
  - These outcomes are expected, not failures. Document actual n per family in `reports/<run_id>/pseudobulk_coverage_summary.csv`.

- [ ] **Per-family interpretation**: report all metrics stratified by cell family and tier. Tag best-effort families (n < 100 pseudobulk groups) as "low-power" in output.

---

## K) Output Artifacts (Pilot-Ready)

- [ ] `reports/<run_id>/condition_schema_check.json` — schema validation + cross-table counts
- [ ] `reports/<run_id>/qc_summary.html` — QC metrics and doublet rates per dataset
- [ ] `reports/<run_id>/composition_confound.csv` — per-family permutation test results
- [ ] `reports/<run_id>/nb_dispersion/` — per cell family: per-gene NB overdispersion parameter `r_gene` (stored as `.uns["nb_dispersion"]` in AnnData and as `r_gene_{cell_family}.csv` for portability). **v1 dependency** for σ_program propagation.
- [ ] `reports/<run_id>/program_library.json` — curated program list with gene counts; NMF program names and top genes
- [ ] `reports/<run_id>/orbital_fit_v{version}_{run_id}.json` + `orbital_weights_v{version}_{run_id}.npz` — fitted orbital parameters (all families). JSON header includes: model version, training dataset IDs, SEED, data checksums. (Replaces deprecated `orbital_fit.pkl`.)
- [ ] `reports/<run_id>/anchoring_check.csv` — fraction of healthy cells inside orbitals per family
- [ ] `reports/<run_id>/universe_scores.csv` — per sample/donor: dominant_universe, confidence_tier, full posterior vector, flag_novel
- [ ] `reports/<run_id>/orbital_membership.csv` — per pseudobulk group: L2 orbital membership probabilities + novelty score + novelty_band
- [ ] `reports/<run_id>/warp_report.csv` — per cell family × warp axis: `warp_magnitude` (L2 norm of center shift), `scale_ratio` (σ expansion/contraction), top-10 programs by |Δ_program| effect size, batch constraint check result
- [ ] `reports/<run_id>/orbital_overlap_matrix.csv` — pairwise Bhattacharyya coefficient (BC) between L2 discovered state orbitals, per cell family; BC > 0.5 flagged as potentially redundant
- [ ] `reports/<run_id>/sample_imprints.csv` — per sample: concatenated per-family warp vectors (imprint), cluster assignment, UMAP coordinates (v0.5 feature)
- [ ] `reports/<run_id>/pseudobulk_coverage_summary.csv` — per cell family × dataset × spatial_label: n_groups, n_groups_passing (above min_cells), n_groups_missing, pct_missing. Critical transparency artifact for sparse families.
- [ ] `reports/<run_id>/interaction_fit_report.csv` — per cell family × (spatial_i, activity_j): n_donors, fitted (Y/N), w_{ij} if fitted
- [ ] `reports/<run_id>/collinearity_check.txt` — condition cross-table + design matrix condition number + VIF values + collapse decision (if triggered)
- [ ] `reports/<run_id>/eval/lodo_results.csv`, `loso_results.csv`, `negative_control_warps.csv`
- [ ] `reports/<run_id>/spatial_validation/` — spatial maps, Moran's I results, rim distance correlation plots
- [ ] `reports/<run_id>/figures/` — UMAP (global + per-family), orbital plots, warp vector visualizations
- [ ] `reports/<run_id>/ms_warp_report.pdf` — 10–20 slide summary report

---

*Version: v0.7 — last updated 2026-04-15. Key changes from v0.6: LOSO asymmetry note added (§J); r_gene storage specified (§C, §K) as v1 dependency; interaction DOF warning added (§H); cell-type cross-dataset mapping table added (§E); PCA elbow criterion formalized (§F4, kneedle algorithm); clustering algorithm specified (§G1b, k-means + gap statistic); partner validation protocol added to pipeline note.*
