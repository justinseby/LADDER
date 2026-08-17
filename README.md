# LADDER

**LADDER** (Literature-Assisted Dual-annotation & Documentation & Evidence-based Reasoning) is a
gene-set annotation and validation pipeline for cancer genomics. Given a gene set, LADDER produces two
independent candidate process annotations (enrichment-driven and direct), each with an LLM-assigned
confidence score, then re-scores the better candidate against literature retrieved specifically for
that gene set, producing a final, literature-grounded annotation and confidence.

This repository contains the full analysis code behind LADDER: the core pipeline, benchmarking against
existing methods, ablations, error analysis, and case studies applying it to real biological questions.

**Manuscript:** in preparation.

## Links

| Resource           | Website / Package                                              | GitHub Repository                                                        |
| ------------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **LADDER**         | [LADDER Website](https://ladder-app.streamlit.app/)            | [Website Repository](https://github.com/justinseby/LADDER-Streamlit-App) |
| **Python package** | [`ladder-gsea` on PyPI](https://pypi.org/project/ladder-gsea/) | [Package Repository](https://github.com/justinseby/ladder-gsea)          |

<table>
  <tr>
    <td align="center" width="50%">
      <strong>LADDER Website</strong><br><br>
      <img src="8_Misc Tests/Figs/Ladder Website Image V1.png" width="100%" alt="LADDER website" />
    </td>
    <td align="center" width="50%">
      <strong>ladder-gsea Python Package</strong><br><br>
      <img src="8_Misc Tests/Figs/image.png" width="100%" alt="ladder-gsea package" />
    </td>
  </tr>
</table>

---

## Repository structure

Each numbered folder is a self-contained analysis task with its own README covering exact input/output
files and how to rerun it. This README is the entry point; see the per-task README for details.

| Task | Folder                               | What it does                                                                                                                                                                                                                                                                                                                                        |
| ---- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | `1_LADDER Annotation and Validation` | The core pipeline: dual annotation (enrichment-driven + direct) followed by literature-grounded validation, run on AML, Breast Cancer, and Lung Cancer gene sets. Every other task builds on this task's output.                                                                                                                                    |
| 2    | `2_LADDER Benchmarking SOTA`         | Benchmarks LADDER's final annotations against two published methods (Hu et al., GeneAgent) using embedding similarity and ROUGE against MSigDB reference descriptions, with paired significance testing.                                                                                                                                            |
| 3    | `3_Confidence Study`                 | Asks whether LADDER's confidence score is meaningful. Uses Gaussian Mixture Modeling on confidence scores across all curated AML/Breast/Lung MSigDB gene sets (n=241, "Original") to fix Low/Medium/High thresholds, then checks those thresholds against Random (n=241, random genes) and 50% Shuffled (n=241, half-randomized) negative controls. |
| 4    | `4_Ablation Analysis`                | Four ablations testing whether literature validation, the scaffolded prompt structure, disease-name leakage in the similarity metric, and run-to-run stability each hold up under scrutiny.                                                                                                                                                         |
| 5    | `5_Error analysis`                   | Checks whether LADDER's confidence scores track how well-studied a gene set's genes actually are, using NCBI GeneRIF counts as a literature-attention proxy.                                                                                                                                                                                        |
| 6    | `6_Case Study`                       | Applies the full pipeline to two targeted biological questions: TP53-mutation status across three cancers, and drug response in ALL (leukemia) cell lines.                                                                                                                                                                                          |
| 7    | `7_LLM Benchmark`                    | Reruns the LADDER pipeline with different LLM backends — GPT-4, Claude Sonnet 4, and DeepSeek-Chat (DeepSeek-V3.2-Exp) — to compare annotation quality across providers.                                                                                                                                                                            |
| 8    | `8_Misc Tests`                       | Small supporting analyses not part of the main paper, e.g. confidence-score visualizations and other exploratory test cases.                                                                                                                                                                                                                        |

```
LADDER/
├── 1_LADDER Annotation and Validation/   Core pipeline (AML, Breast Cancer, Lung Cancer)
├── 2_LADDER Benchmarking SOTA/           vs. Hu et al. and GeneAgent
├── 3_Confidence Study/                   Check how meaninful is the confidence score and define threshold for the confidnece bins
├── 4_Ablation Analysis/                  Validation / Scaffold / Masking / Stability
├── 5_Error analysis/                     GeneRIF well-studied-score correlation
├── 6_Case Study/                         TP53 case study + ALL cell-line case study
├── 7_LLM Benchmark/                      GPT / Claude / DeepSeek
└── 8_Misc Tests/
```

---

## How to reproduce

Every task is a set of Jupyter notebooks plus, for the figures, R scripts. The general setup is the
same across tasks; task-specific file names, extra dependencies, and exact run order are documented in
each task's own README.

### 1. Clone and set up Python

```bash
git clone https://github.com/justinseby/LADDER.git
cd ladder
python3 -m venv venv
source venv/bin/activate
pip install pandas numpy scipy scikit-learn statsmodels torch transformers \
            gseapy rouge-score requests tqdm rapidfuzz openpyxl jupyter
```

Individual tasks may need one or two extra packages beyond this common set — see that task's README.

### 2. Set up R (for figures)

```r
install.packages(c("ggplot2", "dplyr", "tidyr", "patchwork", "ggdist", "scales"))
```

### 3. API keys

```bash
export DEEPSEEK_API_KEY="your_deepseek_key"      # LLM annotation + validation calls
export NCBI_EMAIL="your_email@example.com"       # PubMed / PMC literature retrieval
export NCBI_API_KEY="your_ncbi_key"              # optional, raises NCBI rate limits
```

Task 7 (LLM Benchmark) additionally needs API keys for the other providers it compares (OpenAI/GPT,
Anthropic/Claude) — see that task's README notebooks.

---

## Citing this work

A manuscript describing LADDER is in preparation. Citation details will be added here once available.
