# LADDER Case Studies

Two applications of the full LADDER pipeline (annotation + literature validation, as in
Task 1) to targeted biological questions outside the three main benchmark gene sets, showing the
pipeline generalizes beyond its original AML/Breast Cancer/Lung Cancer inputs.

> **Scope:** This README covers the case-study task as a whole. See the per-folder READMEs for the
> exact input files and columns in each disease/cell-line subfolder.

```
6_Case Study
│
├── TP53 Case Study (AML TP53/, BC TP53/, LC TP53/)
│     Proteomics + TP53-mutation metadata (per disease)
│             │
│             ▼
│     Differential expression: TP53-mutant vs wild-type samples
│     (T-test, log2FC)  →  Up-regulated protein sets
│             │
│             ▼
│     Knowledgebase creation: per-gene PubMed/PMC fetch, query
│     restricted to (disease MeSH terms) AND (TP53/p53 terms)
│             │
│             ▼
│     LADDER annotation (Stage 1: enrichment-driven + direct)
│     → clean process names (strip embedded confidence suffixes)
│             │
│             ▼
│     LADDER validation (Stage 3: JIF-filtered literature
│     re-scoring)  →  Final Up/Down validated process tables
│
└── Cell line Case Study (ALL, acute lymphoblastic leukemia)
      Per-drug differential proteomics (precomputed upstream)
             │
             ▼
      Build per-drug upregulated gene sets (>=10 genes/drug,
      p<0.01, FDR fallback if a set is too large)
             │
             ▼
      Knowledgebase creation: per-gene PubMed/PMC fetch, query
      restricted to (ALL leukemia MeSH terms) AND (drug name),
      one paper CSV per drug-direction, resumable
             │
             ▼
      LADDER annotation (Stage 1) → clean process names
             │
             ▼
      LADDER validation (Stage 3)  →  DrugValidation.csv
```

---

## 1. What each case study does

### TP53 Case Study (`AML TP53/`, `BC TP53/`, `LC TP53/`)

Asks: what biological processes does LADDER identify as altered when TP53 is mutated, in each of the
three cancer types studied elsewhere in this project?

1. **Differential expression.** Loads each disease's proteomics matrix and sample metadata, splits
   samples into TP53 wild-type vs mutant by the metadata's `TP53` (or `TP53_mut`) column, and runs a
   Mann-Whitney U test per protein between the two groups, computing log2 fold-change. Filters to
   significant proteins (disease-specific p-value/FDR threshold) and splits into up- and
   down-regulated sets, saved as `..._Annotation_Up.csv` / `..._Annotation_Down.csv` gene lists.
   (Each disease's raw proteomics/metadata format differs — text matrix + `.txt` metadata for AML,
   multi-sheet Excel for Breast Cancer and Lung Cancer — but the statistical step is identical.)
2. **Knowledgebase creation.** Merges the up and down gene sets, then fetches PubMed (and PMC full
   text where available) per unique gene, with the search query restricted to
   `(disease MeSH/keyword terms) AND (TP53/p53 terms)`, so only TP53-relevant literature for that
   disease is retrieved. Saves a combined `..._Paper_DB.csv`.
3. **LADDER annotation (Stage 1).** Runs the same enrichment-driven + direct dual-prompt annotation
   used in Task 1 on the up and down gene sets separately
4. **LADDER validation (Stage 3).** Runs the same JIF-filtered, literature-grounded validator used in
   Task 1 against the disease+TP53 paper database, producing `..._Validation_Up.csv` /
   `..._Validation_Down.csv`.

### Cell line Case Study (ALL)

Asks: applied to a completely different disease (acute lymphoblastic leukemia) and a completely
different experimental design (per-drug differential proteomics in cell lines, not tumor vs normal),
does LADDER still produce sensible, literature-grounded process annotations?

1. **Gene-set construction.** Starts from `ALL_cell_lines_diff_proteins_upregulated.txt`, a precomputed
   per-drug differential proteomics table (`drug`, `protein`, `pval`, `adjpval`, `m_diff`). For each
   drug, filters to `pval < 0.01` and upregulated proteins (`m_diff > 0`); if that set exceeds 200
   genes, refilters using `adjpval < 0.01` instead. Keeps only drug sets with >=10 genes, saved as
   `drug_gene_sets.csv`.
2. **Knowledgebase creation.** Fetches PubMed/PMC papers per gene, per drug-direction community, with
   the query restricted to `(ALL leukemia MeSH/keyword terms) AND (that specific drug name)`, writing
   one `{drug}_{direction}_Papers.csv` per community under `papers_per_drug/`.
3. **LADDER annotation + validation.** Same (enrichment + direct driven llm) annotation and
   literature validation as the TP53 case studies, producing `DrugValidation.csv`.

---

## 2. Requirements

- Python 3.9+ and Jupyter
- `pandas`, `numpy`, `scipy`, `statsmodels`, `gseapy`, `requests`, `tqdm`, `rapidfuzz`, `openpyxl`
  (for the Excel proteomics files in BC/LC)
- A DeepSeek API key (`DEEPSEEK_API_KEY`) for annotation and validation
- An NCBI email and, optionally, API key for PubMed/PMC fetching
- The Clarivate JIF journal list (`journals_filtered_JIF_ge_4.csv`) for the validator's quality filter

```bash
pip install pandas numpy scipy statsmodels gseapy requests tqdm rapidfuzz openpyxl jupyter
export DEEPSEEK_API_KEY="your_key_here"
export NCBI_EMAIL="your_email@example.com"
export NCBI_API_KEY="your_ncbi_key_here"   # optional, raises rate limits
```

---

## 3. Output columns (validation tables, both case studies)

| Column                                                              | Meaning                                                                            |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `Set_ID` / `Community`                                              | Gene-set identifier (e.g. `AML_TP53_UP`, `Vincristine_UP`)                         |
| `Genes`                                                             | Genes in the set                                                                   |
| `Process_With_Enrichment_Original` / `_Without_Enrichment_Original` | two candidate process names, pre-validation                                        |
| `Confidence_With_Enrichment_Before` / `_After`                      | annotation vs validation confidence for the enrichment-driven candidate            |
| `Confidence_Without_Enrichment_Before` / `_After`                   | annotation vs validation confidence for the direct candidate                       |
| `Final_Process`                                                     | validation stage's selected process (or `"Neither process"` if confidence <= 0.05) |
| `Final_Confidence`                                                  | Confidence of the selected process                                                 |
| `Validation_Analysis_Text`                                          | LLM's literature-grounded reasoning for the final call                             |
| `Conflicting_Evidence_Found` / `Conflict_Description`               | Whether the retrieved papers disagreed on gene function/pathway                    |
| `Total_Papers_Found`                                                | Number of high-quality papers used for that gene set's validation                  |
