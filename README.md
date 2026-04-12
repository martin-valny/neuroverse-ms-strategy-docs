# neuroverse-ms-strategy-docs

MS-focused CNS strategy documentation for the **Verse v0 platform** — a probabilistic single-cell architecture that models cell-type/state orbitals and condition-specific warp fields across CNS lesion niches.

---

## Document Index

### Architecture

| Document | Description |
|----------|-------------|
| [`docs/VERSE_ARCHITECTURE.md`](docs/VERSE_ARCHITECTURE.md) | **Canonical architecture spec** — probabilistic pseudobulk, two-channel program encoder, L1/L2 orbital hierarchy, factorized warp fields, evaluation criteria |
| [`docs/VERSE_ARCHITECTURE_DIAGRAM.md`](docs/VERSE_ARCHITECTURE_DIAGRAM.md) | Visual pipeline diagram — ASCII flowcharts for presentations and onboarding |
| [`docs/ARCHITECTURE_IMPLEMENTATION_MAP.md`](docs/ARCHITECTURE_IMPLEMENTATION_MAP.md) | Architecture → checklist cross-reference — maps every spec section to its implementation step; concept orphan check |

### MS-First Strategy

| Document | Description |
|----------|-------------|
| [`docs/STRATEGY_MEMO_MS_FIRST.md`](docs/STRATEGY_MEMO_MS_FIRST.md) | Scientific strategy — datasets, hypotheses, cell-type scope, warp axes, evaluation suite |
| [`docs/ENGINEERING_CHECKLIST_MS_FIRST.md`](docs/ENGINEERING_CHECKLIST_MS_FIRST.md) | Operational implementation checklist — condition schema, data ingestion, program layer, orbital fitting, warp fitting, spatial validation, evaluation |
| [`docs/PILOT_SOW_MS_FIRST.md`](docs/PILOT_SOW_MS_FIRST.md) | Partner-facing Statement of Work — scope, deliverables, data requirements, timeline, success criteria, IP terms |
| [`docs/VERSE_FOR_MS_ONE_PAGE.md`](docs/VERSE_FOR_MS_ONE_PAGE.md) | One-page partner/investor summary — for slide decks and BD conversations |

---

## Key Design Decisions

- **Reference**: Healthy tissue (not NAWM) is the fixed orbital anchor. NAWM is modeled as a mild warp.
- **Factorized warps**: Spatial (NAWM/rim/core) and lesion activity (chronic-active/inactive) are orthogonal warp axes, separated in the design matrix.
- **All cell types**: L1/L2 orbital hierarchy covers microglia, astrocytes, OL-lineage, OPC, T cells, neurons, endothelial. Microglia is the primary readout story, not the sole scope.
- **Uncertainty-aware**: NB variance + DESeq2 shrinkage at pseudobulk; confidence weighting propagated through the pipeline.
- **Batch**: Harmony at cell level; batch as fixed-effect covariate at pseudobulk and warp stages.
- **Datasets**: GSE180759 (primary + healthy controls), Lerma-Martin 2024 (spatial anchor), GSE301908 (chronic-active rim, external validation).

---

## Reading Order

1. Start with `VERSE_ARCHITECTURE.md` — the canonical spec
2. Use `VERSE_ARCHITECTURE_DIAGRAM.md` for visual orientation
3. Read `STRATEGY_MEMO_MS_FIRST.md` for the scientific narrative
4. Follow `ENGINEERING_CHECKLIST_MS_FIRST.md` to implement
5. Use `ARCHITECTURE_IMPLEMENTATION_MAP.md` to verify nothing is missed
6. Reference `PILOT_SOW_MS_FIRST.md` for partner-facing outputs
7. Use `VERSE_FOR_MS_ONE_PAGE.md` for decks and BD conversations
