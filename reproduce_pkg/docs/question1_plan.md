# Question 1 — Data Compilation Plan
## Compile a dataset of when US cities adopted 311 systems

> *Include at minimum city name, state, adoption year, and a source for each observation.
> Aim for broad coverage; document your methodology.*

---

## How to use this document

This plan is written so that a new agent or researcher can **reproduce the full pipeline
from scratch** by following these steps in order. Every design choice, filter rule,
regex, and edge-case fix is documented here. The only step that cannot be reproduced
deterministically is the web search itself (Step 2) — the 316 search log files in
`data_raw/311_search_log/` are treated as raw data and must be present before running
the downstream notebooks.

**Conditional reproducibility:** Question 1 is fully reproducible **given the archived
search logs** in `data_raw/311_search_log/` (316 txt files). Steps 1, 1B, 2-parse, 3,
and 4 are deterministic. Step 2 (the web search itself) is not — re-running it with a
different agent will produce different adoption years for some cities. To reproduce this
exact dataset, use the existing search logs as raw data.

**To reproduce from scratch:**
1. Download the three raw files listed below.
2. Run `code/step1_city_list.ipynb` to produce `data_clean/city_list.csv`.
3. Using `city_list.csv` as your input, perform a web search for every city (Step 2) and save one txt file per city to `data_raw/311_search_log/`. This is the only manual/agent step.
4. Run the remaining four notebooks in order: `step2_extract_obyrne`, `step2_parse_search_log`, `step3_verify_and_merge`, `step4_analytical_sample`.

---

## Required files — download before running any code

---

#### 1. Census Population Estimates Program — Sub-County Totals

| Field | Detail |
|---|---|
| **Local path** | `data_raw/sub-est2023.csv` |
| **Source** | US Census Bureau, Population Estimates Program |
| **Direct URL** | https://www2.census.gov/programs-surveys/popest/datasets/2020-2023/cities/totals/sub-est2023.csv |
| **How to get** | Download directly from the URL above (public, no login required) |
| **Description** | Annual population estimates for all sub-county governmental units, 2020–2023. Used to identify incorporated places (SUMLEV = 162, FUNCSTAT = A) with 2020 population ≥ 100,000. |
| **Encoding** | `latin-1` — must pass `encoding='latin-1'` when reading with pandas |
| **Used by** | `code/step1_city_list.ipynb` |

---

#### 2. Census Gazetteer 2023 — Place Geographic Attributes

| Field | Detail |
|---|---|
| **Local path** | `data_raw/2023_Gaz_place_national.zip` and `data_raw/2023_Gaz_place_national.txt` |
| **Source** | US Census Bureau, Geography Division |
| **Direct URL** | https://www2.census.gov/geo/docs/maps-data/data/gazetteer/2023_Gazetteer/2023_Gaz_place_national.zip |
| **How to get** | Download the zip; unzip manually to produce `2023_Gaz_place_national.txt` in `data_raw/` |
| **Description** | National file of all Census-designated places with 7-digit GEOID, place type (LSAD), functional status (FUNCSTAT), land area (sq mi), and internal-point coordinates (lat/lon). Tab-separated. **Column names have trailing whitespace** — run `.str.strip()` on columns after reading. |
| **Used by** | `code/step1_city_list.ipynb` |

---

#### 3. O'Byrne (2015) Dissertation PDF

| Field | Detail |
|---|---|
| **Local path** | `data_raw/obyrne_2015_dissertation.pdf` |
| **Source** | O'Byrne, J.C. (2015). *The Diffusion and Evolution of 311 Citizen Service Centers in American Cities from 1996 to 2012*. Ph.D. Dissertation, Virginia Polytechnic Institute and State University. |
| **Handle page** | https://vtechworks.lib.vt.edu/handle/10919/52634 |
| **How to get** | Go to the handle page → click the "Files" tab → download the PDF → save as `data_raw/obyrne_2015_dissertation.pdf` |
| **Description** | Appendix A contains the logistic regression dataset: one row per US city with population > 100,000 in 2003, with binary 311 adoption status as of 2012. |
| **Used by** | `code/step2_extract_obyrne.ipynb` (requires `pdftotext` from poppler: `brew install poppler`) |

---

## Design decisions

### Why population ≥ 100,000 in 2020?

1. **O'Byrne (2015)** covers only cities with population > 100,000 in 2003. This gives
   a verification layer for the cities most likely to have adopted 311. Below 100k,
   there is no systematic prior research to cross-check against.
2. **Tractability.** Searching ~314 cities is feasible within the scope of this task.

### Why 2020 as the population timestamp?

The Census PEP sub-county file (`sub-est2023.csv`) starts from 2020 — it is the earliest
year available. 2020 is the most recent decennial Census base year, the most accurate
population benchmark for current city boundaries.

**Caveat:** Cities above 100k in 2003 but below 100k by 2020 drop out. Both edge cases
affect a small number of cities and are flagged as a limitation in the analysis.

### Why search first, verify with O'Byrne second?

O'Byrne (2015) gives binary adoption status (yes/no as of 2012) but **no adoption year**.
Since the adoption year is the central variable for Parts 2 and 3, web search is the
primary method. O'Byrne then serves as a verification and supplementary labeling
layer — not a data source for the main variable of interest.

---

## Folder structure

```
data_raw/               — original downloaded files and search logs; never hand-edited
  311_search_log/       — one txt file per city from Step 2 search
data_clean/             — processed, analysis-ready files output by notebooks
code/                   — one notebook per step, numbered to match this plan
docs/                   — this plan and other documentation
```

---

## Step 1 — Build the city list (population ≥ 100,000 in 2020)

**Status: COMPLETE**

**Goal:** Define the universe of cities in scope. Every subsequent step operates on this list.

### Filtering rules
1. Read `sub-est2023.csv` with `encoding='latin-1'`, `dtype={'STATE': str, 'COUNTY': str, 'PLACE': str}`
2. Keep rows where `SUMLEV == 162` (incorporated place) AND `FUNCSTAT == 'A'` (active government)
3. Keep rows where `POPESTIMATE2020 >= 100_000`
4. Build 7-digit GEOID: `STATE.str.zfill(2) + PLACE.str.zfill(5)`
5. Read Gazetteer txt (tab-separated, `encoding='latin-1'`); run `.str.strip()` on all column names immediately after reading; the `GEOID` column also needs `.str.strip().str.zfill(7)`
6. Left-join PEP places to Gazetteer on `GEOID`; keep columns: `state_abbr` (from `USPS`), `place_type` (from `LSAD`), `funcstat` (from `FUNCSTAT`), `land_area_sqmi` (from `ALAND_SQMI`), `lat` (from `INTPTLAT`), `lon` (from `INTPTLONG`)

### Output: `data_clean/city_list.csv` — 314 cities across 44 states

| Column | Description |
|---|---|
| `GEOID` | 7-digit Census place FIPS (primary key for all merges) |
| `place_name` | Official Census place name (e.g., "Chicago city") |
| `state_abbr` | 2-letter state abbreviation |
| `state_name` | Full state name |
| `pop_2020` | 2020 population estimate (used for sample selection) |
| `pop_2021` | 2021 population estimate |
| `pop_2022` | 2022 population estimate |
| `pop_2023` | 2023 population estimate |
| `lat`, `lon` | Internal-point coordinates |
| `land_area_sqmi` | Land area in square miles |
| `place_type` | Census place type (e.g., "Incorporated Place") |
| `funcstat` | Functional status code (always "A" = active government) |

**Code:** `code/step1_city_list.ipynb`

---

## Step 1B — Extract O'Byrne (2015) adoption data from dissertation PDF

**Status: COMPLETE**

**Goal:** Parse Appendix A of O'Byrne (2015) to extract the binary 311 adoption indicator
for every city in his sample.

### Method — step by step

**1. Extract text:**
```python
result = subprocess.run(['pdftotext', '-layout', PDF_PATH, '-'],
                        capture_output=True, text=True)
full_text = result.stdout
```

**2. Isolate Appendix A:**
Find all positions of `r'Appendix A\.'` in `full_text`; take the *last* match (the two earlier
occurrences are in the table of contents). The section ends at the next `r'Appendix B\.'`.

**3. Parse data rows:**
Each data row ends with five numeric columns separated by large whitespace gaps.
Match with this regex against each line:
```
r'^(.+?)\s{2,}(\d|NA)\s+(\d|NA)\s+([\d,]+|NA)\s+([\d.]+|NA)\s+([\d,]+|NA)\s*$'
```
Groups: (city_fragment, Adopt, Mayor, Population, ViolentCrime, Damage).

**4. Handle multi-line city names:**
Some city names span two printed lines. Strategy: if the matched city_fragment does NOT
contain a state code, prepend the previous non-blank, non-header line to it.

State code detection pattern (used to decide whether a fragment is a complete city name):
```python
STATE_RE = re.compile(r',\s*[A-Z]{2}(\s|\(|$)')
```
If `STATE_RE.search(fragment)` is falsy, the fragment is incomplete — prepend `prev_name`.

Skip lines matching:
```
r'^(Governments|Adopt|Mayor|Population|Violent|Crime|Damage|Sources|Date|\d+\s*$)'
```

**Parsing numeric fields:**
For each of the five numeric columns (Adopt, Mayor, Population, ViolentCrime, Damage):
- If the matched value is `"NA"` → store as `None`
- Otherwise strip commas, then cast to `int` (Population, Damage, Adopt, Mayor) or `float` (ViolentCrime)

**5. Manual fixes for page-break names:**
After parsing, apply this lookup to correct fragments produced by mid-name page breaks:

| Raw fragment | Correct full name |
|---|---|
| `counties)` | `New York, NY` |
| `County of, PA` | `Philadelphia, PA` |
| `portion of Marion County)` | `Indianapolis, IN` |
| `(consolidated with Duval County)` | `Jacksonville, FL` |
| `County of, CA` | `San Francisco, CA` |
| `County, NC (City Portion)` | `Charlotte, NC` |
| `of Suffolk County)` | `Boston, MA` |
| `Washington, District of Columbia` | `Washington, DC` |
| `CO` | `Denver, CO` |
| `TN` | `Nashville, TN` |
| `(City portion)` | `Miami-Dade, FL` |
| `of, HI (City Portion)` | `Honolulu, HI` |
| `County Government, KY` | `Lexington-Fayette, KY` |
| `(City portion), KY` | `Louisville, KY` |
| `Government, GA` | `Columbus, GA` |
| `County Unified Government, KS` | `Kansas City, KS` |
| `(Ventura), CA` | `Ventura, CA` |
| `GA (City portion)` | `Athens-Clarke, GA` |

**6. Extract state abbreviation:**
Use `re.findall(r',\s*([A-Z]{2})(?:\s|\(|$)', city_raw)` and take the last match.
Special case: `Washington, DC` → state = `DC`.

### Output: `data_clean/obyrne_extracted.csv` — 242 rows

| Column | Description |
|---|---|
| `city_raw` | City name as it appears in O'Byrne's appendix |
| `state_abbr` | 2-letter state abbreviation |
| `adopt_by2012` | 1 = adopted 311 by 2012, 0 = not adopted, NaN = unclear |
| `mayor` | Mayor-council government dummy |
| `pop_2003` | City population in 2003 |
| `violent_crime_2003` | Violent crime rate per 100k in 2003 |
| `damage_2003` | Property damage rate in 2003 |
| `pre_label` | `adopted_by2012` / `not_adopted_by2012` / `unclear` |
| `source` | Always "O'Byrne (2015) Appendix A" |

**Expected distribution:** 53 adopted, 180 not adopted, 9 unclear.

**Code:** `code/step2_extract_obyrne.ipynb`

---

## Step 2 — Search all 314 cities for 311 adoption year

**Status: COMPLETE — 2026-04-22**

---

> **ACTION REQUIRED FOR ANY NEW AGENT REPRODUCING THIS STEP:**
>
> You must web-search every city in `data_clean/city_list.csv` (314 rows) and
> **save one txt file per city** into `data_raw/311_search_log/` before running
> any downstream notebook. The exact filename format and field format are specified
> below. Do not skip this step or proceed to Step 2 parse without all 314 files present.
>
> This search was originally performed by a Claude AI agent (claude-sonnet-4-5)
> in a single automated pass — not by hand. If you are re-running, you should do
> the same: iterate over every row in `city_list.csv` and run a web search for each
> city, writing the result to disk immediately before moving to the next city.

---

This is the only step that cannot be reproduced deterministically. The 316 search log
files in `data_raw/311_search_log/` encode the research judgments made in the original
run and are treated as raw data.

Implications for reproducibility:
- A re-run may find different sources or classify borderline cities differently.
- The original agent conducted one pass per city. Best practice would be multiple
  independent passes with reconciliation. The 138 `unknown` cities are partly a
  consequence of the single-pass approach.
- The manual corrections in Step 4 (Bridgeport year, Raleigh flag) were identified
  by a human reviewer — not produced by the search agent.

**Search procedure per city:**
1. Official city 311 webpage — look for "since YYYY", launch date, or system history
2. City press releases, mayor's office announcements, anniversary coverage
3. GovTech magazine, Governing magazine, local news
4. Search strings: `"[City] 311 launched"`, `"[City] 311 non-emergency service"`,
   `"[City] 311 history"`, `"[City] 311 citizen service center"`

**Log file format** — one file per city at `data_raw/311_search_log/[city_slug]_[STATE].txt`:

```
city: [Name as searched]
state: [2-letter code]
adoption_year: [YYYY or "unknown" or "no_311"]
source_url: [URL]
source_type: [official / news / secondary]
evidence_note: [One sentence describing what the source says]
```

**`adoption_year` values:**

| Value | Meaning |
|---|---|
| `YYYY` | Year 311 launched (4-digit integer) |
| `unknown` | City has 311 but launch year not found in any source |
| `no_311` | No evidence the city operates a 311 system |

**`source_type` values:**

| Value | Meaning |
|---|---|
| `official` | City government website, city press release, or official city document |
| `news` | Local newspaper, GovTech, Governing magazine, or other journalism |
| `secondary` | Third-party aggregator (e.g., SeeClickFix case study, app store listing) |

**Note on duplicate log files:** Two cities have two log files each due to slug collisions:
Lexington-Fayette KY and Kansas City KS. The parse notebook resolves duplicates by
keeping the row with the earliest confirmed year per GEOID (see Step 2 parse below).

**Coverage:**
- 316 log files covering all 314 cities
- 121 cities with confirmed adoption year
- 55 cities confirmed no 311 system
- 138 cities with 311 but year unknown

---

## Step 2 parse — Parse search logs into structured table

**Status: COMPLETE**

### Method — step by step

**1. Parse each txt file:**
For each `*.txt` file in `data_raw/311_search_log/`, split on newlines and partition each
line on the first `:`. The `source_url` and `evidence_note` fields may themselves contain
colons — use `line.partition(':')[2].strip()` (everything after the first colon) for both.
Collect six fields: `city`, `state`, `adoption_year`, `source_url`, `source_type`, `evidence_note`.

**2. Parse `adoption_year` into `year_found`:**
Use `re.search(r'\b(19|20)\d{2}\b', val)` to extract a 4-digit year. Treat
`unknown`, `no_311`, empty, or no match as `None`. Also normalise `adoption_year_raw`
to a clean string: the integer year as string, `"unknown"`, or `"no_311"`.

**3. Match to `city_list.csv` on (city, state) to get GEOID:**
This uses the `city` and `state` fields from *inside* the txt file — not the filename.

Normalization function (apply identically to both `city` from the log and `place_name`
from `city_list.csv`):
```python
SUFFIXES = re.compile(
    r'\b(city|town|village|borough|municipality|county|metro(politan)?'
    r'|government|consolidated|unified|urban|charter)\b',
    re.IGNORECASE
)
def norm(name):
    name = SUFFIXES.sub('', str(name))
    name = re.sub(r'[^a-z\s]', '', name.lower())
    return re.sub(r'\s+', ' ', name).strip()
```

Matching order:
1. Exact match on `(state_abbr, name_norm)` in a lookup dict built from `city_list`
2. If no exact match: `difflib.get_close_matches(name_norm, candidates, n=1, cutoff=0.82)`
   where `candidates` are all normalized names for that state
3. If still no match: `GEOID = None`, `match_type = 'no_match'`

All 316 log files match exactly (no fuzzy matches needed) because the search agent used
city names drawn from `city_list.csv` directly.

**4. Resolve duplicates:**
Sort by `year_found` ascending with NaN last; `drop_duplicates('GEOID', keep='first')`.
This keeps the earliest confirmed year when two log files cover the same city.

**5. Left-join to `city_list`:**
Merge matched records onto `city_list` on GEOID. Cities with no log file get
`adoption_year_raw = 'not_searched'` (none expected; all 314 were searched).

### Output: `data_clean/search_results.csv` — 314 rows

| Column | Description |
|---|---|
| `GEOID` | 7-digit Census place FIPS (joined from city_list) |
| `place_name` | Official Census place name |
| `state_abbr` | State abbreviation |
| `pop_2020` | 2020 population |
| `adoption_year_raw` | Clean string: integer year, "unknown", or "no_311" |
| `year_found` | Integer adoption year (null if unknown/no_311) |
| `source_url` | URL of the source |
| `source_type` | official / news / secondary |
| `evidence_note` | One-sentence description of the evidence |
| `match_type` | How city name was matched: "exact" or "fuzzy:MATCHED_NAME" |

**Code:** `code/step2_parse_search_log.ipynb`

---

## Step 3 — Verify search reliability with O'Byrne (2015); produce final dataset

**Status: COMPLETE — 2026-04-22**

### Purpose

Step 2 produced adoption years from web searches. **Step 3 asks: how reliable is that
search data?**

O'Byrne (2015) provides a fully independent binary classification — adopted / not adopted
by 2012 — derived from a systematic academic survey. We use it as an external audit layer.

Sub-goals:
1. **Reliability audit:** Confusion matrix, scalar metrics, conflict listing,
   source-type breakdown, year plausibility check. All metrics computed by code.
2. **Final dataset assembly:** Merge verified search results with O'Byrne labels.

### 3A — Reliability audit

**Match O'Byrne cities to `city_list.csv`:**
Use the same `norm()` function and fuzzy-match logic (cutoff 0.82) as Step 2 parse.

First, parse each `city_raw` value (from `obyrne_extracted.csv`) into a city name and
state abbreviation using this function:
```python
def parse_obyrne_city(raw):
    raw = re.sub(r'\(.*?\)', '', str(raw)).strip()   # strip parenthetical notes
    parts = raw.split(',')
    if len(parts) >= 2:
        city_part  = parts[0].strip()
        state_part = parts[-1].strip().split()[0]    # first word after last comma
        return city_part, state_part
    return raw, ''
```
This strips notes like "(includes 5 counties)" or "(City portion)" before splitting,
so "New York, NY (includes 5 counties)" → city="New York", state="NY".

Then apply `norm()` to the city part and match against the city_list lookup exactly, then
fuzzy if needed. **No FIXES dict is used here.** Cities that remain unmatched (16 total —
consolidated governments like Washington DC, Honolulu, Louisville that do not appear in
the 2020 city universe) receive `obyrne_label = 'not_in_obyrne'`.

226 exact matches; 16 remain unmatched.

**Confusion matrix** (restricted to cities in both datasets with definitive O'Byrne label):

|  | Search: year found | Search: no_311 | Search: unknown |
|---|---|---|---|
| O'Byrne: adopted_by2012 | TP | FN | ambiguous |
| O'Byrne: not_adopted_by2012 | FP | TN | ambiguous |

Binary scalar metrics exclude the `unknown` search rows.

**Corrected matrix:** Restrict FP to cities where `year_found ≤ 2012`. Cities where
`year_found > 2012` appear as FP in the raw matrix but are not real conflicts — O'Byrne's
survey simply predates the adoption.

**Scalar metrics to compute and save:**

| Metric | Formula |
|---|---|
| `agreement_rate` | (TP+TN)/total |
| `conflict_rate` | (FP+FN)/total |
| `fn_rate` | FN/(TP+FN) |
| `fp_rate` | FP/(TN+FP) |
| Same four metrics with `_corrected` suffix | using corrected matrix |
| `fp_post2012` | raw FP minus corrected FP |
| `fp_true` | corrected FP count |

All metrics saved to `data_clean/search_audit_metrics.csv`.

### 3B — O'Byrne merge and status_final assignment

**O'Byrne labels:**

| `adopt_by2012` | `obyrne_label` |
|---|---|
| 1 | `adopted_by2012` |
| 0 | `not_adopted_by2012` |
| NaN (matched but unclear) | `unclear` |
| (not matched to city_list) | `not_in_obyrne` |

**`status_final` decision table:**

| Search result | O'Byrne label | Condition | `status_final` |
|---|---|---|---|
| year found | not_adopted_by2012 | year ≤ 2012 | `conflict_fp` |
| year found | any other | — | `adopted_year_verified` |
| unknown | any | — | `adopted_year_unknown` ⚠ see note |
| no_311 | not_adopted_by2012 | — | `not_adopted` |
| no_311 | adopted_by2012 | — | `conflict_fn` |
| no_311 | not_in_obyrne | — | `not_adopted` |
| no_311 | unclear | — | `not_adopted_unclear` |

**Important:** `conflict_fp` only when year ≤ 2012 AND O'Byrne says not adopted.
Year > 2012 with O'Byrne "not_adopted" = `adopted_year_verified` (O'Byrne predates it).

**⚠ Variable name note — `adopted_year_unknown` vs `adopted_unknown_year`:**
These are two *different* column values in *different* files — do not confuse them.
- `status_final = 'adopted_year_unknown'` — value in `city_311_final.csv`; city has 311 but year not found in web search
- `adoption_status = 'adopted_unknown_year'` — value in `city_311_analysis.csv` (Step 4 output); city has **formal** 311 confirmed but no reliable year

A city reaches `adopted_unknown_year` in `adoption_status` by two paths: (a) `status_final = adopted_year_unknown` (search confirmed 311, no year), or (b) `status_final = adopted_year_verified` but downgraded by the Step 4 definitional filter (found year, but evidence lacked "311" branding). Path (b) requires **formal 311** confirmation; path (a) does not (it only requires search evidence of any 311-like system). This is why the name changes between the two files.

### Outputs

| File | Description |
|---|---|
| `data_clean/search_audit_metrics.csv` | Scalar audit metrics (23 rows: raw + corrected) |
| `data_clean/search_audit.csv` | Per-city audit flags (314 rows) |
| `data_clean/city_311_final.csv` | 314-row final dataset |

**`data_clean/city_311_final.csv` columns:**

| Column | Description |
|---|---|
| `GEOID` | 7-digit FIPS (primary key) |
| `place_name` | City name |
| `state_abbr` | State |
| `pop_2020` | 2020 population |
| `adoption_year` | Raw string: YYYY, "unknown", or "no_311" |
| `year_found` | Integer adoption year (null if unknown/no_311) |
| `status_final` | Status label (see table above) |
| `source_url` | Source for adoption year |
| `source_type` | official / news / secondary |
| `evidence_note` | One-sentence evidence summary |
| `obyrne_label` | O'Byrne label for cross-reference |
| `adopt_by2012` | Raw O'Byrne binary flag (1/0/NaN) |
| `audit_note` | Explanation for any conflict |

**Code:** `code/step3_verify_and_merge.ipynb`

---

## Step 4 — Analytical sample construction and definitional cleanup

**Status: COMPLETE — 2026-04-23**

**Goal:** Transform `city_311_final.csv` into a clean, analysis-ready flat file for
Parts 2 and 3. Three sub-steps: (A) formal-311 filter, (B) derive clean columns, (C) split.

This step changes nothing in the upstream files — it derives new columns and new outputs.

### 4A — Formal-311 definitional filter

**Standard:** A city is coded `adopted` only if it operates a dedicated **311 non-emergency
phone number or platform explicitly branded as "311"** by the city government. The "311"
designation must appear in the evidence.

**Not counted:**
- Digital-only citizen request apps without 311 number ("Fix It Plano", "MyBellevue", "Engage Toledo", etc.)
- General customer service hotlines without 311 branding
- CRM / tracker with no 311 label (e.g., Hialeah Citizen Request Tracker)
- App later replaced by a different 311 system (Bridgeport "BConnected" 2012)
- County-operated 311 where the city has no city-operated equivalent

**Automated screening rule:**
For every city where `year_found` is not null, use `re.search(r'311', evidence_note, re.IGNORECASE)`.
If "311" does not appear → clear `year_corrected` to NaN and set `definitional_note` to
explain the downgrade. This catches 10 cities in the current data:
Toledo OH, Bellevue WA, Joliet IL, Dayton OH, Victorville CA, Midland TX,
Ann Arbor MI, Clearwater FL, West Jordan UT, Manchester NH.

**Manual corrections** (applied before the automated screen; these two cases require
year or status overrides that the string check alone cannot handle):

| City | ST | Original year | Action | Reason |
|---|---|---|---|---|
| Bridgeport | CT | 2012 | Set `year_corrected = 2016`; override `status_corrected = adopted_year_verified` | "BConnected" (2012) was a citizen-reporting app, not a 311 system; "Bridgeport 311" launched 2016 |
| Raleigh | NC | 2011 | Retain year; add `definitional_note` flagging uncertainty | SeeClickFix "311" platform later replaced by "Ask Raleigh" (2025); keep but flag |

### 4B — Clean analytical columns

**`adoption_year_clean`** (Int64 or NaN — no strings ever):
Populate only for cities where `adoption_status == 'adopted'` (i.e., passes formal-311
test and year is known). All other rows are NaN.

```python
adopted_with_year = (df['adoption_status'] == 'adopted') & df['year_corrected'].notna()
df['adoption_year_clean'] = pd.NA
df.loc[adopted_with_year, 'adoption_year_clean'] = df.loc[adopted_with_year, 'year_corrected'].astype(int)
df['adoption_year_clean'] = df['adoption_year_clean'].astype('Int64')
```

**`adoption_status`** — three-value categorical for main sample (NaN = review set):

| `status_corrected` | `year_corrected` | `adoption_status` |
|---|---|---|
| `adopted_year_verified` | not null | `adopted` |
| `adopted_year_verified` | null (downgraded by 4A) | `adopted_unknown_year` |
| `adopted_year_unknown` | null | `adopted_unknown_year` |
| `not_adopted` | — | `not_adopted` |
| `conflict_fp` | — | NaN → review set |
| `not_adopted_unclear` | — | NaN → review set |

### 4C — Sample split

**Review set:** cities where `adoption_status` is NaN (i.e., `status_corrected ∈ {conflict_fp, not_adopted_unclear}`). Bridgeport is NOT in the review set — its `status_corrected` was overridden to `adopted_year_verified` in 4A.

### Outputs

| File | Description | N |
|---|---|---|
| `data_clean/city_311_analysis.csv` | Main analytical sample | **301** |
| `data_clean/city_311_review.csv` | Review set: conflict_fp + not_adopted_unclear | **13** |

**Expected `adoption_status` distribution in main sample:**
- `adopted`: 99 (formal 311, year known, range 1996–2025)
- `adopted_unknown_year`: 148
- `not_adopted`: 54

**New columns in `city_311_analysis.csv`** (all original columns from `city_311_final.csv` retained):

| Column | Type | Description |
|---|---|---|
| `adoption_year_clean` | Int64 in notebook; **float64 after CSV round-trip** (values are whole numbers e.g. 2003.0, or NaN) | Formal-311-verified adoption year — **primary year variable for analysis** |
| `adoption_status` | string | `adopted` / `adopted_unknown_year` / `not_adopted` |
| `year_corrected` | float or NaN | Year after manual and automated corrections |
| `status_corrected` | string | Status after manual overrides (differs from `status_final` only for Bridgeport CT) |
| `definitional_note` | string or NaN | Non-null for cities corrected or flagged in 4A |

**Code:** `code/step4_analytical_sample.ipynb`

---

## Execution order

```
[1] Download sub-est2023.csv, 2023_Gaz_place_national.zip, obyrne_2015_dissertation.pdf
    Unzip the Gazetteer zip to produce 2023_Gaz_place_national.txt in data_raw/
      ↓
[2] code/step1_city_list.ipynb
      → data_clean/city_list.csv (314 cities)
      ↓
[3] SEARCH STEP — agent or researcher action, not a notebook
    Read data_clean/city_list.csv row by row.
    For each city, perform a web search and write one txt file to
    data_raw/311_search_log/[city_slug]_[STATE].txt (format specified in Step 2).
    Do not proceed until all 314 cities have a log file (316 files for 2 duplicate slugs).
      ↓
[4] code/step2_extract_obyrne.ipynb      (independent of search; can run alongside step 3)
      → data_clean/obyrne_extracted.csv (242 rows)
      ↓
[5] code/step2_parse_search_log.ipynb    (requires all search logs from step 3)
      → data_clean/search_results.csv (314 rows)
      ↓
[6] code/step3_verify_and_merge.ipynb    (requires both obyrne_extracted and search_results)
      → data_clean/search_audit_metrics.csv (23 rows)
      → data_clean/search_audit.csv (314 rows)
      → data_clean/city_311_final.csv (314 rows)
      ↓
[7] code/step4_analytical_sample.ipynb
      → data_clean/city_311_analysis.csv (301 rows)
      → data_clean/city_311_review.csv (13 rows)
```

---

## Reproducibility requirements

| Requirement | Detail |
|---|---|
| `pdftotext` (poppler) | `brew install poppler` (macOS) — required for `step2_extract_obyrne.ipynb` |
| Python packages | `pandas`, `difflib` (stdlib), `re` (stdlib), `subprocess` (stdlib) |
| `sub-est2023.csv` encoding | Must read with `encoding='latin-1'`; UTF-8 will raise UnicodeDecodeError |
| Gazetteer column names | Run `.str.strip()` on all column names immediately after reading |
| Gazetteer GEOID | Run `.str.strip().str.zfill(7)` on the GEOID column |
| Census URLs | All public; no authentication required |
| Gazetteer zip | Unzip manually before running; notebook reads the `.txt` directly |
| Search logs | All 316 files in `data_raw/311_search_log/` must be present before running `step2_parse_search_log.ipynb` |
| Row counts | Verify: city_list=314, obyrne_extracted=242, search_results=314, city_311_final=314, city_311_analysis=301, city_311_review=13 |

---

## Non-reproducible step note

Step 2 (web search) is intentionally non-reproducible. If re-run by a different agent:
- Different sources may be found for the same city
- Some cities classified as `unknown` may be resolved, and vice versa
- The 316 existing log files are treated as raw data and must not be regenerated

All downstream steps (parse, verify, analytical sample) are fully deterministic given
the existing log files.
