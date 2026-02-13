# Verse Architecture: Orbitals, Multiverse Warping, and Geodesics (Condition-Agnostic)

This document summarizes the original ImmuneVerse architecture in a **domain- and condition‑agnostic** way, so it can be applied to immune or CNS (MS/AD) or any other multi-condition single-cell setting. [file:2]

---

## 1) Core thesis

The Verse architecture is built around three ideas: [file:2]

1. There is a **reference latent space** where each cell type/state occupies a probabilistic “orbital” in a pathway/program space.
2. Each **condition** (disease, treatment, timepoint, niche, genotype, etc.) defines its own **warp field** that bends this reference space, shifting/deforming orbitals.
3. **Trajectories** (e.g., progression, treatment) can be represented as paths (geodesics) through these warped spaces, and deviations from geodesics encode resistance or atypical behavior.

The architecture is condition‑agnostic: “condition” is any label that partitions the data and may change cell-state structure.

---

## 2) Layer-by-layer architecture

### 2.1 Inputs

**Input:**

- Single-cell or single-nucleus expression data (optionally multi-omics).
- Per-cell metadata:
  - sample/donor ID,
  - cell type / broad family label,
  - condition label (disease, timepoint, niche, treatment, etc.),
  - technical covariates (batch, protocol, chemistry). [file:2]

---

### 2.2 Layer 1 – Probabilistic Pseudobulk

**Concept:**

Aggregate cells by Sample × CellType (or Sample × CellFamily/State) into **probabilistic pseudobulk**:

- For each group and each gene:
  - mean expression µ,
  - standard deviation σ,
  - number of cells n,
  - confidence ≈ 1 − 1/√n. [file:2]

**Why:**

- Captures biological heterogeneity and sampling uncertainty instead of collapsing to a single point estimate.
- Provides uncertainty estimates that propagate downstream. [file:2]

**Implementation notes:**

- Skip groups below a `min_cells` threshold and handle them with a special “MISSING” representation with low confidence. [file:2]

---

### 2.3 Layer 2 – Probabilistic Pathway / Program Encoder

**Concept:**

Map gene-level pseudobulk distributions into a **pathway/program space**:

- Use a library of pathways/programs (e.g., curated gene sets, regulatory modules).
- Compute pathway scores and corresponding uncertainties from µ/σ.
- Optionally include a **residual encoder** to capture novel programs not covered by known sets. [file:2]

**Minimal v0 implementation note (recommended):**

- Treat the program space as two channels: (1) curated gene-set programs for interpretability, plus (2) a residual/data-driven program channel (e.g., NMF on residualized pseudobulk) to capture biology not covered by curated sets.
- When reporting condition effects, focus on within-cell-type/state shifts (Δprogram vectors and overall warp magnitude) rather than absolute pathway levels, since identity programs are expected to be stable within a cell type.

**Output:**

- For each Sample × CellType group:
  - Program mean vector,
  - Program std vector,
  - Gene-program weights for interpretability (for residuals). [file:2]

---

### 2.4 Layer 3 – Orbital Layer

**Concept:**

Represent each **cell type/state** as a **probabilistic orbital** in program space:

- Each orbital is typically a multivariate Gaussian (or mixture):
  - center (mean program profile),
  - spread (std/covariance),
  - shape (possibly full covariance).
- A sample/program profile has:
  - membership probabilities over orbitals,
  - a **novelty score** indicating how far it lies outside all known orbitals. [file:2]

**Key ideas:**

- Overlap between orbitals encodes transitional states.
- Distance to orbital surface is used for novelty (inside vs outside at a given confidence). [file:2]

**Novelty guardrails (recommended):**

- Make novelty confidence-aware: propagate uncertainty from Layers 1–2 and down-weight low-n / low-confidence groups.
- Only call “novel” states when profiles fall outside all orbitals at a stringent confidence threshold and replicate across donors (and ideally across cohorts).
- Always run negative-control “technical warps” (batch/protocol as pseudo-conditions) to ensure novelty is not driven by technical structure.

**Implementation:**

- Initialize orbitals from reference data (e.g., healthy or baseline conditions).
- Membership uses both sample and orbital uncertainty (combined variance). [file:2]

---

### 2.5 Layer 4 – Multiverse Warping (Condition-Specific Spaces)

**Concept:**

The **Multiverse** is the collection of all condition-specific warped orbital spaces. [file:2]

- **Reference space:** orbital positions in baseline condition (e.g., “healthy” or “baseline”).
- **Warp field for each condition:** learned transformation of orbital means and variances.

Warp types:

- Translation (position shifts),
- Scaling (expansion/contraction),
- Optional rotation (reorienting program axes),
- Optional inter-orbital effects (correlated shifts between orbitals). [file:2]

**Outputs per condition:**

- Warped orbital means and stds,
- Warp magnitude (overall distortion),
- Tools to analyze condition differences (e.g., healthy vs disease warps). [file:2]

---

### 2.6 Layer 5 – Token Construction

**Concept:**

Construct **tokens** from each Sample × CellType group, combining: [file:2]

- Program scores (Layer 2),
- Orbital membership + novelty (Layer 3),
- Warped positions (Layer 4),
- Uncertainty and metadata.

Tokens are **confidence-weighted** so low-confidence groups contribute less to downstream transformers. [file:2]

---

### 2.7 Layer 6 – Cell Type Transformer

**Concept:**

Apply a transformer over the set of tokens for each sample/patient:

- Attention over tokens learns which cell types/states matter for the condition or outcome of interest.
- Outputs:
  - Sample/patient embedding (“imprint”),
  - Attention weights as interpretable cell-type importance. [file:2]

---

### 2.8 Layer 7 – Imprint Space & Multi-Task Heads

**Concept:**

The **imprint space** is a latent representation used for multiple tasks:

- Contrastive objectives (e.g., condition A vs B, responder vs non-responder),
- Supervised heads (e.g., outcome prediction, condition classification). [file:2]

**Tasks:**

- Condition prediction,
- Response prediction,
- Trajectory/paired-condition objectives (pre vs post, timepoint pairs). [file:2]

---

### 2.9 Geodesics & Discovery Modules

**Geodesic Analyzer:**

- Computes expected trajectories in warped space between conditions (e.g., pre → post).
- Measures alignment of actual movement with expected geodesic (response vs resistance). [file:2]

**Warp Field Analyzer:**

- Interprets warp fields as mechanisms:
  - Which orbitals are most warped,
  - Which programs drive shifts,
  - Summary of “condition mechanism” in human-readable form. [file:2]

**Discovery tools:**

- Novel state detection via high novelty scores,
- Biomarker and target discovery from warp-driven program changes and orbital shifts. [file:2]

---

## 3) The “bending” hypothesis (condition-agnostic)

### Core claim

For any set of conditions, the architecture can learn **warp fields** that capture how cell-type/state orbitals are bent in program space between those conditions, and these warps: [file:2]

- are **reproducible** across datasets/cohorts,
- are **distinct** from purely technical effects,
- yield **interpretable** program-level mechanisms.

“Condition” here is generic: disease vs baseline, treatment vs pre-treatment, niche vs other niches, genotype strata, etc.

---

## 4) Minimal evaluation principles (generic)

Regardless of domain (immune, CNS, etc.), to validate the bending idea you need:

1. **Leave-one-cohort/dataset-out warp agreement**
   - Learn orbitals + warps on dataset A,
   - Apply to dataset B,
   - Compare orbital shift vectors and top programs.

2. **Negative-control warps**
   - Treat technical variables (batch/protocol) as pseudo-conditions,
   - Compare their warp patterns to biological conditions.

3. **Orbital anchoring + novelty sanity checks**
   - Check that “reference” cells lie inside orbitals,
   - Condition-specific cells shift in expected directions,
   - New domains (e.g., in vitro models) show novelty when appropriate.

4. **Utility vs unwarped baselines**
   - Show that features derived from warped orb ital spaces improve cross-dataset performance on tasks compared to unwarped program embeddings.

---

## 5) Recommended v0 cuts for a small team

To keep it buildable as an MVP while preserving the unique flavor:

- **Keep:**
  - Program layer,
  - Orbitals + membership + novelty,
  - Per-condition translation + scale warps.

- **Defer:**
  - Full rotation / inter-orbital coupling,
  - Transformer and imprint space (unless needed for a specific downstream task),
  - Geodesics as a primary product (use later for selected settings with enough temporal structure).

This condition-agnostic document should serve as your **canonical architecture reference** for any Verse-based project (MS-first, AD, immune, etc.), with domain specifics living in separate docs.
