# Machine-Learning Pharmacovigilance of Drug-Associated Hidradenitis Suppurativa in FAERS

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.12](https://img.shields.io/badge/python-3.12-blue.svg)](https://www.python.org/)

Reproducible analysis code accompanying the manuscript "Patient-Level Risk Characterization of Drug-Associated Hidradenitis Suppurativa Using Machine Learning" (Maas et al.). This repository contains the full pipeline used to characterize drug-associated hidradenitis suppurativa (HS) as a rare paradoxical adverse event using the FDA Adverse Event Reporting System (FAERS).

> **Data note:** FAERS is public-domain data released by the FDA, but the raw quarterly files are large and are **not** redistributed here. See [Obtaining the data](#obtaining-the-faers-data). All results are report-based (not patient-based); FAERS does not support incidence or prevalence estimation.

---

## Overview / background

TNF-inhibitors (TNFi) and other immunomodulators can paradoxically induce or worsen HS. Because HS is a rare event in spontaneous-reporting data, standard classification framing is uninformative. This project builds a rare-event machine-learning framework — combining disproportionality analysis with calibrated risk ranking and multi-model interpretation — to characterize which drugs, indications, and agent–indication combinations carry HS reporting signals.

## Scientific objective

To develop and apply a reproducible framework that (1) quantifies drug–HS disproportionality (reporting odds ratios), (2) ranks patient-level HS risk among TNFi-exposed reports using calibrated supervised models evaluated by precision-recall and top-percentile enrichment (not accuracy), and (3) identifies concordant predictors and indication × TNFi-agent interactions.

## Methods summary

- **Cohorts.** Reports with a MedDRA preferred term containing "hidradenitis" (concept `37320281`), stratified into drug-worsened (HS also an indication) vs. drug-induced (no HS indication).
- **Disproportionality.** Haldane-corrected reporting odds ratios (RORs) with Wald 95% CIs and Benjamini–Hochberg FDR within HS-history × sex strata.
- **Rare-event ML.** 70/30 stratified split; 10-fold cross-validated feature selection (29 of 62 candidates); five supervised models (elastic net, ridge logistic regression, random forest, XGBoost, explainable boosting machine) plus an unsupervised isolation-forest baseline; isotonic calibration; evaluation by PR-AUC, top-percentile enrichment, and calibration.
- **Interpretation.** TreeSHAP, EBM shape functions, cross-model permutation-importance concordance, and a formal indication × TNFi-agent interaction logistic regression (marginal standardization).

See [`docs/DATA_DICTIONARY.md`](docs/DATA_DICTIONARY.md) for the input tables and schema.

## Repository structure

```
faers-hs-ml/
├── README.md                     # this file
├── LICENSE                       # MIT
├── CITATION.cff                  # how to cite
├── requirements.txt              # Python dependencies (pinned)
├── environment.yml               # optional conda environment
├── .gitignore
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── REPRODUCIBILITY.md            # reproducibility audit + known caveats
├── RELEASE_CHECKLIST.md          # pre-publication checklist
├── .github/                      # issue + PR templates
├── notebooks/
│   └── HS_ML_Pipeline_Reorganized.ipynb   # full pipeline (run top-to-bottom)
├── src/
│   ├── hs_ml_pipeline.py         # linear script export of the notebook
│   └── figures/                  # standalone figure-generation scripts
├── data/
│   ├── raw/                      # (gitignored) place downloaded FAERS CSVs here
│   ├── processed/                # (gitignored) checkpoints / intermediate frames
│   └── dictionaries/
│       └── sanitized_outcome_dictionary.csv   # concept ID -> preferred term
├── results/
│   ├── figures/                  # (gitignored) generated figures
│   └── tables/
│       └── Full_Supplemental_Tables_v4.xlsx   # supplementary tables
└── docs/
    └── DATA_DICTIONARY.md        # FAERS table + column reference
```

## Installation

```bash
git clone https://github.com/<ORG_OR_USER>/faers-hs-ml.git
cd faers-hs-ml

# Option A — pip + virtualenv
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Option B — conda
conda env create -f environment.yml
conda activate faers-hs
```

The notebook was developed on Google Colab; it runs anywhere once the data path is set (see below). Colab-only calls (`google.colab`, `drive.mount`) are guarded and skipped off-Colab.

## Required software and package versions

Python **3.12.13**, with the exact tested versions:

| Package | Version | Role |
|---|---|---|
| scikit-learn | 1.6.1 | RF, penalized/elastic-net LR, isotonic, permutation importance, isolation forest |
| xgboost | 3.3.0 | gradient-boosted trees |
| interpret | 0.7.8 | explainable boosting machine |
| statsmodels | 0.14.6 | interaction regression, BH-FDR |
| shap | 0.52.0 | TreeSHAP |
| betacal | (see lock) | beta calibration |
| numpy / pandas / scipy | latest tested | data + statistics |
| matplotlib / seaborn / adjustText | latest tested | figures |
| openpyxl | latest tested | Excel export |

Run `pip freeze > requirements-lock.txt` in your environment to capture an exact lock.

## Obtaining the FAERS data

FAERS is public-domain data from the FDA; it is not redistributed here due to size.

1. Download the FAERS quarterly ASCII/XML files from the FDA:
   https://fis.fda.gov/extensions/FPD-QDE-FAERS/FPD-QDE-FAERS.html
   (this project used quarters spanning **2004–2023**).
2. Standardize drug and event terms to concept identifiers (OMOP/MedDRA) as described in the manuscript Methods, producing the sanitized tables listed in [`docs/DATA_DICTIONARY.md`](docs/DATA_DICTIONARY.md):
   `sanitized_demographics.csv`, `sanitized_drug_data.csv`, `sanitized_outcome_data.csv`,
   `sanitized_ther.csv`, `standard_case_indication.csv`, `standard_case_outcome_category.csv`,
   `table_of_indications.csv`, `Updated_Drug_Dictionary_Fullnames.csv`, and the provided
   `sanitized_outcome_dictionary.csv`.
3. Place these files in `data/raw/`.

## Reproducing the analysis (step by step)

1. Complete installation and place the sanitized CSVs in `data/raw/`.
2. Set the data directory. In the notebook's **Section 0**, set the data path to your local folder, e.g.
   ```python
   FAERS_DIR = Path("data/raw")        # off-Colab
   # or, on Colab, the mounted Drive folder
   ```
3. Run `notebooks/HS_ML_Pipeline_Reorganized.ipynb` **top to bottom**. The logical flow is:
   setup → data loading → cohorts → ROR statistics → comorbidity/demographics →
   feature engineering → split + feature selection → model training → calibration →
   evaluation + enrichment → SHAP + interaction regression → multi-model framework →
   TNFi rates / volcano / time-to-onset → cross-model concordance → main + supplemental
   figures → tables → checkpoint save.
4. **Resume without re-running:** after one full run, the final **Checkpoint Save** cell writes
   `checkpoint_dataframes_v2.pkl` + `checkpoint_models_full.pkl`; the **Resume Point** cell (immediately after) restores every object so you can jump straight to the figures.
5. Fixed random seeds throughout (feature selection `456`, XGBoost/EBM `42`, elastic net `123`, SHAP `101`, permutation importance `42/792`) make results deterministic.

## Expected outputs

- **Main figures** (`Figure1`–`Figure4`) and **supplemental figures** (`eFigure1`–`eFigure10`) as PNG + PDF in `results/figures/`.
- **Tables** T4–T12 and the supplementary workbook in `results/tables/`.
- **CSV artifacts**: `stats_combined` (disproportionality), multi-model performance/enrichment, interaction-regression output, SHAP importances, cross-model permutation importance.
- A manifest cell (`FINAL EXPORT`) relabels every figure by its manuscript number.

## Citation

If you use this code, please cite the manuscript 'Patient-Level Risk Characterization of Drug-Associated Hidradenitis Suppurativa Using Machine Learning' by Kyle Maas et al



## License

Released under the [MIT License](LICENSE).

## Contact

- Corresponding author: Kyle Maas — `<kyle.p.maas@vanderbilt.edu>`
- Code / repository issues: please open a [GitHub Issue](../../issues).

## Acknowledgements

Data derived from the FDA Adverse Event Reporting System (FAERS). We thank the FDA for maintaining FAERS as a public resource. Code generation was assisted by Gemini 3.1 Pro; all code and outputs were reviewed and validated by the authors.
