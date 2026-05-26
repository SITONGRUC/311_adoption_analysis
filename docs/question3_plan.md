# Question 3 — Hypothesis Test Plan
## Does 311 adoption change city government spending?

---

## Data coverage — read this first

Census Individual Unit Finance files are publicly available for **2017–2022 only** (6 years).
This has direct consequences for every design choice below.

**What "2017–2022 only" means in practice:**

| Adoption cohort | k range in panel | Pre-period available? | Post-period available? |
|---|---|---|---|
| Adopted 1996–2016 | k ≥ 1 (all post) | No | Yes — always treated |
| Adopted 2017 | k = 0..5 | No | Yes — adoption year + after |
| Adopted 2018 | k = −1..4 | Reference year only (k=−1) | Yes |
| **Adopted 2019–2022** | **k = −2..+3 or less** | **Yes — testable pre-trend** | **Yes** |
| Adopted 2023–2025 | k = −8..−3 (all pre) | Yes | No — not yet treated |

The event study can only test parallel trends for cities that adopted **2019–2022**.
Cities adopting earlier are always-treated in this panel; cities adopting later have
no post-period observations.

---

## Design

**Baseline regression — TWFE:**
```
log(expenditure_pc)_ct = β · post311_ct + α_c + γ_t + ε_ct
```
`α_c` = city FE, `γ_t` = year FE, SE clustered at city level.

**Goodman-Bacon note:** With staggered adoption, `β` is a weighted average of
heterogeneous 2×2 DiDs. Some weights can be negative when early adopters serve as
the implicit control for later adopters. The scalar `β` is reported for completeness;
the event study is the primary exhibit.

**R² note:** pyfixest reports both overall R² and R² Within. In TWFE, the overall
R² is typically ~0.97 because city FEs absorb stable between-city spending levels.
The meaningful measure is **R² Within** — the share of within-city variation explained
by `post311`. A near-zero R² Within alongside a near-zero coefficient is internally
consistent and confirms the null.

---

## Samples

### Baseline TWFE sample
| Group | N | Role |
|---|---|---|
| `adopted` (year known) | 99 | Treated at their adoption year |
| `not_adopted` | 54 | Never-treated controls |
| `adopted_unknown_year` | 148 | **Excluded** — no treatment timing |

### Event study sample (separate, more restrictive)
| Group | N | Role |
|---|---|---|
| Adopted **2019–2022** | 18 | Treated — have both pre-period (k ≤ −2) and post-period (k ≥ 0) |
| `not_adopted` | 54 | Controls |

**Why 2019–2022:** Cities adopting in 2019 have k = −2 in 2017, giving one non-reference
pre-period observation. Cities adopting in 2018 have only k = −1 (the reference year),
so they contribute no testable pre-trend evidence. Cities adopting after 2022 have no
post-period in the panel.

**Event window: [−3, +3].** This is the widest window where:
- Every treated cohort has at least one non-reference pre-period observation
- N per cell ≥ 8 (k=+3 has 3 cities — do not interpret)

Actual N per k cell (treated cities only):

| k | −3 | −2 | −1 (ref) | 0 | +1 | +2 | +3 |
|---|---|---|---|---|---|---|---|
| N cities | 14 | 15 | 18 | 16 | 10 | 8 | 3 |

**k = 0 caveat:** Adoption year itself. Fiscal year data may not reflect a mid-year
adoption; k=0 mixes pre- and post-adoption periods. Do not over-interpret this cell.

---

## Data download

**Source:** Census Annual Survey of Government Finances — Individual Unit Data

**Critical URL note:** Files must be downloaded from the `tables` directory,
**not** the `datasets` directory. The `datasets` directory has older, differently-
formatted files with 14-char government IDs (34-char lines). The `tables` directory
files use 12-char IDs (32-char lines) and are available for all 6 required years.

```
https://www2.census.gov/programs-surveys/gov-finances/tables/2017/2017_Individual_Unit_File.zip
https://www2.census.gov/programs-surveys/gov-finances/tables/2018/2018_Individual_Unit_File.zip
https://www2.census.gov/programs-surveys/gov-finances/tables/2019/2019_Individual_Unit_File.zip
https://www2.census.gov/programs-surveys/gov-finances/tables/2020/2020_Individual_Unit_File.zip
https://www2.census.gov/programs-surveys/gov-finances/tables/2021/2021_Individual_Unit_File.zip
https://www2.census.gov/programs-surveys/gov-finances/tables/2022/2022_Individual_Unit_File.zip
```

Cache ZIPs to `data_raw/indFin/`. Use pattern matching on ZIP contents (not hardcoded
filenames) because 2022 places files inside a subdirectory.

### Files inside each ZIP

| Pattern | File | Description |
|---|---|---|
| `Fin_PID` | `Fin_PID_{YEAR}.txt` | Government identifier + place FIPS + population |
| `FinEstDAT` | `{YEAR}FinEstDAT_*.txt` | Finance data — one row per (government, item) |

---

## File format — exact field layouts

### PID file — 146 chars per line

| Field | Slice (0-indexed) | Notes |
|---|---|---|
| Government ID | `[0:12]` | Matches finance file govid exactly |
| **Type code** | `[2]` | **Filter: keep `'2'` = municipalities only** |
| Government name | `[12:76]` | Strip whitespace |
| County name | `[76:111]` | Not needed |
| **FIPS place code** | `[111:116]` | 5-char place FIPS → used to build GEOID |
| Population | `[116:125]` | In persons; convert to int |

```python
GEOID       = line[0:2].zfill(2) + line[111:116].strip().zfill(5)   # 7-digit
is_muni     = line[2] == '2'
pop_survey  = int(line[116:125].strip()) if line[116:125].strip().isdigit() else None
```

**Coverage note:** 2017 and 2022 are Census of Governments enumeration years
(~19,400 municipal rows). 2018–2021 are annual survey years (~3,500–3,900 rows).
The regression panel is therefore unbalanced — cities appear in 5–6 of 6 years.
This is expected and acceptable for TWFE.

### Finance file — 32 chars per line

| Field | Slice (0-indexed) | Notes |
|---|---|---|
| Government ID | `[0:12]` | Matches PID govid |
| Item code | `[12:15]` | 3-char code — see classification below |
| **Amount** | `[15:27]` | Right-justified; **thousands of dollars** |
| Year | `[27:31]` | 4-digit year |
| Flag | `[31]` | R=revised, E=estimated |

---

## Item code classification

### Include in total general expenditure

| Prefix | Meaning | Example codes |
|---|---|---|
| `E##` | Current operations by function | E23 (financial admin), E24 (fire), E62 (police), E80 (sewerage), E89 (general NEC) |
| `F##` | Capital construction by function | F44 (highways), F62 (police) |
| `G##` | Other capital outlay | G44 (highways) |
| `I89` | General interest on debt | I89 only |

### Exclude — NOT expenditure flows

| Items | Reason |
|---|---|
| `##90`, `##91`, `##92`, `##93`, `##94` | Utility enterprise (liquor/water/electric/gas/transit) — enterprise funds, not general govt |
| `19T`, `19U`, `24T`, `29U`, `34T`, `39U`, `44T`, `49U`, `61V`, `64V` | **Debt stock items** — beginning/ending balances, issued, retired. These have very large values and will corrupt the outcome if included |
| `T##` | Tax revenues |
| `A##` | Charges and fees (revenue) |
| `B##`, `C##` | Intergovernmental revenue |
| `W##` | Debt/cash/securities balance sheet stocks |

```python
UTIL_SUFFIXES = {'90', '91', '92', '93', '94'}
EXP_PREFIXES  = {'E', 'F', 'G'}
INTEREST_CODE = 'I89'

def is_general_expenditure(item: str) -> bool:
    item = item.strip()
    if item == INTEREST_CODE:
        return True
    if len(item) == 3 and item[0] in EXP_PREFIXES:
        return item[1:] not in UTIL_SUFFIXES
    return False
```

**Sanity check:** Chicago 2018 total general expenditure ≈ **$7.8–8.1B** (~$2,900–3,000
per capita). If your value is an order of magnitude higher, you are including debt stock
items (19T, 49U etc.) by mistake.

---

## Matching: finance data → city_311_analysis.csv

**Method: direct GEOID merge — no name matching needed.**

The PID file contains the Census FIPS place code (positions 111–116). Combined with the
state FIPS (positions 0–1), this gives the same 7-digit GEOID used in
`city_311_analysis.csv`.

```python
# Build GEOID lookup from PID file for each year
gid_df['GEOID'] = gid_df['state_fips'].str.zfill(2) + gid_df['place_fips'].str.zfill(5)

# Direct merge — no fuzzy matching
panel = raw_panel.merge(cities[['GEOID', 'adoption_status', 'adoption_year_clean', ...]],
                        on='GEOID', how='inner')
```

**Achieved match rate:** 153 / 153 (100%) for the regression sample.

**Drop the GEOID column from the finance panel before merging** to avoid duplicate
column conflicts (pandas will silently rename to `GEOID_x` / `GEOID_y`).

---

## Panel construction

```python
# Post-treatment indicator
def make_post(row):
    if row['adoption_status'] == 'not_adopted':   return 0
    if pd.isna(row['adoption_year_clean']):        return np.nan
    return int(row['year'] >= row['adoption_year_clean'])

panel['post311']        = panel.apply(make_post, axis=1)
panel['event_time']     = np.where(panel['adoption_status'] == 'adopted',
                                    panel['year'] - panel['adoption_year_clean'], np.nan)

# BASELINE: event_time_bin clipped to [-5, +5]  (unused in event study below)
panel['event_time_bin'] = panel['event_time'].clip(-5, 5)

# Outcome
panel['pop_denom']  = panel['pop_survey'].where(
    panel['pop_survey'].notna() & (panel['pop_survey'] > 50_000), panel['pop_2020'])
panel['log_exp_pc'] = np.log(panel['total_gen_exp'] * 1_000 / panel['pop_denom'])

# Drop rows with missing outcome or missing post311
panel = panel.dropna(subset=['log_exp_pc', 'post311'])
```

---

## Estimation (pyfixest throughout)

### Baseline TWFE — two specifications

```python
import pyfixest as pf

# Spec 1: full sample (all 99 adopted + 54 never-adopted)
fit_base = pf.feols('log_exp_pc ~ post311 | GEOID + year',
                    data=panel_reg, vcov={'CRV1': 'GEOID'})

# Spec 2: sensitivity check — exclude pre-2012 adopters.
# Note: 2012–2016 adopters are still always-treated in this 2017-2022 panel
# (all their k values are ≥ 1). This spec only drops 1996–2011 adopters.
late_mask = (
    (panel_reg['adoption_status'] == 'adopted') &
    (panel_reg['adoption_year_clean'] >= 2012)
) | (panel_reg['adoption_status'] == 'not_adopted')

fit_late = pf.feols('log_exp_pc ~ post311 | GEOID + year',
                    data=panel_reg[late_mask], vcov={'CRV1': 'GEOID'})
```

Both report: coefficient, SE, p-value, N obs, city FE ✓, year FE ✓, R² Within.

### Event study — corrected sample and window

```python
# Event study mask: adopted 2019-2022 (pre AND post in panel) + never-adopted
es_mask = (
    (panel_reg['adoption_status'] == 'adopted') &
    (panel_reg['adoption_year_clean'] >= 2019) &
    (panel_reg['adoption_year_clean'] <= 2022)
) | (panel_reg['adoption_status'] == 'not_adopted')

es_data = panel_reg[es_mask].copy()
es_data['event_time_bin'] = es_data['event_time'].clip(-3, 3).astype('Float64')
# Never-adopted cities: event_time_bin = NaN → pyfixest i() treats as zeros (controls)

fit_es = pf.feols('log_exp_pc ~ i(event_time_bin, ref=-1) | GEOID + year',
                  data=es_data, vcov={'CRV1': 'GEOID'})
```

### Coefficient extraction and plotting

```python
# pyfixest tidy() returns DataFrame indexed by Coefficient name
# Columns: Estimate, Std. Error, t value, Pr(>|t|), 2.5%, 97.5%
tidy = fit_es.tidy().reset_index()
tidy.columns = ['term','estimate','se','tstat','pval','ci_low','ci_high']
tidy['k'] = tidy['term'].str.extract(r'::(-?\d+(?:\.\d+)?)').astype(float)
# Add reference row: k=-1, estimate=0, ci=0

# pyfixest N observations: fit._N  (attribute, not method)
```

**Figure layout:** Two-panel figure (main: coefficients + CI; bottom: N cities per k cell).
This makes the thin support at extreme k values visible.

---

## Output files

| File | Description |
|---|---|
| `results/figures/fig_event_study.png` | Two-panel: event study coefficients + N per k |
| `results/tables/tab_twfe.tex` | pyfixest etable, two columns (full / late-adopter spec) |
| `results/tables/tab_twfe.csv` | CSV summary of both specs |

`pf.etable([fit_base, fit_late], type='tex', coef_fmt='b (se)')` produces the LaTeX table.

---

## Results (achieved)

### Baseline TWFE

| Spec | N obs | post311 coef | SE | p-value | R² Within |
|---|---|---|---|---|---|
| All treated (99) + controls (54) | 867 | −0.025 | 0.034 | 0.47 | 0.002 |
| Late adopters ≥2012 (59) + controls (54) | 630 | −0.020 | 0.035 | 0.56 | — |

### Event study (adopted 2019–2022, window [−3, +3])

| k | Estimate | SE | N treated cities |
|---|---|---|---|
| −3 | +0.047 | 0.060 | 14 |
| −2 | +0.012 | 0.033 | 15 |
| −1 | 0 (ref) | — | 18 |
| 0 | −0.044 | 0.041 | 16 |
| +1 | −0.094 | 0.084 | 10 |
| +2 | −0.051 | 0.127 | 8 |
| +3 | −0.079 | 0.218 | 3 ← do not interpret |

**Result: clean null.** Pre-period coefficients at k=−3 and k=−2 are small and
insignificant — consistent with parallel trends. Post-period coefficients are also
insignificant throughout. 311 adoption is not associated with a measurable change in
log per capita total general expenditure.

**Interpretation caveats:**
- 18 treated cities is a small sample; the event study is underpowered
- The null is informative: 311 is an administrative routing system whose direct fiscal
  footprint is small relative to total city spending
- Sample skews toward larger cities (mean pop 497k for adopted-year-known cities)
- Panel covers only 2017–2022; any fiscal effect materialising before 2017 is undetectable

---

## Execution order

```
[1] Download 6 ZIPs (2017–2022) from tables directory.
    Default (CACHE_TO_DISK=False): stream each ZIP into RAM, parse, discard bytes.
    Set CACHE_TO_DISK=True in the code to save ZIPs to data_raw/indFin/ for faster re-runs.

[2] For each year:
    - Parse PID: filter line[2]=='2', extract govid + GEOID + pop_survey
    - Parse finance: filter to muni govids, sum is_general_expenditure() items
    - Merge PID + finance on govid → one row per (govid, year)

[3] Load city_311_analysis.csv; merge on GEOID (drop duplicate GEOID from finance
    panel first to avoid _x/_y conflict) → 153/153 matched

[4] Build panel variables: post311, event_time, log_exp_pc

[5] Estimate baseline TWFE (2 specs) → tab_twfe.tex + tab_twfe.csv

[6] Define event study sample (adopted 2019–2022 + never-adopted);
    clip event_time to [-3,+3]; estimate event study → fig_event_study.png

Code: code/question3_analysis.ipynb
```

---

## Reproducibility requirements

| Requirement | Detail |
|---|---|
| Python packages | `pyfixest ≥ 0.50`, `pandas`, `numpy`, `requests`, `zipfile`, `matplotlib` |
| URL | `tables` directory only — `datasets` uses incompatible format |
| Finance file line length | 32 chars (not 34) — amount at `[15:27]`, year at `[27:31]` |
| Government type filter | `line[2] == '2'` in PID file |
| GEOID construction | `line[0:2].zfill(2) + line[111:116].strip().zfill(5)` |
| Debt item exclusion | `19T`, `19U`, `24T`, `29U`, `34T`, `39U`, `44T`, `49U` — exclude these explicitly |
| Event study sample | Adopted **2019–2022** + never-adopted (not 2012+) |
| Event window | [−3, +3] — not [−5, +5] |
| pyfixest i() syntax | `i(event_time_bin, ref=-1)` with `Float64` dtype |
| N per event-time cell | Must be reported alongside coefficients |
| `fit._N` | Attribute (not method) for observation count in pyfixest |
| Row counts | panel: 867 rows, 153 cities, 6 years (unbalanced) |
