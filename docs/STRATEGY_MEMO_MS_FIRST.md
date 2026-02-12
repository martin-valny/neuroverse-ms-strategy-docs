# NeuroVerse / Verse v0 Strategy Memo (MS-first)

## One-sentence wedge

Build a lesion-niche "multiverse" for progressive MS white matter and quantify how cell-type/state orbitals warp across three universes—NAWM, rim/edge, core—then use this state space to score and map new samples (and later iPSC microglia) into lesion states with uncertainty.

## Why MS-first (for you)

MS lesion pathology gives crisp, spatially grounded condition labels (core vs edge/rim vs periplaque/NAWM) that are ideal for testing the Verse "bending/warping" hypothesis without needing longitudinal biopsies.

## Anchor datasets (exact)

1) **GSE180759** (Absinta/Maric et al.): snRNA-seq from 5 progressive MS autopsy brains sampling lesion **core**, **edge**, and **periplaque**, plus healthy controls; includes chronic active vs inactive lesion context.
2) **GSE301908**: snRNA-seq component for "Single-cell spatial transcriptomic profiling defines a pathogenic inflammatory niche in chronic active MS lesions" (paired with MERFISH in the paper).
3) **Lerma‑Martin 2024 (Nat Neurosci)**: paired snRNA-seq + spatial transcriptomics atlas mapping **lesion rim**, **lesion core**, perivascular and other niches across chronic active vs inactive stages in subcortical MS lesions.

## Define the first 3 "universes"

We explicitly operationalize "condition-specific space" (ImmuneVerse multiverse) as lesion compartments:

- **Universe U0: NAWM/periplaque (reference)**
  Purpose: anchor "least perturbed" white matter state.
- **Universe U1: Rim/edge**
  Purpose: compartmentalized inflammation and active microglia niches.
- **Universe U2: Core**
  Purpose: chronic damage, scar-associated states.

These universes are directly aligned with the ImmuneVerse notion that each condition defines a warped orbital space relative to a healthy/reference frame.

## Core scientific hypothesis ("bending")

Cell-type/state orbitals (especially microglia, astrocytes, oligodendrocyte lineage) occupy probabilistic regions in program space, and the mapping from NAWM → rim → core is best described as condition-specific warp fields that shift/deform those orbitals.

## Minimal product v0 (8 weeks)

### Output artifacts (sellable)

- A reproducible MS lesion state space (UMAP/latent + program axes) with NAWM/rim/core universes.
- A "warp report":
  - Which orbitals shift the most from NAWM→rim and rim→core,
  - Which programs/pathways drive each shift,
  - A novelty/OOD score for samples that do not match any known orbital.
- A "lesion scoring API/notebook":
  - Input: a new snRNA sample or pseudobulk,
  - Output: probability of belonging to NAWM/rim/core universes and the dominant cell-state programs.

### What we defer

- Full transformer/patient imprint and geodesic trajectory analysis (needs richer longitudinal structure).

## Minimum eval suite (non-negotiable)

1) **Leave-one-donor-out**: learn orbitals+warps on N-1 donors, apply to held-out donor; rim/core separation and key warp programs must persist.
2) **Leave-one-dataset-out**: train on GSE180759 + Lerma-Martin; test on GSE301908 (and vice versa) to prove warps aren't dataset artifacts.
3) **Negative control warps**: define "protocol/batch" as a fake condition; show disease/lesion warps are distinct (or detect confounding).
4) **Interpretability checks**: top warp-driving programs should be coherent and match known lesion biology (interferon/lipid handling at rim niches is one example emphasized in chronic active lesion work).

## Roadmap after v0

- Add more universes (perivascular niche, inactive lesion, remyelinating).
- Add iPSC microglia as a query domain: map iPSC under inflammatory cues into rim-like microglia orbitals and quantify gaps.
