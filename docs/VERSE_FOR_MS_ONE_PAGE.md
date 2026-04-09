# Verse for MS: One-Page Summary

> For partner conversations, investor decks, and scientific collaborations.

---

## The Problem

Multiple sclerosis white matter lesions are **not one thing**. A lesion rim is biologically distinct from its core, and a chronic-active lesion is distinct from an inactive one. Standard single-cell RNA-seq analysis treats these as groupings to compare — but cannot:

- **Factor** spatial position and inflammatory activity simultaneously as orthogonal axes
- **Quantify** how each cell type's state shifts along each axis with calibrated uncertainty
- **Score** any new sample into that factored state space without retraining
- **Distinguish** genuine novel biology from technical noise at the right confidence level

The result: teams get long lists of differentially expressed genes, no unified mechanistic model, and findings that don't replicate across cohorts.

---

## The Verse Approach

Verse builds a **probabilistic orbital state space** — a living atlas where each cell type/state occupies a probabilistic "orbital" in pathway/program space. Conditions are modeled as **warp fields** that deform these orbitals in a factorized, interpretable way.

```
Healthy tissue (fixed reference orbitals)
    └── NAWM                (mild pre-lesion warp)
          ├── Active Rim    (spatial warp + inflammatory warp + interaction)
          ├── Inactive Rim  (spatial warp + minimal inflammatory signal)
          ├── Active Core   (spatial + inflammatory warps)
          └── Inactive Core (spatial warp + scar transition)
```

**What makes it different:**

| Feature | Standard analysis | Verse |
|---------|------------------|-------|
| Reference | Per-study baseline | Fixed healthy tissue anchor |
| Condition model | One-at-a-time comparisons | Factorized orthogonal axes |
| Cell type scope | Usually one focus cell type | All major families simultaneously |
| Uncertainty | Point estimates | Propagated NB variance + confidence weighting |
| Novelty detection | Unsupervised clustering artifact | Calibrated χ² threshold + replication requirement |
| Cross-dataset | Batch correction then ignore | Explicit batch covariates; validates structured overlap |

---

## MS-First Proof Point

Three complementary public datasets:
- **GSE180759** — progressive MS autopsy: core / edge / periplaque / healthy controls
- **Lerma-Martin 2024** (Nat Neurosci) — paired snRNA-seq + spatial: chronic active vs inactive lesions
- **GSE301908** (Immunity 2025) — foamy microglia / CD8+ T cell inflammatory niche at the active rim

Together these cover the full spatial × activity lesion design, enabling rigorous leave-one-dataset-out validation.

**Key scientific findings the model should recover:**
- Foamy (ABCA1/G1-low) microglia + CD8+ Trm T cell co-localization at the active rim
- IFN-response programs peak at the active rim; lipid/sterol metabolism is disrupted
- Scar-associated astrocytes dominate the core; remyelinating OL attempts are more prominent in inactive lesions
- These warp signals are reproducible across datasets (structured overlap, not exact identity)

---

## v0 Deliverables

**1. MS lesion orbital atlas** — reproduced from 3 public datasets; covers microglia, astrocytes, OL-lineage, OPC, T cells, neurons, endothelial.

**2. Factorized warp report** — spatial axis + activity axis decomposed separately; top programs per axis per cell family (FDR-corrected); warp magnitude rankings.

**3. Universe classification** — for any new sample: U0/U1/U2 label + confidence tier (high/medium/low) + full posterior + novel flag.

**4. Spatial validation** — orbital membership projected into tissue coordinates; foamy microglia density correlated with histological rim boundary distance.

---

## Quantitative Targets (v0)

| What | Target |
|------|--------|
| Leave-one-donor-out warp reproducibility (Spearman ρ) | ≥ 0.7 |
| Leave-one-dataset-out program overlap (Jaccard) | ≥ 0.5 |
| Batch warp vs biological warp | < 50% |
| Healthy reference cells inside orbitals | ≥ 80% |

---

## Roadmap

- **v0** (now): MS lesion orbital atlas + factorized warp report + universe classification
- **v1**: iPSC microglia trajectory scoring; perivascular niche; sex-stratified warp; inactive sub-orbital expansion
- **v2**: Transformer patient imprint; geodesic trajectory modeling (pre/post treatment response)
- **v2+**: Multi-omic fusion; non-linear warp fields; full foundation model vibes

---

## One-Sentence Pitch

Verse factors the spatial and inflammatory axes of MS lesion pathology into a unified, uncertainty-aware orbital state space — and scores any new sample into that space with calibrated confidence.

---

*Full technical specification: `VERSE_ARCHITECTURE.md` | MS strategy: `STRATEGY_MEMO_MS_FIRST.md` | Pilot terms: `PILOT_SOW_MS_FIRST.md`*
