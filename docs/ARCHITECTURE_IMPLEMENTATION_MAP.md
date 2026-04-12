# Architecture → Implementation Map

This document maps every section of `VERSE_ARCHITECTURE.md` to the corresponding implementation step(s) in `ENGINEERING_CHECKLIST_MS_FIRST.md`. It serves as a consistency check — any architecture concept not covered in the checklist is flagged.

---

## Section-by-Section Map

| Architecture section | Checklist section(s) | v0 status | Notes |
|---------------------|---------------------|-----------|-------|
| **§2 Inputs** — metadata schema, required columns | Checklist §0 (Condition Schema) | **v0 core** | Schema enforced as hard prerequisite; checklist §0 defines exact column types and vocabularies |
| **§3 Layer 1** — NB variance + DESeq2 shrinkage | Checklist §C (QC & Preprocessing) | **v0 core** | NB dispersion fit per gene; min_cells=50 hard threshold; MISSING flag |
| **§3 Layer 1** — min_cells=50, confidence scoring | Checklist §C | **v0 core** | Confidence propagated to Layer 5 token weights |
| **§4 Layer 2** — Curated program library (~180) | Checklist §F1 | **v0 core** | Reactome CNS + Hallmarks + GO:BP + DoRothEA; see checklist §F1 for full list |
| **§4 Layer 2** — Residual NMF programs | Checklist §F2 | **v0 core** | Consensus NMF, cophenetic K selection, 20× seeds, top-50 genes per component |
| **§4 Layer 2** — Residualization recipe | Checklist §F2 | **v0 core** | Regress genes on curated scores (OLS), take residuals as NMF input |
| **§4 Layer 2** — Z-scoring before PCA | Checklist §F3 | **v0 core** | Per program, across all pseudobulk units |
| **§4 Layer 2** — Two-level PCA (global + per-family) | Checklist §F4 | **v0 core** | Global: 20–30 components for UMAP. Per-family: elbow criterion, 10–20 components |
| **§4 Layer 2** — Back-projection (Δ_PCA → Δ_program) | Checklist §H (Warp Fitting) | **v0 core** | W_family^T × Δ_PCA; FDR correction on back-projected effect sizes |
| **§4 Layer 2** — Shift vs identity framing | Checklist §H; Checklist §J | **v0 core** | Report within-cell-type contrasts, not absolute levels |
| **§5 Layer 3** — L1/L2 orbital hierarchy | Checklist §G1 | **v0 core** | L1 = cell family (healthy ref); L2 = canonical states per family |
| **§5 Layer 3** — Low-rank + diagonal covariance | Checklist §G1 | **v0 core** | Σ = Λ Λ^T + Ψ; diagonal fallback if < 3 × n_components pseudobulk groups |
| **§5 Layer 3** — Orbital initialization (all families) | Checklist §G1 (anchor table) | **v0 core** | Per-family anchor table with fallback NMF K=3–5 for unlisted families |
| **§5 Layer 3** — Anchoring criterion (≥80% at 99% CI) | Checklist §G1; Checklist §J | **v0 core** | Hard gate before proceeding to warp fitting |
| **§5 Layer 3** — Novelty: two bands (95–99%, >99%) | Checklist §G2 | **v0 core** | "Extreme known state" vs "truly novel"; replication ≥2 donors required |
| **§5 Layer 3** — Novelty: combined variance σ²_sample + σ²_orbital | Checklist §G2 | **v0 core** | Prevents low-confidence groups from generating novelty calls |
| **§5 Layer 3** — Uncertainty-limited flag (< 100 pseudobulk groups) | Checklist §G2 | **v0 core** | Novelty reported but not promoted for sparse families |
| **§5 Layer 3** — In vitro / iPSC: trajectory distance, not OOD | Checklist (not in v0 scope) | **v1** | iPSC microglia = dedicated reference orbital + trajectory scoring |
| **§5 Layer 3** — Technical novelty negative controls | Checklist §J (negative control warp) | **v0 core** | Batch warp magnitude < 50% of biological; also applied to novelty |
| **§6 Layer 4** — Healthy tissue as fixed reference | Checklist §G1 (reference_flag=True); §0 (schema) | **v0 core** | NAWM = first mild warp, not the reference baseline |
| **§6 Layer 4** — Factorized warp design matrix | Checklist §H | **v0 core** | w_spatial + w_activity + w_interaction + w_dataset + sex + disease_duration |
| **§6 Layer 4** — Collinearity check (condition number > 30) | Checklist §G2 (collinearity); §H | **v0 core** | Build cross-table → compute condition number → collapse if near rank-1 |
| **§6 Layer 4** — Interaction sparsity rule (≥3 donors/cell) | Checklist §H | **v0 core** | Log all skipped interaction terms |
| **§6 Layer 4** — L2 regularization + CV | Checklist §H | **v0 core** | Leave-one-donor-out CV to select λ |
| **§6 Layer 4** — Warp magnitude definition (L2 norm in PCA space) | Checklist §H; §J | **v0 core** | Scale changes reported separately |
| **§6 Layer 4** — FDR correction on back-projected programs | Checklist §H | **v0 core** | Benjamini-Hochberg across all programs per family × warp axis |
| **§6 Layer 4** — Batch constraint (batch warp < 50% biological) | Checklist §H; §J | **v0 core** | Hard check; failure → revise integration |
| **§7 Layer 5** — Confidence-based token weighting | Checklist §C (confidence scoring) | **v0 core** | < 0.5: excluded; 0.5–0.8: proportional; > 0.8: full weight |
| **§8 Layer 6** — Transformer (deferred) | Not in checklist | **v2+** | Only implement if required for a specific near-term task |
| **§9 Layer 7** — Imprint space + multi-task heads (deferred) | Not in checklist | **v2+** | Deferred |
| **§10 Geodesics + discovery modules (deferred)** | Not in checklist | **v2+** | Warps in v0 are already a mechanistic readout |
| **§11 Batch: Harmony cell-level** | Checklist §D | **v0 core** | Harmony on cell embedding; NOT on pseudobulk expression |
| **§11 Batch: covariate at pseudobulk/warp level** | Checklist §D; §H | **v0 core** | dataset_id + protocol as fixed effects in scoring and warp models |
| **§11 Composition confound check** | Checklist §C | **v0 core** | Permutation test per family vs dataset_id, spatial_label, activity_label |
| **§12 Evaluation: LODO ρ ≥ 0.7** | Checklist §J | **v0 core** | Per cell family |
| **§12 Evaluation: LOSO Jaccard ≥ 0.5** | Checklist §J | **v0 core** | Per cell family; GSE301908 held out |
| **§12 Evaluation: batch < 50% biological** | Checklist §J | **v0 core** | |
| **§12 Evaluation: anchoring ≥ 80%** | Checklist §J | **v0 core** | |
| **§12 Evaluation: neuron negative control < 25%** | Checklist §J | **v0 core** | |
| **§12 Evaluation: per-family reporting** | Checklist §J | **v0 core** | Tag low-power families |
| **Spatial validation** (Lerma-Martin) | Checklist §I | **v0 core** | Moran's I; foamy microglia density vs rim distance |

---

## Concept Orphan Check

The following architecture concepts have NO corresponding checklist item and are either intentionally deferred or require follow-up:

| Concept | Status | Action needed |
|---------|--------|---------------|
| Transformer (Layer 6) | **v2+, intentional** | None |
| Imprint space (Layer 7) | **v2+, intentional** | None |
| Geodesics | **v2+, intentional** | None |
| iPSC trajectory scoring | **v1, intentional** | Add to checklist when iPSC data is available |
| Non-linear warp fields | **v2+, intentional** | None |
| Full covariance orbitals | **v2+, intentional** | None |
| Rotation + inter-orbital coupling | **v2+, intentional** | None |

**No unintentional orphans detected.** All v0 architecture concepts are covered in the checklist.

---

## Build Order (Dependency Graph)

```
0. Condition schema + validation
   └── A. Repo scaffolding (SEED=42, make demo_ms)
         └── B. Data ingestion (GSE180759, GSE301908, Lerma-Martin)
               └── C. QC + pseudobulk + NB variance + composition check
                     └── D. Harmony (cell-level) + batch covariates
                           └── E. Cell type annotation (cell_family vocabulary)
                                 └── F. Program layer (curated + NMF + PCA)
                                       ├── G. Orbital fitting (L1/L2, all families, collinearity check)
                                       │     └── H. Warp fitting (factorized, L2 reg, FDR, back-projection)
                                       │           ├── I. Spatial validation (Lerma-Martin)
                                       │           └── J. Evaluation suite (LODO, LOSO, batch, anchoring)
                                       └── K. Output artifacts
```

---

## Version Tracking

| File | Version | Last reviewed |
|------|---------|---------------|
| VERSE_ARCHITECTURE.md | v0.3 | 2026-04-09 |
| ENGINEERING_CHECKLIST_MS_FIRST.md | v0.3 | 2026-04-09 |
| STRATEGY_MEMO_MS_FIRST.md | v0.3 | 2026-04-09 |
| PILOT_SOW_MS_FIRST.md | v0.3 | 2026-04-09 |

When either `VERSE_ARCHITECTURE.md` or `ENGINEERING_CHECKLIST_MS_FIRST.md` is updated, review this map for new orphans or coverage gaps.
