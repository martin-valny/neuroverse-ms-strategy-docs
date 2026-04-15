# Architecture → Implementation Map

This document maps every section of `VERSE_ARCHITECTURE.md` to the corresponding implementation step(s) in `ENGINEERING_CHECKLIST_MS_FIRST.md`. It serves as a consistency check — any architecture concept not covered in the checklist is flagged.

---

## Section-by-Section Map

| Architecture section | Checklist section(s) | v0 status | Notes |
|---------------------|---------------------|-----------|-------|
| **§2 Inputs** — metadata schema, required columns | Checklist §0 | **v0 core** | spatial_label={healthy,NAWM,rim,core,unknown} spatial only; activity_label separate |
| **§2 Inputs** — spatial_label / activity_label schema separation | Checklist §0 | **v0 core** | Schema validator rejects activity suffix in spatial_label (e.g., rim_active invalid) |
| **§3 Layer 1** — pseudobulk granularity (Sample×CellFamily×SpatialLabel) | Checklist §C | **v0 core** | Corrected from v0.3 (was Sample×CellFamily only — broke spatial resolution) |
| **§3 Layer 1** — NB variance + DESeq2 shrinkage | Checklist §C | **v0 core** | NB dispersion fit per gene; MISSING flag |
| **§3 Layer 1** — per-family min_cells thresholds | Checklist §C | **v0 core** | Default=50; T_cell/endothelial=20; neuron=30; T cell is best-effort tier |
| **§4 Layer 2** — Curated program library (~195) | Checklist §F1 | **v0 core** | Reactome CNS + Hallmarks + GO:BP + DoRothEA + PROGENy |
| **§4 Layer 2** — Residual NMF programs | Checklist §F2 | **v0 core** | Consensus NMF, cophenetic K, 20× seeds, top-50 genes per component |
| **§4 Layer 2** — Residualization: ridge regression (not OLS) | Checklist §F2 | **v0 core** | OLS replaced with ridge; guard: skip if n_groups < 2×n_programs |
| **§4 Layer 2** — Z-scoring: global vs per-family (two separate operations) | Checklist §F3 | **v0 core** | Global z-score → global PCA only; per-family z-score → per-family PCA only |
| **§4 Layer 2** — Two-level PCA (global + per-family) | Checklist §F4 | **v0 core** | Global: 20–30 components. Per-family: elbow criterion, 10–20 components |
| **§4 Layer 2** — σ_program not propagated (v0 limitation) | Checklist §F3 | **v0 core** | Program scores from µ only; σ_program gap documented; novelty uses σ²_orbital only |
| **§4 Layer 2** — Back-projection (Δ_PCA → Δ_program) | Checklist §H | **v0 core** | W_family^T × Δ_PCA; rank by effect size |Δ_program| — no BH FDR |
| **§4 Layer 2** — Shift vs identity framing | Checklist §H; §J | **v0 core** | Report within-cell-type contrasts, not absolute levels |
| **§5 Layer 3** — L1/L2 orbital hierarchy | Checklist §G1 | **v0 core** | L1 = cell family; L2 = discovered disease states (not pre-defined inputs) |
| **§5 Layer 3** — Diagonal covariance (v0); low-rank is v1 | Checklist §G1 | **v0 core** | Low-rank + diagonal covariance requires more healthy donors; diagonal is v0 default |
| **§5 Layer 3** — Data-driven orbital fitting from healthy tissue | Checklist §G1 | **v0 core** | No pre-defined disease states; reference orbital = fit from healthy data |
| **§5 Layer 3** — Disease state discovery post-fitting | Checklist §G1b; §H | **v0 core** | Cluster disease warp vectors → annotate against Expected State Annotation Guide |
| **§5 Layer 3** — Anchoring criterion (≥80% at 99% CI) | Checklist §G1; §J | **v0 core** | Validation check on healthy data fit |
| **§5 Layer 3** — Novelty: two bands (95–99%, >99%) | Checklist §G2 | **v0 core** | "Extreme known state" vs "truly novel"; replication ≥2 donors required |
| **§5 Layer 3** — Novelty uses σ²_orbital only (v0) | Checklist §G2 | **v0 core** | σ_program not available in v0; combined variance partial |
| **§5 Layer 3** — Uncertainty-limited flag (< 100 pseudobulk groups) | Checklist §G2 | **v0 core** | Novelty reported but not promoted |
| **§5 Layer 3** — In vitro / iPSC: trajectory distance, not OOD | Not in v0 checklist | **v1** | iPSC = dedicated reference + trajectory scoring |
| **§6 Layer 4** — Healthy tissue as fixed reference | Checklist §G1; §0 | **v0 core** | NAWM = first mild warp, not the baseline |
| **§6 Layer 4** — Factorized warp design matrix | Checklist §H | **v0 core** | w_spatial + w_activity + w_{ij} interaction + w_dataset + sex + disease_duration |
| **§6 Layer 4** — Interaction notation (element-wise indicator, not Kronecker) | Checklist §H | **v0 core** | Σ_{i,j} w_{ij}·1[spatial=i]·1[activity=j]; not a tensor product |
| **§6 Layer 4** — Collinearity check: VIF > 10 (primary); condition number > 30 (fast check) | Checklist §G1; §H | **v0 core** | VIF replaces condition-number-only criterion |
| **§6 Layer 4** — Interaction sparsity rule (≥3 donors/cell) | Checklist §H | **v0 core** | Log all skipped terms |
| **§6 Layer 4** — Nested CV for λ selection (outer=LODO, inner=λ) | Checklist §H | **v0 core** | Prevents data leakage from using same splits for tuning and evaluation |
| **§6 Layer 4** — Donor non-independence documented as v0 limitation | Checklist §H | **v0 core** | Same donor in multiple compartments = not independent; mixed model in v1 |
| **§6 Layer 4** — Warp magnitude definition (L2 norm in PCA space) | Checklist §H; §J | **v0 core** | Scale changes reported separately |
| **§6 Layer 4** — Effect-size ranking (no FDR on back-projection) | Checklist §H | **v0 core** | Back-projection → effect sizes, not p-values; BH FDR invalid here |
| **§6 Layer 4** — Batch constraint (batch warp < 50% biological) | Checklist §H; §J | **v0 core** | Hard check; failure → revise integration |
| **§7 Layer 5** — Confidence-based token weighting | Checklist §C | **v0 core** | < 0.5: excluded; 0.5–0.8: proportional; > 0.8: full weight |
| **§8 Layer 6** — Transformer (deferred) | Not in checklist | **v2+** | Deferred |
| **§9 Layer 7** — Imprint space + multi-task heads (deferred) | Not in checklist | **v2+** | Deferred |
| **§10 Geodesics + discovery modules (deferred)** | Not in checklist | **v2+** | Warps in v0 are already a mechanistic readout |
| **§13 Sample Imprint** — concatenated warp vectors per sample | Checklist §K (sample_imprints.csv) | **v0.5** | Holistic ecosystem signature without transformer |
| **§11 Batch: Harmony cell-level** | Checklist §D | **v0 core** | NOT on pseudobulk |
| **§11 Batch: covariate at pseudobulk/warp level** | Checklist §D; §H | **v0 core** | dataset_id + protocol as fixed effects |
| **§11 Composition confound check** | Checklist §C | **v0 core** | Permutation test per family |
| **§12 Evaluation: LODO ρ ≥ 0.65 aspirational (nested CV)** | Checklist §J | **v0 core** | Primary families only; nested CV required; realistic range 0.50–0.65 with N≈20 |
| **§12 Evaluation: LOSO Jaccard ≥ 0.5 (sanity check, not stat test)** | Checklist §J | **v0 core** | 3 folds → report all 3 individually; qualitative sanity check |
| **§12 Evaluation: primary vs best-effort family tiers** | Checklist §J | **v0 core** | Primary: microglia/astrocyte/OL/OPC; best-effort: T_cell/endothelial/neuron |
| **§12 Evaluation: batch < 50% biological** | Checklist §J | **v0 core** | |
| **§12 Evaluation: anchoring ≥ 80%** | Checklist §J | **v0 core** | |
| **§12 Evaluation: neuron negative control < 25%** | Checklist §J | **v0 core** | |
| **Spatial validation** (Lerma-Martin) | Checklist §I | **v0 core** | Moran's I per discovered state; spatial gradient (Kruskal-Wallis p<0.05); rim-distance correlation as observation; post-hoc annotation |
| **Pipeline targets: demo_ms / run_ms split** | Checklist §A | **v0 core** | demo_ms = downsampled ~30min; run_ms = full production ~12–48h |
| **Orbital model versioning: JSON + npz (not pkl)** | Checklist §A; §K | **v0 core** | Portable format; version + seed + checksums in JSON header |
| **PROGENy (14 perturbation-calibrated pathways)** | Checklist §F1 | **v0 core** | Added to curated library; total ~195 programs; use decoupleR/progeny |
| **Orbital overlap check (Bhattacharyya distance)** | Checklist §G1c; §K | **v0 core** | Pairwise BC between L2 orbitals; BC > 0.5 → flag redundant; output orbital_overlap_matrix.csv |
| **Orbital scaling warp (scale_ratio σ expansion)** | Checklist §H; §K | **v0 core** | scale_ratio = σ_condition / σ_reference per warp axis; added to warp_report.csv |
| **σ_program propagation — delta method formula** | Checklist §F3 (v0 limitation note) | **v1** | Full formula in §4 and FUTURE_VERSIONS_ROADMAP.md §v1; MC alternative documented |
| **CNS cell-state signature library (~20–30 programs)** | Checklist §F1 (v1 placeholder) | **v1** | GSE180759 + GSE301908 + Lerma-Martin + MS literature; planned table in FUTURE_VERSIONS_ROADMAP.md §v1 |
| **Biomarker discovery interface (Mann-Whitney U)** | §10 (v1+ note) | **v1+** | Two-group comparison of program scores + orbital membership; valid FDR context; spec in FUTURE_VERSIONS_ROADMAP.md §v1 |
| **Warp interpolation (trajectory visualization)** | §13 sample imprint | **v0.5** | interp(t) = (1−t)·imprint(A) + t·imprint(B); visualization tool only; spec in FUTURE_VERSIONS_ROADMAP.md §v0.5 |
| **Learnable [MISSING] token for absent cell families** | §8 Layer 6 (v2+ note) | **v2+** | Per-family trainable embedding for absent families; spec in FUTURE_VERSIONS_ROADMAP.md §v2+ |
| **Inter-orbital coupling matrix** | §1 conservative modeling note | **v2+** | Cross-family covariance C_{AB}; requires ~100+ samples; spec in FUTURE_VERSIONS_ROADMAP.md §v2+ |
| **Masked cell type prediction pre-training** | §8 Layer 6 (v2+ note) | **v2+** | BERT-style 15% masking; predicts masked token program score from context; spec in FUTURE_VERSIONS_ROADMAP.md §v2+ |
| **Contrastive learning — SupCon + trajectory triplet** | §9 Layer 7 (v2+ note) | **v2+** | SupCon for condition labels + ranking triplet for spatial ordering; spec in FUTURE_VERSIONS_ROADMAP.md §v2+ |

---

## Concept Orphan Check

The following architecture concepts have NO corresponding checklist item and are either intentionally deferred or require follow-up:

| Concept | Status | Action needed |
|---------|--------|---------------|
| Transformer (Layer 6) | **v2+, intentional** | [MISSING] token + masked prediction spec added to §8; spec in FUTURE_VERSIONS_ROADMAP.md |
| Imprint space (Layer 7) | **v2+, intentional** | Contrastive learning objectives spec added to §9; spec in FUTURE_VERSIONS_ROADMAP.md |
| Geodesics | **v2+, intentional** | None |
| Sample Imprint (§13) | **v0.5** | Warp interpolation added to §13; FUTURE_VERSIONS_ROADMAP.md §v0.5 |
| iPSC trajectory scoring | **v1, intentional** | Add to checklist when iPSC data is available |
| Non-linear warp fields | **v2+, intentional** | None |
| Low-rank + diagonal covariance | **v1** | Spec in FUTURE_VERSIONS_ROADMAP.md §v1 |
| Bootstrap/permutation significance for back-projection | **v0+** | Statistical validation of effect-size ranking |
| Donor random effects (mixed model) | **v1** | Spec in FUTURE_VERSIONS_ROADMAP.md §v1 |
| σ_program propagation (delta method / MC) | **v1** | Full formula now in VERSE_ARCHITECTURE.md §4 + FUTURE_VERSIONS_ROADMAP.md §v1 |
| Rotation + inter-orbital coupling | **v2+, intentional** | Coupling matrix spec in §1 + FUTURE_VERSIONS_ROADMAP.md §v2+ |
| PROGENy pathways | **v0 core** | Added to §F1; covered in checklist and map |
| CNS cell-state signatures | **v1** | Planned table in FUTURE_VERSIONS_ROADMAP.md §v1 |
| Bhattacharyya orbital overlap | **v0 core** | Added to §G1c + §K; covered in checklist and map |
| Orbital scaling warp (scale_ratio) | **v0 core** | Added to §H + §K; covered in checklist and map |
| Biomarker discovery interface | **v1+** | Spec in §10 + FUTURE_VERSIONS_ROADMAP.md §v1 |
| Warp interpolation | **v0.5** | Added to §13; spec in FUTURE_VERSIONS_ROADMAP.md §v0.5 |
| Learnable [MISSING] token | **v2+, intentional** | Spec in §8 + FUTURE_VERSIONS_ROADMAP.md §v2+ |
| Druggability check | **out of scope** | Not in Verse core; noted in FUTURE_VERSIONS_ROADMAP.md out-of-scope section |

**No unintentional orphans detected.** All v0 and v0.5 architecture concepts are covered in the checklist. v1/v2+ concepts are tracked in FUTURE_VERSIONS_ROADMAP.md.

---

## Build Order (Dependency Graph)

```
0. Condition schema + validation (spatial_label/activity_label separation)
   └── A. Repo scaffolding (SEED=42, demo_ms / run_ms, orbital versioning)
         └── B. Data ingestion (GSE180759, GSE301908, Lerma-Martin)
               └── C. QC + pseudobulk (Sample×CellFamily×SpatialLabel)
                         + NB variance + per-family min_cells + composition check
                     └── D. Harmony (cell-level) + batch covariates
                           └── E. Cell type annotation (cell_family vocabulary)
                                 └── F. Program layer (curated + ridge-NMF
                                       + two z-scores + two-level PCA)
                                       ├── G. Orbital fitting (data-driven from healthy,
                                       │     diagonal cov, collinearity VIF check)
                                       │     └── H. Warp fitting (factorized, nested CV,
                                       │           effect-size ranking, post-fitting discovery)
                                       │           ├── I. Spatial validation (Lerma-Martin)
                                       │           └── J. Evaluation (LODO, LOSO sanity,
                                       │                 batch, anchoring, family tiers)
                                       └── K. Output artifacts (JSON+npz, sample imprints)
```

---

## Version Tracking

| File | Version | Last reviewed |
|------|---------|---------------|
| VERSE_ARCHITECTURE.md | v0.7 | 2026-04-15 |
| ENGINEERING_CHECKLIST_MS_FIRST.md | v0.7 | 2026-04-15 |
| FUTURE_VERSIONS_ROADMAP.md | v0.5 | 2026-04-13 |
| STRATEGY_MEMO_MS_FIRST.md | v0.6 | 2026-04-15 |
| PILOT_SOW_MS_FIRST.md | v0.6 | 2026-04-15 |
| VERSE_ARCHITECTURE_DIAGRAM.md | v0.6 | 2026-04-14 |
| VERSE_FOR_MS_ONE_PAGE.md | v0.2 | 2026-04-15 |
| ARCHITECTURE_IMPLEMENTATION_MAP.md | v0.7 | 2026-04-15 |

When either `VERSE_ARCHITECTURE.md` or `ENGINEERING_CHECKLIST_MS_FIRST.md` is updated, review this map for new orphans or coverage gaps.
