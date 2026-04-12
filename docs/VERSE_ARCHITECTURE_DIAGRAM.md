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
│  Aggregate: Sample × CellFamily → (µ, σ, n, confidence)                    │
│                                                                              │
│  • Uncertainty model: NB variance + DESeq2-style dispersion shrinkage       │
│  • Hard filter: min_cells = 50   (below → MISSING flag, zero confidence)    │
│  • Confidence: f(n, gene overdispersion r_gene)                             │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
┌────────────────────────┐           ┌────────────────────────────┐
│  CURATED PROGRAMS      │           │  RESIDUAL NMF PROGRAMS     │
│  (~180 gene sets)      │           │  (data-driven novelty)     │
│                        │           │                            │
│  • Reactome CNS  ~50   │           │  1. Regress genes on       │
│  • Hallmarks     ~50   │           │     curated scores         │
│  • GO:BP         ~30   │           │  2. Consensus NMF on       │
│  • DoRothEA TFs  ~50   │           │     residuals (20× seeds)  │
│                        │           │  3. K via cophenetic corr  │
│  UCell / ssGSEA on     │           │     (≥0.95, K=5..20)       │
│  pseudobulk µ          │           │  4. Top-50 genes/component │
└──────────┬─────────────┘           └────────────┬───────────────┘
           │                                      │
           └──────────────┬───────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Z-SCORE             │
              │   per program,        │
              │   across all          │
              │   pseudobulk units    │
              └───────────┬───────────┘
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
│  L2: Canonical state orbitals per family (warps operate here)                │
│                                                                              │
│  Covariance: Σ = Λ Λᵀ + Ψ  (low-rank + diagonal)                           │
│  Fallback:   Σ = diag(σ²)  (if < 3×n_components pseudobulk groups)         │
│                                                                              │
│  ┌─ Microglia ───────────────────────────────────────────────────┐          │
│  │  homeostatic │ DAM-like │ IFN-responsive │ foamy │ iron       │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ Astrocytes ──────────────────────────────────────────────────┐          │
│  │  homeostatic │ A1/neuroinflammatory │ A2 │ pan-reactive │ scar│          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ OL-lineage ──────────────────────────────────────────────────┐          │
│  │  myelinating │ pre-myelinating │ stress/injured               │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ OPC ─────────────────────────────────────────────────────────┐          │
│  │  quiescent │ activated (PPARG+)                               │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ T cells ─────────────────────────────────────────────────────┐          │
│  │  CD8+ Trm │ CD4+ Th17-like │ Treg                            │          │
│  └───────────────────────────────────────────────────────────────┘          │
│  ┌─ Neurons ─────────────────────────────────────────────────────┐          │
│  │  baseline  ← NEGATIVE CONTROL (minimal warp expected)         │          │
│  └───────────────────────────────────────────────────────────────┘          │
│                                                                              │
│  Outputs: membership probabilities + novelty score per pseudobulk group     │
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
│                   │w_interaction│  spatial × activity (if ≥3 donors/cell)   │
│                   └──────┬──────┘                                            │
│                          │◄──── + ────►                                     │
│                   ┌──────┴──────┐                                            │
│                   │ w_dataset   │  technical covariate (not biological)      │
│                   └─────────────┘                                            │
│                                                                              │
│  Collinearity check: if rank(design) ≈ 1 → collapse to probabilistic       │
│                       lesion axis + document                                 │
│                                                                              │
│  L2 regularization: L = fit_error + λ‖w‖²    (λ by LODO cross-validation)  │
│  Warp magnitude:     ‖Δ_PCA‖₂  per cell family per warp axis               │
│  Back-projection:    Δ_program = W_family^T × Δ_PCA  (FDR-corrected)       │
│  Batch constraint:   batch warp magnitude < 50% of biological warp          │
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
│  OUTPUTS (v0)              │         │  LAYERS 6–7 (v2+)          │
│                            │         │                            │
│  • Universe scores         │         │  • Transformer over tokens │
│    (U0/U1/U2 + posterior   │         │  • Patient "imprint"       │
│     + confidence tier)     │         │  • Multi-task heads        │
│  • Orbital membership      │         │  • Geodesic trajectories   │
│  • Warp report             │         │  • Response prediction     │
│    (top programs, FDR,     │         │                            │
│     warp magnitude)        │         └────────────────────────────┘
│  • Spatial validation      │
│    (Moran's I, rim dist)   │
│  • Evaluation metrics      │
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
Program scoring (curated + NMF) → Z-score → PCA
       ↓ Per-family orbital fitting (healthy reference only)
Warp fitting (factorized linear model: spatial + activity)
       ↓ Back-projection → named programs → FDR
Universe scores + orbital membership + warp report
       ↓ Evaluation (LODO, LOSO, batch check, spatial validation)
Deliverable outputs (CSV + PDF + reproducible pipeline)
```
