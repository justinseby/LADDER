# LADDER Ablation Analysis

Three independent ablations, each isolating one design choice in the LADDER pipeline to check whether
it is actually earning its keep, rather than benchmarking LADDER against outside methods.

> **Scope:** This README covers the ablation task as a whole. See the per-folder READMEs for the exact
> files and columns used in each ablation.

```
                     LADDER pipeline (Task 1)
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ Before/After   │   │ Freeform vs      │   │ Masked Analysis  │
│ Validation     │   │ Scaffolded       │   │                  │
│ Ablation       │   │                  │   │                  │
│                │   │                  │   │                  │
│ Does Stage-3   │   │ Does the         │   │ Is LADDER's edge │
│ literature     │   │ scaffolded,      │   │ over baselines a │
│ validation     │   │ multi-step       │   │ real semantic    │
│ improve on the │   │ prompt beat a    │   │ match, or does   │
│ pre-validation │   │ single freeform  │   │ it just win by   │
│ pick?          │   │ prose prompt?    │   │ naming the       │
│                │   │                  │   │ disease?         │
└───────────────┘   └──────────────────┘   └──────────────────┘
```

---

## 1. What each ablation tests

### Before and After Validation Ablation

Tests whether Stage 3 (literature-grounded re-scoring, Task 1) improves annotation quality over just
taking the better of Stage 1's two candidate annotations.

1. Pools all three diseases' `Validation V1` tables and MSigDB descriptions into single combined files.
2. For each gene set, builds four candidate annotations to compare:
   - `Enrichment_driven_LLM_without_validation` — Stage 1's enrichment-driven candidate, no literature
     grounding
   - `Direct_LLM_without_validation` — Stage 1's direct candidate, no literature grounding
   - `Final_without_validation` — whichever of the two candidates above had the higher _pre-validation_
     confidence (the best answer achievable without literature)
   - `Final_with_validation` — Stage 3's actual literature-validated final process
3. Embeds all four candidates and the MSigDB description with each embedding model (BioLORD-2023,
   MedCPT), computes cosine similarity of each candidate to the description, and records the winner
   (or tied winners) per gene set per model.
4. Plots win counts for all four candidates side by side, grouped by embedding model, so the
   literature-validated answer can be checked against not just the un-validated "best pick" but the two
   raw Stage-1 candidates individually

### Freeform vs Scaffolded

Tests whether LADDER's structured, multi-step scaffold (name-and-score twice, then reconcile) is doing
better than just asking the LLM to reason freely in prose.

1. `Freeform Prompt Annotation.ipynb` re-runs annotation on the same AML gene-set communities with a
   single freeform prompt: the model writes continuous prose reasoning (still given the same enrichment
   context) and ends with a fixed `FINAL_ANNOTATION` / `FINAL_CONFIDENCE` / `FINAL_CONTRIBUTING_GENES`
   block that gets parsed out.
2. `Comparison.ipynb` merges the freeform output with the original scaffolded Stage-1 result for the
   same gene sets, merges with MSigDB descriptions, and reruns the embedding-similarity comparison
   (Scaffold vs Paragraph) against the reference.
3. `Comparison R visulization.ipynb` renders the resulting win-count/similarity figures.

### Masked Analysis

Tests whether LADDER's benchmarking wins (Task 2) hold up once disease-name mentions are removed from
its annotation text, since MSigDB descriptions often contain the disease name and a method could win
semantic similarity trivially just by repeating it.

1. Takes LADDER's own annotation column, strips disease-name and abbreviation mentions
   (e.g. "acute myeloid leukemia", "AML") via regex.
2. Merges with MSigDB descriptions and reruns the same embedding-similarity comparison against Hu and
   GeneAgent used in Task 2, to see whether LADDER still wins with the disease name masked out.
3. R visualization renders the masked-vs-baseline win counts.
