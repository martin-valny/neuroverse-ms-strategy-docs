# Pilot SOW: MS Lesion Universe Mapping & Warp Mechanism Report (Verse v0)

## Objective

Provide a mechanistic, interpretable mapping of MS tissue samples into a lesion-niche orbital state space, quantify how cell-type/state orbitals are warped across lesion compartments and lesion activity states, and deliver actionable cell-program readouts that support MoA and target validation discussions.

## Model Architecture Note

> The underlying Verse model factors **lesion location** (spatial axis: NAWM / rim / core) and **lesion activity** (activity axis: chronic-active vs inactive) as orthogonal warp axes. The U0/U1/U2 classification is a **v0 reporting abstraction** layered on this richer factorized model. Partner outputs include the simplified classification for operational use, plus full posterior distributions and warp decompositions for scientific interpretation. See `VERSE_ARCHITECTURE.md` for the full technical specification.

## Reference Datasets (Public)

This pilot is grounded in three public snRNA/spatial resources with distinct roles:

| Dataset | Role |
|---------|------|
| GSE180759 (Absinta/Maric et al.) | Primary training cohort; healthy reference orbitals |
| Lerma-Martin 2024 (Nat Neurosci) | Spatial validation anchor; paired snRNA + spatial |
| GSE301908 (Immunity 2025) | Chronic-active rim anchor; external validation |

## Scope (What We Will Do)

1. Ingest partner tissue dataset (snRNA or pseudobulk; optional spatial).
2. Map partner samples into the **lesion orbital state space**, reporting via 3-universe classification:
   - U0: NAWM/periplaque
   - U1: rim/edge (active and inactive)
   - U2: core (active and inactive)
3. Produce factorized **warp mechanism readouts** across all major cell families:
   - Which cell families and orbitals are most shifted between universes
   - Top programs driving shifts per warp axis (spatial and activity separately)
   - Novelty/OOD flags when sample states do not match reference orbitals
4. Deliver a reproducible pipeline and partner-facing report.

## Out of Scope (Explicit)

- Clinical prediction of disease progression.
- Guaranteeing therapeutic efficacy.
- Full longitudinal geodesic modeling (requires longitudinal sampling).
- iPSC microglia scoring (exploratory only, not in v0 quantitative outputs).
- Grey matter lesion analysis.

## Deliverables

### 1. Universe classification output
Per sample/donor CSV with:
- `dominant_universe`: U0 / U1 / U2
- `confidence_tier`: **high** (max posterior probability > 0.7) / **medium** (0.5–0.7) / **low** (< 0.5)
- Full posterior vector: [P(U0), P(U1), P(U2)]
- `flag_novel`: boolean (True if any cell family shows truly-novel states replicated across ≥2 donors)

Ambiguous samples (confidence_tier = low) are flagged — not silently assigned. The deliverable includes a dedicated section explaining why specific samples are ambiguous (incomplete metadata, biologically hybrid lesion stage, or insufficient cell counts).

### 2. Warp mechanism report (CSV + PDF)
- Per cell family × warp axis (spatial, activity): warp magnitude, top-10 FDR-significant programs
- Batch constraint check results (confirms technical effects < 50% of biological effects)
- Evaluation summary: LODO/LOSO metrics, anchoring check, neuron negative control

### 3. PDF report (10–20 slides)
- Universe assignments with uncertainty
- Orbital shift visualizations (warp vectors in program space)
- Key warp-driving programs per cell family
- Spatial orbital mapping (if partner data includes spatial)
- Comparison vs public references (GSE180759, GSE301908, Lerma-Martin)

### 4. Reproducible pipeline
- Containerized (Docker/conda) pipeline runnable via `make demo_ms`
- Documented with provenance for all public reference data
- Partner data stays within partner's compute environment; only trained orbital parameters + report are delivered

## Partner Data Requirements

| Requirement | Minimum | Preferred |
|-------------|---------|-----------|
| Samples | ≥ 6 total | ≥ 10 |
| Per spatial group | ≥ 3 (NAWM, rim, core) | ≥ 5 per group |
| Cells per sample | ≥ 5,000 | ≥ 10,000 |
| Required metadata | donor_id, region/compartment, lesion_type (if available) | + staging (active/inactive), sex, disease duration |
| Optional | Spatial coordinates (spot/cell level) | + histological annotations (iron rim, lipid) |

Samples with missing lesion activity staging are included in the **spatial axis only** (w_spatial estimated; w_activity set to "unknown"). They do not contribute to activity-axis warp fitting but are still classified into U0/U1/U2.

## Timeline

**Week 1: Data QC + Go/No-Go Gate**
- Data transfer and ingestion
- QC report: per-sample cell counts, doublet rates, gene detection rates, metadata completeness
- **Go/No-Go decision**: proceed if ≥ 80% of samples pass QC thresholds. If > 30% of samples fail QC, timeline extends by one additional week (Week 1b) for QC iteration with partner before proceeding.

**Weeks 2–3: Mapping + Warp Analysis**
- Label harmonization and condition schema enforcement
- Universe scoring + orbital membership
- Factorized warp fitting (spatial + activity axes)
- Preliminary warp mechanism report

**Week 4: Final Report + Handoff**
- Final report (PDF + CSVs)
- Reproducible pipeline handoff
- 1-hour walkthrough session with partner team

## Success Criteria

We **target** the following thresholds. If any target is not met, the final report includes a root-cause analysis explaining why (e.g., small sample size, missing activity labels, batch structure) and what would be needed to meet it in a follow-on engagement.

| Criterion | Target |
|-----------|--------|
| Universe assignment agreement on 80% cell subsamples | ≥ 80% |
| Top-10 warp program overlap (Jaccard) vs public reference | ≥ 0.5 |
| Batch warp magnitude / biological warp magnitude | < 50% |
| ≥ 1 literature-supported program per cell family per warp axis | Required |
| Healthy-reference cell anchoring | ≥ 80% inside reference orbitals |

## Intellectual Property & Data Confidentiality

- Partner retains full ownership of all raw and processed data provided.
- Verse retains IP over the generic Verse model architecture, code, and algorithms.
- Trained orbital parameters derived from partner data are not shared, published, or incorporated into public reference atlases without explicit written consent from partner.
- All partner data is processed within a secure environment; no data is uploaded to third-party services.
