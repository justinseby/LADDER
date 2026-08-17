# Confidence Study

Asks whether LADDER's confidence score is meaningful, independently of the benchmarking datasets used
elsewhere in this project. Establishes fixed Low/Medium/High confidence thresholds via Gaussian Mixture
Modeling on real gene sets, then checks whether those thresholds behave sensibly on two negative
controls of increasing difficulty.

> **Scope:** This README covers all of Task 3. The AML/Breast Cancer/Lung Cancer pipelines follow an
> identical procedure per disease; file names differ only by disease name.

```
MSigDB gene sets (AML, Breast Cancer, Lung Cancer)
        │
        ├──────────────────────────────┬──────────────────────────────┐
        ▼                              ▼                              ▼
┌────────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐
│ Original           │      │ Random                 │      │ 50% Shuffled           │
│ (n=241 total)      │      │ (n=241 total)          │      │ (n=241 total)          │
│                    │      │                        │      │                        │
│ The real, curated  │      │ Same-size gene sets,   │      │ Each original set with │
│ gene sets, run     │      │ genes drawn at random  │      │ 50% of genes kept and  │
│ through the normal │      │ from the full cancer   │      │ 50% replaced at random │
│ LADDER pipeline    │      │ gene pool              │      │                        │
└────────────────────┘      └────────────────────────┘      └────────────────────────┘
        │                              │                              │
        └──────────────────────────────┴──────────────────────────────┘
                                        ▼
                        ┌────────────────────────────┐
                        │ GMM.ipynb                  │  -  fit 3 GMM on
                        │                            │    Final_Confidence.
                        │                            │  - fix Low/Medium/High thresholds at the
                        │                            │    midpoints between component
                        │                            │
                        │                            │
                        └────────────────────────────┘
                                        │
                                        ▼
                        ┌────────────────────────────┐
                        │ R Visualization.ipynb      │
                        │                            │
                        └────────────────────────────┘
```

---

## 1. What each stage does

### Negative-control gene-set construction

For each disease (AML, Breast Cancer, Lung Cancer), starting from that disease's real MSigDB gene-set
communities:

- **Random Pipeline** (`LADDER Random <Disease> Pipeline.ipynb`): for every real community, builds a
  same-size replacement community by repeatedly drawing random genes from the full pool of genes across
  all of that disease's communities This removes shared biological function/complex membership
  by construction. (`<Disease>_random_communities.txt`).
- **Mixed Pipeline** (`LADDER Mixed <Disease> Pipeline.ipynb`): for every real community, keeps a random
  half of its genes and replaces the other half with genes drawn at random from the rest of that
  disease's gene pool (excluding the original community's own genes), then shuffles the combined list.
  This is the semi-plausible "50% Shuffled" control — same size, half-original signal, half noise
  (`random_communities_<Disease>.txt`).

### Annotation and validation (both pipelines, all diseases)

Each Random/Mixed community file is run through the same two-stage process used everywhere else in this
project:

1. **Annotation** (`OptimizedPathwayAnalyzer`): the same enrichment-driven + direct dual-prompt LLM
   annotation as Task 1, then process names are cleaned of embedded confidence-score suffixes
   (e.g. `Name (0.68)` -> `Name`).
2. **Validation** (`GeneSetValidator`): the same JIF-filtered, literature-grounded validator as Task 1,
   run against that disease's paper database, producing a `Final_Process` / `Final_Confidence` per
   community.

This produces one validation CSV per disease per condition — 9 files total
(3 diseases x {Original (from Task 1), Random, 50% Shuffled}).

### GMM.ipynb — fixing the confidence thresholds

1. Merges the three diseases' **Original** validation files (i.e. Task 1's real output, not
   regenerated here) into `Original_Validation_Merged.csv`, and separately merges the three Random and
   three Partial (Mixed) files into `Random_Validation_Merged.csv` and `Partial_Validation_Merged.csv`.
   All nine are also merged together into `All_Validation_Merged.csv`.
2. Fits a Gaussian Mixture Model to `Final_Confidence` **on the Original set only**, sorts
   the three component means ascending, and sets the Low/Medium/High boundary at the midpoint between
   each pair of adjacent means. These thresholds are then fixed and reused everywhere confidence tiers
   are reported in this project (Task 2, Task 4, etc.):

   | Tier   | Range       |
   | ------ | ----------- |
   | Low    | 0.00 – 0.45 |
   | Medium | 0.45 – 0.77 |
   | High   | 0.77 – 0.97 |

### R Visualization.ipynb — reading out the result

Loads `Original_Validation_Merged.csv`, `Random_Validation_Merged.csv`, and
`Partial_Validation_Merged.csv`, bins every row into the fixed Low/Medium/High tiers above, and produces
a raincloud plot of the confidence distribution per dataset plus a stacked bar chart of tier proportions
per dataset — the figure referenced in the manuscript (`confidence_analysis_figurev2.pdf`).

---

## 2. Requirements

- Python 3.9+ and Jupyter
- `pandas`, `numpy`, `scikit-learn`, `gseapy`, `requests`, `tqdm`
- CORUM complex data (`CORUM_dat.csv`) for the CORUM-disjoint random gene sampling
- A DeepSeek API key (`DEEPSEEK_API_KEY`) for annotation and validation
- The Clarivate JIF journal list (`journals_filtered_JIF_ge_4.csv`) for the validator's quality filter
- R 4.x with `ggplot2`, `dplyr`, `tidyr`, `patchwork`, `ggdist`, `scales`

```bash
pip install pandas numpy scikit-learn gseapy requests tqdm jupyter
export DEEPSEEK_API_KEY="your_key_here"
```
