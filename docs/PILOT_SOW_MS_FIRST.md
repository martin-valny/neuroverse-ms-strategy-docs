# Pilot SOW: MS Lesion Universe Mapping & Warp Mechanism Report (Verse v0)

## Objective

Provide a mechanistic, interpretable mapping of MS tissue samples into a lesion-niche orbital state space, quantify how cell-type/state orbitals are warped across lesion compartments and lesion activity states, and deliver actionable cell-program readouts that support MoA and target validation discussions.

## Model Architecture Note

> The underlying Verse model factors **lesion location** (spatial axis: NAWM / rim / core) and **lesion activity** (activity axis: chronic-active vs inactive) as separable warp axes. The U0/U1/U2 classification is a **v0 reporting abstraction** layered on this richer factorized model. Partner outputs include the simplified classification for operational use, plus full posterior distributions and warp decompositions for scientific interpretation. See `VERSE_ARCHITECTURE.md` for the full technical specification.

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
- Per cell family × warp axis (spatial, activity): warp magnitude, top-10 programs by effect size (|Δ_program|)
- Batch constraint check results (confirms technical effects < 50% of biological effects)
- Evaluation summary: LODO/LOSO metrics, anchoring check, neuron negative control
- **Sparse family note**: T cell and endothelial warp analysis is best-effort; reported only if ≥ 5 donors meet the minimum cell count threshold per compartment

### 3. PDF report (10–20 slides)
- Universe assignments with uncertainty
- Orbital shift visualizations (warp vectors in program space)
- Key warp-driving programs per cell family
- Spatial orbital mapping (if partner data includes spatial)
- Comparison vs public references (GSE180759, GSE301908, Lerma-Martin)

### 4. Reproducible pipeline
- Containerized (Docker/conda) pipeline. Two targets:
  - `make demo_ms` — downsampled fast run (~30 min): 1,000 cells/donor, microglia only, no LOSO/spatial validation. Smoke test only.
  - `make run_ms` — full production run (~12–48h): all families, all donors, full evaluation.
- Documented with provenance for all public reference data
- **Partner data deployment model**: partner data stays within the partner's compute environment. Verse delivers: (a) the pre-trained reference orbital model (JSON + npz format), (b) the inference pipeline (Docker image), and (c) the final report. Partner runs inference locally on their data; only results (orbital membership, warp report, PDF) leave their environment. No raw or processed partner data is transferred to Verse systems.
- **Partner validation protocol**: before production inference, partner must (1) execute `make demo_ms` on the delivered Docker image and share the QC output with Verse, (2) confirm that demo metrics (anchoring check, program library size) match the expected reference values provided by Verse, (3) share the `condition_schema_check.json` output from their data to verify metadata conformity. Only after these checkpoints pass does Verse issue the final report.

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

> **Scope assumption**: the 4-week timeline below assumes partner data arrives with pre-existing cell-type annotations and complete metadata (donor_id, spatial_label, activity_label). If partner data requires de novo cell-type annotation, Harmony integration with reference datasets, or extensive metadata curation, add 2–3 weeks to Weeks 1–2. Realistic total for unvetted partner data: **6–8 weeks**.

**Week 1: Data QC + Go/No-Go Gate**
- Data transfer and ingestion
- QC report: per-sample cell counts, doublet rates, gene detection rates, metadata completeness
- Harmony integration (if partner data requires batch correction against reference datasets)
- Cell-type annotation harmonization to canonical `cell_family` vocabulary
- **Go/No-Go decision**: proceed if ≥ 80% of samples pass QC thresholds. If > 30% of samples fail QC, timeline extends by one additional week (Week 1b) for QC iteration with partner before proceeding.

**Weeks 2–3: Mapping + Warp Analysis**
- Pseudobulk construction (Sample × CellFamily × SpatialLabel) + program scoring
- Per-family PCA + reference orbital fitting from healthy tissue
- Factorized warp fitting (spatial + activity axes) with nested CV for λ selection
- Post-fitting disease state discovery (clustering + annotation)
- Preliminary warp mechanism report
- **Compute note**: LODO evaluation alone requires re-fitting N≈20 times per cell family (~40+ compute hours). Full evaluation suite (LODO + LOSO + batch checks + spatial validation) requires dedicated compute resources for 2–4 days.

**Week 4: Final Report + Handoff**
- Final report (PDF + CSVs)
- Reproducible pipeline handoff
- 1-hour walkthrough session with partner team

## Success Criteria

We **target** the following thresholds. If any target is not met, the final report includes a root-cause analysis explaining why (e.g., small sample size, missing activity labels, batch structure) and what would be needed to meet it in a follow-on engagement.

| Criterion | Target | Notes |
|-----------|--------|-------|
| Universe assignment agreement on 80% cell subsamples | ≥ 80% | Stability check |
| LODO warp direction (Spearman ρ, primary families) | ≥ 0.65 aspirational | Realistic with N≈20; report ρ distribution across folds |
| Top-10 warp program overlap (Jaccard) vs public reference | ≥ 0.5 in ≥ 2 of 3 LOSO folds | Qualitative sanity check only — 3 folds cannot support generalization claims |
| Batch warp magnitude / biological warp magnitude | < 50% | Hard constraint |
| ≥ 1 of the top-3 warp-driving programs per cell family per warp axis is explicitly annotated in one of the three anchor papers or in the Expected Disease State Annotation Guide | Required | Documents that discovered biology is interpretable. Mapping is done **after** ranking programs by effect size. Programs with only passing mentions in unrelated literature do not satisfy this criterion. |
| Healthy-reference cell anchoring | ≥ 80% inside reference orbitals | Per family |

## Intellectual Property & Data Confidentiality

- Partner retains full ownership of all raw and processed data provided.
- Verse retains IP over the generic Verse model architecture, code, and algorithms.
- **Reference orbitals** fitted from the three public datasets (GSE180759, Lerma-Martin 2024, GSE301908) are Verse IP and may be published. Orbital parameters fitted from partner data are partner-owned.
- Trained orbital parameters derived from partner data are not shared, published, or incorporated into public reference atlases without explicit written consent from partner.
- All partner data remains within the partner's compute environment (see Deliverable 4 above for partner data deployment model).
- **Novel discoveries**: if partner data reveals novel cell states (flag_novel = TRUE, replicated ≥2 donors), ownership and publication rights for such discoveries should be covered in a **separate detailed IP agreement** executed before partner engagement begins. This SOW establishes the operational framework; a dedicated IP agreement is required to address: novel discovery ownership, publication priority, embargo periods, and dispute resolution.

## Known v0 Limitations

The following limitations are acknowledged and documented in the final report:
- **Donor non-independence**: pseudobulk groups from the same donor across spatial compartments are treated as independent in v0. Mixed-model correction (v1) will address within-donor correlation.
- **σ_program not propagated**: novelty scoring uses orbital uncertainty only; gene-level uncertainty is not propagated through program scoring in v0.
- **T cell / sparse family analysis**: best-effort only; most donors will not have sufficient T cells per compartment for robust warp analysis.
- **LOSO reproducibility**: 3-dataset LOSO is a sanity check, not a statistical test of generalization.

---

*Version: v0.6 — last updated 2026-04-15. Key changes from v0.5: timeline scope note added (6-8 weeks for unvetted data); compute note for LODO added; literature criterion tightened (top-3 programs, anchor papers only); IP section clarified (reference orbital ownership, novel discovery IP note, separate agreement required); partner validation protocol added; "orthogonal" → "separable."*
