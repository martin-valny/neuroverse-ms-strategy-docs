# Verse Architecture: Visual Pipeline Diagram

> Use this diagram for presentations, onboarding, and partner conversations. All technical details are in `VERSE_ARCHITECTURE.md`.

---

## End-to-End Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              INPUTS                                          │
│                                                                              │
│  Single-cell / single-nucleus RNA-seq data                                  │
│  + metadata: donor_id │ spatial_label │ activity_label │ dataset_id         │
│              reference_flag (True = healthy control)                         │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: PROBABILISTIC PSEUDOBULK                                           │
│                                                                              │
│  Aggregate: Sample × CellFamily × SpatialLabel → (µ, σ, n, confidence)     │
│                                                                              │
│  • Uncertainty model: NB variance + DESeq2-style dispersion shrinkage       │
│  • Hard filter: min_cells=50 (default); T_cell/endothelial=20, neuron=30   │
│  • Confidence: f(n, gene overdispersion r_gene)                             │
│  • spatial_label: {healthy, NAWM, rim, core, unknown}  (spatial only)       │
│  • activity_label: {healthy, chronic_active, chronic_inactive, unknown}     │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
┌────────────────────────┐           ┌────────────────────────────┐
│  CURATED PROGRAMS      │           │  RESIDUAL NMF PROGRAMS     │
│  (~195 gene sets)      │           │  (data-driven novelty)     │
│                        │           │                            │
│  • Reactome CNS  ~50   │           │  1. Ridge regression on    │
│  • Hallmarks     ~50   │           │     curated scores (OLS    │
│  • GO:BP         ~30   │           │     replaced; n<2p→skip)   │
│  • DoRothEA TFs  ~50   │           │     NOTE: guard fires for  │
│  • PROGENy        14   │           │     all v0 datasets (n<390)│
│    (perturbation-cal.) │           │  2. Consensus NMF on       │
│                        │           │     residuals (20× seeds)  │
│  UCell / ssGSEA on     │           │  3. K via cophenetic corr  │
│  pseudobulk µ          │           │     (≥0.85–0.95, K=5..20) │
│                        │           │  4. Top-50 genes/component │
└──────────┬─────────────┘           └────────────┬───────────────┘
           │                                      │
           └──────────────┬───────────────────────┘
                          │
                          ▼
              ┌───────────────────────────────────────────┐
              │   Z-SCORE (two separate operations)       │
              │                                           │
              │  Global z-score (all families pooled)     │
              │    → feeds Global PCA only                │
              │                                           │
              │  Per-family z-score (within each family)  │
              │    → feeds Per-family PCA only            │
              │  (do NOT mix these two)                   │
              └───────────────────┬───────────────────────┘
                          │
              ┌───────────┴───────────────────────────────────────┐
              │                                                   │
              ▼                                                   ▼
┌─────────────────────────┐                       ┌─────────────────────────┐
│  GLOBAL PCA             │                       │  PER-FAMILY PCA         │
│  (20–30 components)     │                       │  (10–20 comp, per       │
│                         │                       │   cell family)          │
│  → Cross-family UMAP    │                       │                         │
│  → Patient embedding    │                       │  → Orbital fitting      │
│  → Sanity checks        │                       │  → Warp estimation      │
│                         │                       │  → Novelty scoring      │
└─────────────────────────┘                       └───────────┬─────────────┘
                                                              │
                                                              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: ORBITAL LAYER (L1 / L2 HIERARCHY)                                 │
│                                                                              │
│  L1: Cell family identity orbitals (defined from healthy tissue)             │
│  L2: States discovered post-fitting (data-driven, not pre-defined)           │
│                                                                              │
│  Covariance: Σ = diag(σ²)  (v0; low-rank + diagonal is v1)                 │
│                                                                              │
│  REFERENCE ORBITAL FITTING (healthy tissue only, data-driven):              │
│    • Take all reference_flag=True pseudobulk groups per family              │
│    • Check healthy data for substructure (gap stat / silhouette)            │
│    • Fit K Gaussians if K≥2 clusters detected, else 1 Gaussian             │
│    • That IS the reference orbital — no pre-labeling required               │
│                                                                              │
│  ┌─ Microglia ─ reference orbital (from healthy data) ──────────┐          │
│  │  → disease states: data-driven discovery from warp vectors   │          │
│  │    post-hoc annotation against state reference guide         │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ Astrocytes ─ reference orbital (from healthy data) ─────────┐          │
│  │  → disease states: data-driven discovery from warp vectors   │          │
│  │    (reactive continuum; A1/A2 binary not used)               │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ OL-lineage / OPC / T cells / Endothelial / Neurons ─────────┐          │
│  │  → reference orbital from healthy; disease states from data  │          │
│  │  Neurons: baseline ← NEGATIVE CONTROL (minimal warp expected)│          │
│  └───────────────────────────────────────────────────────────────┘          │
│                                                                              │
│  Outputs: orbital membership probabilities + novelty score                  │
│           (novelty uses σ²_orbital only in v0; σ_program not propagated)   │
│                                                                              │
│  Novelty bands:                                                              │
│    95–99% outside best orbital  →  "Extreme known state"                    │
│    > 99% outside all orbitals   →  "Truly novel" (if replicated ≥2 donors) │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: FACTORIZED WARP FIELDS                                             │
│                                                                              │
│                   HEALTHY TISSUE (fixed reference)                           │
│                          │                                                   │
│                   ┌──────┴──────┐                                            │
│                   │  w_spatial  │  compartment axis (NAWM / rim / core)      │
│                   └──────┬──────┘                                            │
│                          │◄──── × ────►                                     │
│                   ┌──────┴──────┐                                            │
│                   │ w_activity  │  lesion activity (chronic-active/inactive) │
│                   └──────┬──────┘                                            │
│                          │◄──── + ────►                                     │
│                   ┌──────┴──────┐                                            │
│                   │w_{ij}       │  per (spatial_i, activity_j) pair         │
│                   │             │  = element-wise indicator product          │
│                   │             │  (NOT a Kronecker product)                 │
│                   └──────┬──────┘                                            │
│                          │◄──── + ────►                                     │
│                   ┌──────┴──────┐                                            │
│                   │ w_dataset   │  technical covariate (not biological)      │
│                   └─────────────┘                                            │
│                                                                              │
│  Collinearity: VIF > 10 for any axis → collapse to probabilistic lesion axis│
│  (condition number > 30 = fast first check; VIF is definitive criterion)    │
│                                                                              │
│  L2 reg: L = fit_error + λ‖w‖²   (λ via nested CV: outer=LODO, inner=λ)   │
│  Warp magnitude:   ‖Δ_PCA‖₂  per cell family per warp axis                 │
│  Back-projection:  Δ_program = W_family^T × Δ_PCA                          │
│                    → rank by |Δ_program| (effect size)                      │
│                    (no BH FDR — effect sizes, not p-values)                 │
│  Batch constraint: batch warp magnitude < 50% of biological warp            │
│                                                                              │
│  Post-fitting: cluster disease warp vectors → annotate states               │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 5: TOKEN CONSTRUCTION                                                 │
│                                                                              │
│  Per Sample × CellFamily token = program scores + orbital membership        │
│                                  + warped positions + metadata               │
│                                                                              │
│  Confidence-based weighting:                                                 │
│    confidence < 0.5  →  EXCLUDED from token set                             │
│    confidence 0.5–0.8 → downweighted proportionally                         │
│    confidence > 0.8  →  full weight                                          │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
              ┌─────────────────┴────────────────────┐
              │  v0 outputs                          │  v2+ (deferred)
              ▼                                      ▼
┌────────────────────────────┐         ┌────────────────────────────┐
│  OUTPUTS (v0)              │         │  v0.5 / v2+                │
│                            │         │                            │
│  • Universe scores         │         │  v0.5: Sample Imprint      │
│    (U0/U1/U2 + posterior   │         │  imprint(s) = [Δ_µglia |   │
│     + confidence tier)     │         │   Δ_astro | Δ_OL | ...]    │
│  • Orbital membership      │         │  → cluster for holistic    │
│  • Warp report             │         │    ecosystem signature     │
│    (top programs by        │         │  (no transformer needed)   │
│     effect size,           │         │                            │
│     warp magnitude)        │         │  v2+: Transformer imprint  │
│  • Sample imprints (v0.5)  │         │  Geodesic trajectories     │
│  • Spatial validation      │         │  Response prediction       │
│    (Moran's I, rim dist)   │         │                            │
│  • Evaluation metrics      │         └────────────────────────────┘
│    (LODO, LOSO, batch)     │
└────────────────────────────┘
```

---

## Condition Axes (MS Example)

```
                        ACTIVITY AXIS
                  chronic-active ◄──────► chronic-inactive
                        │                      │
                        │                      │
  S    NAWM  ───────────┼──────────────────────┼──── (mild warp from healthy)
  P           w0        │                      │
  A           ┌─────────▼──────────────────────▼─────────────┐
  T           │                                               │
  I    RIM    │  ACTIVE RIM           INACTIVE RIM           │
  A    (U1)   │  (foamy µglia,         (DAM µglia,           │
  L           │   CD8+Trm, IFN↑,       iron↑, less IFN)     │
              │   lipid↑)                                     │
  A           └─────────┬──────────────────────┬─────────────┘
  X                     │                      │
  I    CORE    ┌────────▼──────────────────────▼─────────────┐
  S    (U2)   │  ACTIVE CORE          INACTIVE CORE          │
              │  (scar astrocyte,      (chronic scar,         │
              │   OL injury)           remyelinating OL↑)    │
              └────────────────────────────────────────────────┘
```

The v0 reporting surface collapses this 2D grid to 3 labels (U0, U1, U2). The underlying factorized warp model retains the full 2D resolution.

---

## Novelty Scoring Schematic

```
Program space (per-family PCA)

          ╔════════════════════════════╗
          ║  99% confidence region     ║   ← Truly novel if outside here
          ║  ╔══════════════════════╗  ║      AND replicated ≥2 donors
          ║  ║  95% CI             ║  ║
          ║  ║  ┌──────────────┐  ║  ║   ← Extreme known state (95–99%)
          ║  ║  │   ORBITAL    │  ║  ║
          ║  ║  │   CENTER     │  ║  ║
          ║  ║  └──────────────┘  ║  ║
          ║  ╚══════════════════════╝  ║
          ╚════════════════════════════╝

  •  Inside 95%  →  "in orbital"  (normal, expected state)
  •  95%–99%     →  "extreme known state" (tail, not novel)
  •  > 99%       →  "truly novel candidate" (if replicated)
```

---

## Data Flow Summary

```
Raw scRNA-seq (3 datasets)
       ↓ Harmony (cell-level batch correction)
Cell type annotation (cell_family vocabulary)
       ↓ Pseudobulk aggregation (min_cells=50)
Program scoring (curated + NMF) → Z-score (global for global PCA;
                                  per-family for per-family PCA) → PCA
       ↓ Per-family orbital fitting (healthy reference only, data-driven)
Warp fitting (factorized linear model: spatial + activity, nested CV)
       ↓ Back-projection → rank by |Δ_program| effect size (no FDR)
       ↓ Post-fitting disease state discovery (cluster warp vectors)
Universe scores + orbital membership + warp report + sample imprints
       ↓ Evaluation (LODO, LOSO, batch check, spatial validation)
Deliverable outputs (CSV + PDF + reproducible pipeline)
```

---

*Version: v0.6 — last updated 2026-04-14. Key changes from v0.5: orbital layer box updated — named disease states removed from microglia and astrocyte boxes; replaced with "data-driven discovery from warp vectors / post-hoc annotation against state reference guide."*
