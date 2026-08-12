# AML Gene Set Pathway Annotation & Literature Validation Pipeline

This describes the three-stage pipeline used to annotate AML (Acute Myeloid Leukemia) gene sets with biological process labels and then validate those annotations against high-quality
published literature.

> **Scope:** This README covers the **AML** pipeline only. The lung cancer and breast cancer pipelines
> follow the same overall design (annotation → knowledgebase → literature validation)

```
AML gene communities (input)
        │
        ▼
┌───────────────────────────┐
│  Stage 1: Annotation      │
│                            │   - enrichment driven and direct LLM annotation
└───────────────────────────┘
        │  gene-set annotation table
        ▼
┌───────────────────────────┐
│  Stage 2: Knowledgebase    │
│  Creation                  │  - PubMed/PMC retrieval for every gene in the
│                             │    annotated gene sets, consolidated into a
│                             │    single AML literature database
└───────────────────────────┘
        │  AML literature database
        ▼
┌───────────────────────────┐
│  Stage 3: Validation       │
│                            │  - literature-grounded validation and re-scoring
└───────────────────────────┘
        │
        ▼
  Final validated annotations
```

---

## 1. What each stage does

### Stage 1 — Annotation

For every AML gene community, the notebook:

1. Runs enrichment analysis against `GO_Biological_Process_2021`, `Reactome_2022`, and
   `KEGG_2021_Human`, keeping the top 5 significant terms (by adjusted p-value) per database.
2. Sends a single combined prompt to an LLM asking it to:
   - Name and score (0.00–1.00) a biological process **using** the enrichment terms as context
     (Enrichment Driven LLM Annotation).
   - Independently name and score a process **from prior knowledge only**, ignoring enrichment
     (Direct LLM Annotation).
   - Compare the two and select a **Final Process** (the higher-confidence one).
3. Parses the model's structured response into columns (process names, confidence scores, reasoning
   text, contributing genes for each condition, plus the final selection).
4. Writes one row per gene set to a results table.

This stage produces the **pre-validation annotations** — LLM enrichment-driven/direct-LLM-based labels
with no literature grounding yet.

### Stage 2 — Knowledgebase Creation

Before validation can happen, every gene referenced anywhere in Stage 1's annotated gene sets needs
supporting literature. This stage builds that literature database:

1. Reads Stage 1's annotation table and collects the set of **unique genes** across all gene-set
   communities, tracking which geneset each gene belongs to.
2. For each unique gene, queries PubMed (via NCBI E-utilities) restricted to AML-specific MeSH terms
   and free-text disease terms (`"Leukemia, Myeloid, Acute"[MeSH Terms]`, `"acute myeloid leukemia"`,
   `"AML"` + `"leukemia"/"myeloid"`, etc.), within a configurable publication-year window, retrieving
   up to a fixed number of top (relevance-ranked) results per gene.
3. Parses each returned PubMed record (title, abstract, journal, ISSN/eISSN, year, volume, pages,
   authors, DOI, PMCID).
4. Where a PMC ID is available, fetches the **full text** of the article via PMC E-utilities in
   addition to the abstract; falls back to "Full text not available via PMC" otherwise.
5. Consolidates every gene's papers into a **single unified AML literature database** , tagging each paper row with the gene(s)/communities it was retrieved for, and
   de-duplicates on (PMID, gene).
6. Supports resuming an interrupted run: if an output database already exists, genes already present
   are skipped and only remaining genes are fetched, then appended.

This stage produces the **AML literature database** consumed by Stage 3

### Stage 3 — Literature Validation

Takes Stage 1's annotation output and Stage 2's literature database and re-scores each gene set against
real papers:

1. Loads the AML literature database (Stage 2's output) and a journal-quality reference list, and tags
   each paper as high-quality (HQ) via ISSN → eISSN → normalized journal name matching.
2. For each gene set, filters papers to those mentioning the query genes, keeps **only HQ papers**, and
   ranks them by how many query genes they mention (using the combined abstract + full-text corpus).
3. Builds a validation prompt containing the original Stage 1 results plus the top
   matching HQ papers, and asks the LLM to:
   - Independently assess evidence for the "Enrichment Driven" and "Direct LLM" processes.
   - Produce updated confidence scores strictly from the provided literature.
   - Select a final process (or "Neither process" if both updated confidences are ≤ 0.05).
   - Flag any conflicting evidence across papers.
4. Saves before/after confidence scores, the final validated process, supporting citations, and
   conflict flags, checkpointing periodically during the run.

This stage produces the **final, literature-grounded confidence scores** used downstream.

---

## 2. Requirements

- Python 3.9+ and Jupyter
- `pandas`, `requests`, `tqdm`, `gseapy` (Stage 1 only)
- An LLM API key with access to the model used in the notebooks (Stages 1 and 3)
- An NCBI account email, and optionally an NCBI API key, for PubMed/PMC E-utilities access (Stage 2)

```bash
pip install pandas requests tqdm gseapy jupyter
export LLM_API_KEY="your_key_here"
export NCBI_EMAIL="your_email@example.com"
export NCBI_API_KEY="your_ncbi_api_key_here"   # optional, raises E-utilities rate limits
```

---

## 3. Output columns (Stage 3 final table)

| Column                                                                       | Meaning                                                                        |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `Set_ID`                                                                     | Community/gene-set identifier                                                  |
| `Genes`                                                                      | List of genes in the set                                                       |
| `Process_With_Enrichment_Original` / `Process_Without_Enrichment_Original`   | Stage 1 process names (Enrichment driven annotation and Direct LLM annotation) |
| `Confidence_With_Enrichment_Before` / `Confidence_Without_Enrichment_Before` | Stage 1 confidence scores                                                      |
| `Confidence_With_Enrichment_After` / `Confidence_Without_Enrichment_After`   | Literature-updated confidence scores                                           |
| `Final_Process` / `Final_Confidence`                                         | The process and score selected after validation                                |
| `Validation_Analysis_Text`                                                   | Full reasoning text from the validation step                                   |
| `Supporting_Citations`                                                       | Citations backing the final process                                            |
| `Conflicting_Evidence_Found` / `Conflict_Description`                        | Whether papers disagreed, and how                                              |
| `Total_Papers_Found`                                                         | Number of high-quality, gene-matching papers used for validation               |

---
