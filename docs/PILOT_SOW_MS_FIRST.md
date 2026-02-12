# Pilot SOW: MS Lesion Universe Mapping & Warp Mechanism Report (Verse v0)

## Objective

Provide a mechanistic, interpretable mapping of MS tissue samples into a lesion-niche multiverse (NAWM/periplaque, rim/edge, core), quantify how cell-type/state orbitals are warped across niches, and deliver actionable cell-program readouts that support MoA and target validation discussions.

## Reference datasets (public)

This pilot is grounded in public snRNA/spatial resources:

- GSE180759 (core/edge/periplaque, progressive MS autopsy)
- GSE301908 (chronic active lesion niche atlas, snRNA + MERFISH paper; snRNA component in GEO)
- Lerma‑Martin 2024 Nat Neurosci (snRNA + spatial niche atlas of subcortical MS lesions)

## Scope (what we will do)

1) Ingest partner tissue dataset (snRNA or bulk-like pseudobulk; optional spatial).
2) Map partner samples into the 3-universe multiverse:
   - U0 NAWM/periplaque
   - U1 rim/edge
   - U2 core
3) Produce "warp mechanism" readouts:
   - Which cell families and orbitals are most shifted between universes
   - Top programs driving shifts (e.g., IFN response, lipid metabolism, scar-associated astrocyte programs)
   - Novelty/OOD calls when sample states do not match reference orbitals (important for rare pathology).
4) Deliver a report and reusable pipeline.

## Out of scope (explicit)

- Clinical prediction of progression.
- Guaranteeing therapeutic efficacy.
- Full longitudinal geodesic modeling (requires longitudinal sampling).

## Deliverables

- PDF report (10–20 slides) summarizing:
  - Universe assignments with uncertainty
  - Orbital shifts (warp vectors) and key programs
  - Comparison vs public references
- CSV outputs:
  - universe scores, orbital membership, warp report
- Reproducible notebook or containerized pipeline.

## Data requirements from partner

- Count matrix and metadata (donor, region, lesion annotation if available).
- If spatial: coordinate + spot/cell metadata.
- Any histology lesion staging labels if present (rim/core/periplaque).

## Timeline

Week 1: data transfer + QC + label harmonization plan.
Week 2–3: mapping + universe scoring + preliminary warp report.
Week 4: final report + handoff of notebooks/pipeline.

## Success criteria

- Stable universe mapping under sensitivity analyses.
- Warp reports reproducible under leave-one-donor/dataset tests (internal validation).
- Programs highlighted in rim vs core shifts are coherent and interpretable in MS lesion biology.
