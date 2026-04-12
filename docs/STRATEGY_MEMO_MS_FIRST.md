# NeuroVerse / Verse v0 Strategy Memo (MS-first)

## One-Sentence Wedge

Prove that Verse can factor the **spatial** and **inflammatory** axes of CNS white matter pathology into a principled, uncertainty-aware orbital state space — and score any new sample into that factored space with calibrated uncertainty — in a way no standard scRNA-seq analysis pipeline can match.

## Why MS-First

MS lesion pathology provides crisp, spatially grounded condition labels (core vs rim/edge vs NAWM) that are ideal for testing the Verse bending/warping hypothesis without needing longitudinal biopsies. Importantly, these spatial labels are **orthogonal** to a second biological axis — lesion activity (chronic-active vs inactive) — giving us a natural factorized design to demonstrate that Verse can resolve multiple condition axes simultaneously.

MS also has three complementary public datasets covering different biological angles, enabling rigorous cross-dataset validation.

## Architecture: Model vs Reporting Surface

> The underlying Verse model factors lesion location (spatial axis) and lesion activity (activity axis) as **orthogonal warp axes**, with interaction terms where data support them. The U0/U1/U2 three-universe classification is a **v0 reporting abstraction** layered on this richer factorized model. Partners see the simplified classification; scientists see the full warp decomposition.

**Reference chain:**
```
Healthy tissue (fixed orbital reference)
  └── NAWM                (mild warp w0 — already disease-shifted, not a true baseline)
        ├── Active Rim     (w_spatial + w_activity + w_interaction)
        ├── Inactive Rim   (w_spatial + w_activity)
        ├── Active Core    (w_spatial + w_activity + w_interaction)
        └── Inactive Core  (w_spatial + w_activity)
```

The factorized warp model supports **5 biologically meaningful universes**: NAWM, active-rim, inactive-rim, active-core, inactive-core. In v0, these collapse to 3 reporting labels (U0=NAWM, U1=rim, U2=core), but the underlying model retains full resolution.

## Anchor Datasets and Their Roles

| Dataset | Role | Key content |
|---------|------|-------------|
| **GSE180759** (Absinta/Maric et al.) | **Primary training cohort + healthy reference** | snRNA-seq from progressive MS autopsy brains: lesion core, edge, periplaque + healthy controls; chronic active vs inactive context |
| **Lerma-Martin 2024** (Nat Neurosci) | **Spatial validation anchor** | Paired snRNA-seq + spatial transcriptomics of subcortical MS lesions: rim, core, perivascular niches across chronic active and chronic inactive stages |
| **GSE301908** (Immunity 2025) | **Chronic-active rim anchor + external validation** | snRNA-seq + MERFISH of chronic active MS lesions; defines the foamy-microglia / CD8+ T cell inflammatory niche at the rim (ABCA1/G1 axis) |

**Cross-dataset strategy**: GSE180759 + Lerma-Martin used for training (with Harmony integration); GSE301908 held out as external validation. Rotate all three in leave-one-dataset-out evaluation. Target: structured overlap of top warp-driving programs across datasets — not exact identity (different protocols, donors, and regions will cause some divergence).

## Cell Type Scope: All Families, Microglia as Primary Readout

Verse operates across **all major cell families** present in white matter. Microglia is the primary scientific readout and validation story for v0, but the architecture is not microglia-only.

**Expected biology across cell families:**

| Cell family | Spatial axis (rim vs core vs NAWM) | Activity axis (active vs inactive) |
|-------------|-----------------------------------|------------------------------------|
| **Microglia** | Foamy/ABCA1-low + IFN states peak at active rim; homeostatic in NAWM | Chronic-active: foamy + CD8 niche; Inactive: DAM-like, iron-accumulating |
| **Astrocytes** | Pan-reactive and A1 (neuroinflammatory) at rim; scar-associated in core | Chronic-active: stronger A1 signature; Inactive: gradual scar transition |
| **OL-lineage** | Stress/injured OL increases rim→core; myelinating OL depleted in core | Inactive lesions show remyelinating OL attempts more than active |
| **OPC** | Activated OPC (PPARG+, differentiation-impaired) at lesion rim | Chronic-active: OPC blocked from differentiating; Inactive: partial recovery |
| **T cells** | CD8+ Trm concentrated at active rim (GSE301908 CD8 niche) | Chronic-active only; minimal in inactive |
| **Neurons** | Axonal depletion in core; near-zero warp in WM = **negative control** | Should not shift with activity |
| **Endothelial** | Activated at rim; homeostatic in NAWM | Chronic-active: BBB disruption signals |

## Microglia Sub-State Detail (Primary Readout)

Within the rim/active universe, microglia are heterogeneous. Define sub-orbitals:

| Sub-state | Key markers | Expected location | Anchor |
|-----------|------------|-------------------|--------|
| Homeostatic | P2RY12, CX3CR1, TMEM119 | NAWM | Healthy reference |
| DAM-like | TREM2, APOE, SPP1 | NAWM → rim transition | MS literature |
| IFN-responsive | ISG15, MX1, IFIT1–3 | Active rim, perivascular | Absinta; Lerma-Martin |
| Foamy / ABCA1-G1-low | ABCA1↓, ABCG1↓, LIPA, CD68, PLIN2 | Active rim (lipid accumulation) | GSE301908 (Immunity 2025) |
| Iron-accumulating | FTH1, FTLL1, SLC40A1 | Chronic active / inactive rim | MS literature; Absinta |

The foamy/ABCA1-low state and its spatial co-localization with CD8+ Trm T cells (the "pathogenic inflammatory niche" in GSE301908) is the strongest mechanistic anchor for the chronic-active rim universe.

## Core Scientific Hypotheses (Testable with v0)

1. **Spatial hypothesis**: Cell-type/state orbitals shift progressively from healthy → NAWM → rim → core, and this spatial warp is reproducible across donors and datasets.
2. **Activity hypothesis**: Chronic-active and chronic-inactive lesions at the same spatial compartment have different orbital shifts (especially at the rim), driven by distinct programs (IFN + lipid in active; iron + scar in inactive).
3. **Factorizability**: The spatial and activity warps are approximately orthogonal and can be separated in the design matrix, enabling independent interpretation.
4. **Mechanism**: The top warp-driving programs at the rim are IFN-response and lipid/sterol metabolism; the top programs at the core are scar-associated gliosis and myelin loss. These should replicate across datasets.

## v0 Outputs (Sellable Artifacts)

- **Lesion state space**: reproducible MS orbital atlas (UMAP + program axes) with NAWM/rim/core universes for all major cell families.
- **Warp mechanism report**: per cell family — which orbitals shift the most, which programs drive each shift (spatial axis + activity axis separately), warp magnitude rankings.
- **Universe classification API/notebook**: input a new snRNA sample or pseudobulk → output: dominant universe (U0/U1/U2), confidence tier (high/medium/low), full posterior vector, novel flag.
- **Spatial validation**: orbital membership projected into tissue coordinates (Lerma-Martin spatial data), confirming that foamy microglia orbital density peaks at the histological rim boundary.

## What We Defer in v0

- Full transformer / patient imprint and geodesic trajectory analysis (requires richer longitudinal structure)
- iPSC microglia as a query domain (exploratory only — see below)
- Perivascular niche, remyelinating lesion, and grey matter lesion universes
- Inactive lesion sub-orbital expansion (v1)

## iPSC Microglia Roadmap (Exploratory, v1)

iPSC microglia lack yolk-sac lineage developmental imprinting and differ metabolically from tissue microglia. They will score as high novelty relative to tissue orbitals for non-biological reasons. Therefore:

- **Do not** apply absolute novelty scoring to iPSC microglia in the current orbital reference.
- **Do** build a dedicated **iPSC-microglia reference orbital** (from baseline iPSC microglia without inflammatory stimulus).
- **Score** iPSC microglia under inflammatory cues as **trajectory distance toward tissue-microglia rim orbitals**, not as OOD novelty.
- Mark all iPSC results as exploratory; do not include in v0 quantitative evaluation suite.

## Covariates and Confounds

- **Sex and disease duration** should be included as covariates in warp fitting. MS prevalence and progression differ by sex, and sex differences can masquerade as disease warp signal.
- **Donor age** as an additional covariate where available.
- **Composition confounds**: verify that cell-type proportions are not significantly predicted by `dataset_id` before interpreting warps (see `ENGINEERING_CHECKLIST_MS_FIRST.md` Section C).

## Evaluation Suite (Non-Negotiable)

1. **Leave-one-donor-out (LODO)**: learn orbitals + warps on N-1 donors, apply to held-out donor. Warp direction vectors (Spearman ρ ≥ 0.7) and key programs must persist. Evaluate per cell family.
2. **Leave-one-dataset-out (LOSO)**: train on 2 datasets, test on the 3rd (rotate). Top-10 warp-driving programs must overlap (Jaccard ≥ 0.5). Goal: structured overlap, not exact identity.
3. **Negative control warps**: treat `dataset_id/protocol` as a pseudo-condition; show that dataset warp magnitude < 50% of biological warp magnitude.
4. **Anchoring check**: ≥ 80% of healthy-reference cells inside reference orbitals at 99% confidence, per family.
5. **Neuron negative control**: neuron warp magnitude < 25% of microglia warp magnitude.
6. **Interpretability check**: top warp-driving programs at the active rim must include IFN-response and/or lipid/sterol metabolism (literature-anchored expectation). Document which known biology is recovered.

## Roadmap After v0

1. **v1**: Inactive lesion sub-orbital expansion; perivascular niche universe; sex-stratified warp analysis; OPC differentiation trajectory. **Also v1**: Alsema et al. 2024 WM spatial data as a third independent spatial validation dataset (see "Datasets Evaluated and Deferred" below).
2. **v2**: Transformer imprint space; geodesic trajectory modeling (pre/post treatment); non-linear warp fields.
3. **v2+**: iPSC-microglia trajectory scoring as a drug-target validation tool; multi-omic fusion (ATAC + RNA). DARG orbital modeling if functional in vitro data matures (see below).

---

## Datasets Evaluated and Deferred

The following datasets were explicitly considered and decided against for v0. Record of reasoning kept here to avoid re-litigating these decisions.

### Alsema et al. 2024 (Nature Neuroscience) — Deferred to v1

**What it is**: Spatial transcriptomics atlas of both **white matter and gray matter** MS lesions, post-mortem human brain. Re-analyzed (alongside Lerma-Martin 2024) by Park & Nicaise 2025 (Neuron). Covers >200 tissue sections from a large multi-site cohort.

**Why not v0**:
- Gray matter lesions are a fundamentally different biology (cortical layering, neuronal content, different cell-type composition, different lesion progression dynamics). Adding GM would require a separate universe ontology — cortical demyelination does not fit the NAWM→rim→core WM axis.
- Training on combined WM + GM spatial data would contaminate WM orbitals: microglia homeostatic state in white matter ≠ microglia in cortex.
- Three datasets already stress v0 scope. Adding a fourth adds harmonization cost without improving the core WM warp story.

**Why v1**: The **white matter sections only** from Alsema 2024 could serve as a third independent spatial validation dataset (alongside Lerma-Martin), significantly strengthening LOSO spatial claims. Bookmark the GEO accession when confirmed and revisit for v1 spatial extension.

**Action**: Find and record accession number. Do not download or integrate until v1 scope is confirmed.

---

### Park & Nicaise 2025 (Neuron, DOI: 10.1016/j.neuron.2025.09.022) — Not included

**What it is**: Characterizes **DARGs (Disease-Associated Radial Glia-like Cells)** — a novel senescent, IRF1/IFN-driven glial population in progressive MS brain. Primary new data is **iNSC (induced neural stem cells)** from patient fibroblasts (snRNA-seq + snATAC-seq + proteomics). Spatial validation is done by re-analyzing Lerma-Martin 2024 and Alsema 2024 (datasets already evaluated above) — no new primary tissue spatial data is generated.

**Why not included**:
1. **iNSC data is in vitro** — same fundamental problem as iPSC microglia (see iPSC section above). Reprogrammed fibroblasts lack tissue-niche imprinting. They will show high novelty vs. tissue orbitals for non-biological reasons and cannot anchor orbital states.
2. **DARGs are not in the current L2 orbital hierarchy** (radial glia-like cells are absent from the anchor table). Defining a reliable DARG orbital requires primary tissue data, not iNSC validation.
3. **Senescence/SASP is a genuine new axis** but out of v0 scope. Adding senescence programs to the curated library would be meaningful — but this is scope creep that dilutes the rim/core factorized warp story being proven in v0.
4. **No new spatial data** — the spatial validation in the paper reuses Lerma-Martin 2024, which is already in the plan.

**One important operational note for spatial validation**: DARGs are enriched near lesion boundaries / active rim areas (confirmed in Lerma-Martin re-analysis). When running §I spatial validation, spots near the rim that receive `flag_novel = TRUE` may correspond to DARGs. **This is expected correct behavior**, not an artifact — the reference atlas predates DARG characterization. Document this in the novel-spot interpretation section of the warp report.

**If v1/v2**: If a dedicated DARG tissue snRNA-seq dataset becomes available (not iNSC), adding a DARG L2 sub-orbital under a new "radial glia-like / senescent glia" L1 family would be meaningful, particularly combined with senolytic treatment response data.
