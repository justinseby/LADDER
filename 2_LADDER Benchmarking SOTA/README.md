# LADDER Benchmarking Against SOTA Methods

This describes how LADDER's final, literature-validated annotations (Task 1 output) are benchmarked
against two published gene-set annotation methods, Hu et al. and GeneAgent, across three cancer types.

> **Scope:** This README covers the benchmarking task as a whole. Each disease folder (`AML/`,
> `Breast Cancer/`, `Lung Cancer/`) runs the identical procedure on its own annotation files; see the
> per-folder READMEs for disease-specific file names. `Statistical Testing/` aggregates the outputs of
> all three.

```
LADDER final annotations (Task 1)  +  Hu et al. / GeneAgent annotations
        │
        ▼
┌────────────────────────────┐
│  Per-disease benchmarking   │  - align gene sets across methods (With/Without context)
│  (AML/, Breast Cancer/,     │  - merge with MSigDB reference descriptions
│  Lung Cancer/)              │  - embedding cosine similarity (BioLORD-2023, MedCPT) + ROUGE-1/2/L
└────────────────────────────┘  - attach LADDER's Final_Confidence, bin into Low/Med/High
        │  per-disease semantic + ROUGE result tables
        ▼
┌────────────────────────────┐
│  R Visualizations           │  - win-count bar panels + raincloud plots per disease
│  (per disease folder)       │
└────────────────────────────┘
        │
        ▼
┌────────────────────────────┐
│  Statistical Testing        │  - pools all three diseases, both contexts
│                              │  - paired t-tests, LADDER vs Hu / LADDER vs GeneAgent
└────────────────────────────┘  - per-disease and pooled significance tables
```

---
