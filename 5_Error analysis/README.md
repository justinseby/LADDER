# GeneRIF Well-Studied Score vs. Final Confidence

This checks whether LADDER's `Final_Confidence` tracks how well-studied the genes in a gene set
actually are, using NCBI GeneRIF counts as a literature-attention proxy, and correlates the two.

```
AML / Breast Cancer / Lung Cancer            NCBI GeneRIF + Gene Info
Validation V1.csv (Task 1 output)            (generifs_basic.gz, gene_info.gz)
        │                                              │
        ▼                                              ▼
┌────────────────────┐                    ┌──────────────────────────┐
│ Combine diseases    │                    │ Build symbol → GeneID     │
│                     │                    │ map (incl. synonyms) and  │
│                     │                    │ per-gene GeneRIF/PubMed   │
│                     │                    │ counts                    │
└────────────────────┘                    └──────────────────────────┘
        │                                              │
        └───────────────────┬──────────────────────────┘
                             ▼
                ┌────────────────────────────┐
                │ Score each gene set          │  - map every gene in the set to its
                │                               │    GeneRIF count
                │                               │  - aggregate to total/mean/median
                └────────────────────────────┘
                             │
                             ▼
                ┌────────────────────────────┐
                │ Spearman correlation         │  - GeneRIF metrics vs Final_Confidence
                │                               │  - overall, and per cancer type
                └────────────────────────────┘
```

---

## 1. What it does

1. **Combine.** Loads the three per-disease `Validation V1.csv` files (AML, Breast Cancer, Lung
   Cancer — Task 1's Stage-3 output), tags each row with its `Cancer_Type`, and concatenates them into
   `Combined_Validation_V1.csv`.
2. **Download reference data.** Pulls two NCBI FTP resources if not already cached locally:
   - `generifs_basic.gz` — every GeneRIF (short, PubMed-linked, human-curated statement of gene
     function) for all organisms.
   - `Homo_sapiens.gene_info.gz` — the human gene reference table (symbols, synonyms, GeneIDs).
     Both are filtered down to human (`tax_id == "9606"`).
3. **Build lookup tables.** Constructs a gene-symbol → GeneID map (falling back to known synonyms for
   symbols that don't match directly), and counts, per GeneID: total GeneRIF entries and number of
   unique PubMed IDs citing that gene.
4. **Score each gene set.** Parses the `Genes` column of the combined table back into a list, looks up
   the GeneRIF count for every gene in the set, and aggregates into `GeneRIF_total`, `GeneRIF_mean`,
   `GeneRIF_median`, plus how many genes were/weren't found in the NCBI reference
   (`Genes_found` / `Genes_not_found`). Saves the result as
   `Combined_Validation_V1_with_GeneRIF.csv`.
5. **Correlate.** Runs Spearman rank correlation between each GeneRIF metric
   (`GeneRIF_total`/`_mean`/`_median`) and `Final_Confidence`, both across all diseases combined and
   separately per cancer type (using `GeneRIF_mean`).

---

## 2. Output columns (`Combined_Validation_V1_with_GeneRIF.csv`)

| Column                            | Meaning                                                                    |
| --------------------------------- | -------------------------------------------------------------------------- |
| `Cancer_Type`                     | Which disease's Task 1 output this row came from                           |
| `Gene_List`                       | Parsed Python list version of the `Genes` column                           |
| `GeneRIF_total`                   | Sum of GeneRIF entries across all genes in the set that were found         |
| `GeneRIF_mean` / `GeneRIF_median` | Mean / median GeneRIF count per gene in the set                            |
| `Genes_found` / `Genes_not_found` | How many of the set's genes did / didn't resolve to an NCBI GeneID         |
| `Final_Confidence`                | LADDER's Stage-3 validated confidence (from Task 1), used as the correlate |
