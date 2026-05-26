# 311 Adoption Analysis — Reproduction Package

Candidate #85 | Data task submission

---

## What this repo contains

A full reproduction package for a three-part analysis of 311 non-emergency service adoption
across US cities:

- **Part 1** — Data compilation: when did US cities adopt 311?
- **Part 2** — Descriptive analysis: timing and geography of adoption
- **Part 3** — Hypothesis test: does 311 adoption change city government spending?

---

## File tree

```
reproduce_pkg/
├── README.md
├── .gitignore
│
├── code/
│   ├── step1_city_list.ipynb          # build 314-city universe from Census PEP + Gazetteer
│   ├── step2_extract_obyrne.ipynb     # extract O'Byrne (2015) binary adoption data from PDF
│   ├── step2_parse_search_log.ipynb   # parse 316 AI search logs → search_results.csv
│   ├── step3_verify_and_merge.ipynb   # cross-validate search vs O'Byrne; produce audit
│   ├── step4_analytical_sample.ipynb  # apply definitional filter → city_311_analysis.csv
│   ├── question2_descriptive.ipynb    # Part 2 figures and summary table
│   └── question3_analysis.ipynb       # Part 3 TWFE + event study (downloads Census IndFin)
│
├── data_raw/
│   ├── sub-est2023.csv                # Census PEP sub-county population estimates
│   ├── 2023_Gaz_place_national.txt    # Census Gazetteer (place geography)
│   ├── obyrne_2015_dissertation.pdf   # O'Byrne (2015) — external validation source
│   └── 311_search_log/               # 316 .txt files, one per city (AI search output)
│
├── data_clean/
│   ├── city_list.csv                  # 314 rows — city universe
│   ├── obyrne_extracted.csv           # 242 rows — O'Byrne binary adoption data
│   ├── search_results.csv             # 314 rows — parsed AI search results
│   ├── city_311_final.csv             # 314 rows — merged + cross-validated
│   ├── city_311_analysis.csv          # 301 rows — canonical analysis file (Q2 & Q3 input)
│   ├── city_311_review.csv            #  13 rows — conflict/unclear cities (held out)
│   ├── search_audit.csv               # 314 rows — row-level audit flags
│   └── search_audit_metrics.csv       #  23 rows — aggregate audit statistics
│
├── docs/
│   ├── question1_plan.md              # full data compilation spec (Steps 1–4)
│   ├── question2_plan.md              # descriptive analysis spec
│   ├── question3_plan.md              # hypothesis test spec
│   └── submission_writeup_plan.md     # paper structure plan
│
├── results/
│   ├── figures/
│   │   ├── fig_adoption_timing.png
│   │   ├── fig_adoption_region.png
│   │   └── fig_event_study.png
│   └── tables/
│       ├── tab_adopter_summary.tex / .csv
│       └── tab_twfe.tex / .csv
│
├── paper/
│   ├── main.tex
│   └── sections/
│       ├── part1.tex
│       ├── part2.tex
│       ├── part3.tex
│       └── refs.bib
│
└── logs/                              # empty placeholder (search step logs go here)
```

---

## Canonical execution order

Run in this exact order. Every step depends on the outputs of the previous one.

```
[1] code/step1_city_list.ipynb
      input:  data_raw/sub-est2023.csv
              data_raw/2023_Gaz_place_national.txt
      output: data_clean/city_list.csv  (314 cities)

[2] code/step2_extract_obyrne.ipynb
      input:  data_raw/obyrne_2015_dissertation.pdf
      output: data_clean/obyrne_extracted.csv  (242 rows)

[3] SEARCH STEP — agent/human action, not a notebook
      input:  data_clean/city_list.csv
      output: data_raw/311_search_log/*.txt  (316 files)
      note:   Already complete. Do not re-run unless intentionally refreshing.
              See docs/question1_plan.md Step 2 for full instructions.

[4] code/step2_parse_search_log.ipynb
      input:  data_raw/311_search_log/*.txt
              data_clean/city_list.csv
      output: data_clean/search_results.csv  (314 rows)

[5] code/step3_verify_and_merge.ipynb
      input:  data_clean/search_results.csv
              data_clean/obyrne_extracted.csv
      output: data_clean/city_311_final.csv   (314 rows)
              data_clean/search_audit.csv      (314 rows)
              data_clean/search_audit_metrics.csv  (23 rows)

[6] code/step4_analytical_sample.ipynb
      input:  data_clean/city_311_final.csv
      output: data_clean/city_311_analysis.csv  (301 rows) <- canonical input for Q2 & Q3
              data_clean/city_311_review.csv     (13 rows)

[7] code/question2_descriptive.ipynb
      input:  data_clean/city_311_analysis.csv
      output: results/figures/fig_adoption_timing.png
              results/figures/fig_adoption_region.png
              results/tables/tab_adopter_summary.tex
              results/tables/tab_adopter_summary.csv

[8] code/question3_analysis.ipynb
      input:  data_clean/city_311_analysis.csv
              Census IndFin 2017-2022 (downloaded from web at runtime)
      output: results/figures/fig_event_study.png
              results/tables/tab_twfe.tex
              results/tables/tab_twfe.csv
```

---

## Canonical files — use these, not older versions

| File | Rows | Description |
|---|---|---|
| `data_clean/city_list.csv` | 314 | City universe (pop >= 100k in 2020) |
| `data_clean/obyrne_extracted.csv` | 242 | O'Byrne (2015) binary adoption data |
| `data_clean/search_results.csv` | 314 | Web-search adoption years |
| `data_clean/city_311_final.csv` | 314 | Verified adoption dataset |
| `data_clean/city_311_analysis.csv` | **301** | **Main analysis file — use for Q2 and Q3** |
| `data_clean/city_311_review.csv` | 13 | Conflict/unclear cities excluded from analysis |

`city_311_analysis.csv` is the single entry point for both Part 2 and Part 3.
Do not read `city_311_final.csv` in Q2 or Q3.

---

## Key variable names — use exactly these

| Variable | File | Type | Description |
|---|---|---|---|
| `GEOID` | all files | str 7-digit | Primary key; always read with `dtype={'GEOID': str}` |
| `adoption_status` | city_311_analysis.csv | str | `adopted` / `adopted_unknown_year` / `not_adopted` |
| `adoption_year_clean` | city_311_analysis.csv | float64 after CSV read (int values or NaN) | Formal-311 year; NaN = no reliable year |
| `status_final` | city_311_final.csv | str | `adopted_year_verified` / `adopted_year_unknown` / `not_adopted` / `conflict_fp` / `not_adopted_unclear` |
| `post311` | constructed in Q3 | float | 1 if year >= adoption year, 0 if never-adopted, NaN if unknown-year |

**Common confusion:** `adopted_year_unknown` (value of `status_final` in city_311_final.csv)
is NOT the same as `adopted_unknown_year` (value of `adoption_status` in city_311_analysis.csv).
They are in different files. Always use `adoption_status` from `city_311_analysis.csv` in Q2/Q3.

---

## Expected output numbers

| Output file | Key numbers |
|---|---|
| `results/figures/fig_adoption_timing.png` | 99 cities in timing sample; n_unknown = 148 |
| `results/figures/fig_adoption_region.png` | 301 cities across 4 Census regions |
| `results/tables/tab_adopter_summary.tex` | 7 data rows (N + 2 pop rows + 4 region rows) |
| `results/figures/fig_event_study.png` | 18 treated (adopted 2019-2022); 54 controls; window [-3, +3] |
| `results/tables/tab_twfe.tex` | 2 columns; post311 coef = -0.025 (SE 0.034, p = 0.467) |

---

## Plan documents and AI-agent reproducibility

| File | Covers |
|---|---|
| `docs/question1_plan.md` | Full pipeline for data compilation (Steps 1–4): city universe, search protocol, O'Byrne extraction, verification, definitional filter |
| `docs/question2_plan.md` | Descriptive analysis specification: exhibit list, variable usage, regional share calculation |
| `docs/question3_plan.md` | Hypothesis test specification: TWFE design, event-study sample logic, output format |

**These three plan files are sufficient to reproduce this project with an AI coding agent.**
Provide them to the agent together with `city_311_analysis.csv` (the canonical analysis file)
and the archived search logs in `data_raw/311_search_log/`. The agent can then execute
every notebook in order without further instruction. Q1 is conditionally reproducible
(requires the archived search logs; the web search step itself is not deterministic).
Q2 and Q3 are fully deterministic given `city_311_analysis.csv`.

Each plan documents every design decision made during development — filter rules,
variable definitions, edge-case fixes, and output format — so that a fresh agent
session can reconstruct the analysis without access to the original conversation history.

---

## Raw data files to download manually

Two large Census files are **not included** in this zip (to keep the archive small).
Download them before running `step1_city_list.ipynb`:

**1. Census Population Estimates Program (PEP) sub-county file**
- Save as: `data_raw/sub-est2023.csv`
- URL: https://www2.census.gov/programs-surveys/popest/datasets/2020-2023/cities/totals/sub-est2023.csv

**2. Census Gazetteer (place geography)**
- Save as: `data_raw/2023_Gaz_place_national.zip` and unzip to `data_raw/2023_Gaz_place_national.txt`
- URL: https://www2.census.gov/geo/docs/maps-data/data/gazetteer/2023_Gazetteer/2023_Gaz_place_national.zip

Steps 2–8 do not need these files. If you only want to reproduce Parts 2 and 3,
start from `data_clean/city_311_analysis.csv` (already included) and skip step 1.

---

## Dependencies

```
Python 3.11+
pandas, numpy, matplotlib, requests, pyfixest >= 0.50
pdftotext (poppler): brew install poppler   # for step2_extract_obyrne only
```

Internet access required for `question3_analysis.ipynb` (downloads ~50 MB of
Census IndFin ZIPs at runtime). All other notebooks run offline.
