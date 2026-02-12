# MS-first Engineering Checklist (Verse v0)

## A) Repo scaffolding & provenance

- [ ] `docs/PROVENANCE.md` (URL, access date, license, checksum per file).
- [ ] `scripts/downloadmanifest.tsv` includes:
  - GSE180759 raw + processed files
  - GSE301908 raw + processed files
  - Lerma-Martin 2024 data portal links (snRNA + spatial)
- [ ] Pinned environment file (`environment.yml` or lockfile).
- [ ] One command runs the demo: `make demo_ms` or `just demo_ms`.

## B) Data ingestion (exact datasets)

### GSE180759

- [ ] Download and load into AnnData.
- [ ] Harmonize metadata fields:
  - donor_id, lesion_id, compartment ∈ {periplaque/NAWM, edge/rim, core}, lesion_type ∈ {chronic active, inactive}, control flag.
- [ ] Create `universe` column:
  - NAWM/periplaque → U0
  - edge/rim → U1
  - core → U2

### GSE301908

- [ ] Download and load snRNA-seq component.
- [ ] Harmonize donor/lesion metadata + lesion compartment labels if provided; map to U0/U1/U2 where possible, otherwise mark as "unknown/other" and use for out-of-domain tests.

### Lerma-Martin 2024

- [ ] Retrieve snRNA-seq + spatial metadata.
- [ ] Harmonize niche labels; map lesion rim/core/perivascular etc.
- [ ] For v0: collapse to rim vs core vs NAWM/control and store original niche labels for later expansion.

## C) QC & preprocessing

- [ ] Standard QC (n_genes, UMIs, pct_mito if available, ambient RNA markers).
- [ ] Remove doublets if possible or at least flag likely doublets.
- [ ] Ensure consistent gene naming (HGNC) across datasets.

## D) Cell-type annotation layer (v0)

- [ ] Use dataset-provided labels where available.
- [ ] Otherwise do conservative annotation for major classes:
  - microglia/macrophage, astrocytes, oligodendrocyte lineage, OPC, neurons, endothelial, T cells.
- [ ] Store `cell_family` as a stable vocabulary.

## E) Program/pathway layer (interpretability)

- [ ] Build a CNS/MS-relevant program library (start small).
- [ ] Compute per-cell or per-pseudobulk program scores.
- [ ] Store program matrix in `.obsm["X_prog"]`.

## F) Orbitals (probabilistic state regions)

- [ ] For each cell_family (start with microglia), define orbitals:
  - option A: coarse states (homeostatic vs reactive vs IFN-like vs lipid/foam-like, etc.)
  - option B: unsupervised clusters, then fit Gaussian orbitals in program space.
- [ ] Compute membership probabilities + novelty/OOD score (distance outside orbitals).

## G) Multiverse warps (minimal)

- [ ] Implement per-universe warp fields:
  - translation (shift vectors per orbital),
  - scale (expansion/contraction).
- [ ] Defer rotation/inter-orbital effects for v0.

## H) Minimum eval suite (automated)

- [ ] Leave-one-donor-out generalization:
  - metrics: universe classification accuracy (U0/U1/U2), stability of top warped orbitals, agreement of top warp-driving programs.
- [ ] Leave-one-dataset-out:
  - train on 2 datasets, test on the third.
- [ ] Negative control warp:
  - treat dataset_id/protocol as condition and compare warp structure/magnitude.
- [ ] Confounder audit:
  - can embedding predict dataset_id too well? if yes, revise integration.

## I) Output artifacts (pilot-ready)

- [ ] `reports/<run_id>/universe_scores.csv` (per sample/donor)
- [ ] `reports/<run_id>/orbital_membership.csv` (per cell or pseudobulk)
- [ ] `reports/<run_id>/warp_report.csv` (top orbitals + top programs per warp)
- [ ] `reports/<run_id>/figures/` (UMAP + orbitals + warp vectors)
- [ ] `reports/<run_id>/ms_warp_report.pdf` (10–20 slides)
