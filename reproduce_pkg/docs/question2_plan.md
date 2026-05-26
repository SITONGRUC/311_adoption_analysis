# Question 2 — Descriptive Analysis

> Describe the geography and timing of 311 adoption. Fewer, more insightful exhibits are better.

---

## Input

**File:** `data_clean/city_311_analysis.csv` (301 rows)

| Column | Use |
|---|---|
| `adoption_year_clean` | numeric year for timing exhibit — **float64 after CSV read** (whole numbers e.g. 2003.0, or NaN); 202 NaN values |
| `adoption_status` | `adopted` / `adopted_unknown_year` / `not_adopted` |
| `pop_2020` | population comparisons |
| `state_abbr` | derive Census region |

**Do not read `city_311_final.csv` for this step.** Step 4 already cleaned the columns.

---

## Derived variables — create immediately after loading

```python
REGION_MAP = {
    **{s: 'Northeast' for s in ['CT','ME','MA','NH','RI','VT','NJ','NY','PA']},
    **{s: 'Midwest'   for s in ['IL','IN','MI','OH','WI','IA','KS','MN','MO','NE','ND','SD']},
    **{s: 'South'     for s in ['DE','FL','GA','MD','NC','SC','VA','WV','DC',
                                 'AL','KY','MS','TN','AR','LA','OK','TX']},
    **{s: 'West'      for s in ['AZ','CO','ID','MT','NV','NM','UT','WY','AK','CA','HI','OR','WA']},
}
df['region'] = df['state_abbr'].map(REGION_MAP)

df['ever_adopt'] = df['adoption_status'].map(
    {'adopted': 1, 'adopted_unknown_year': 1, 'not_adopted': 0}
)  # NaN for review-set cities (none present in this file)
```

---

## Samples

| Sample | Filter | N (approx) | Used for |
|---|---|---|---|
| Timing | `adoption_status == 'adopted'` | 99 | Exhibit 1 |
| Cross-section | `adoption_status` in all three values | 301 | Exhibit 2 + Table 1 |

---

## Exhibit 1 — Timing of 311 diffusion

**File:** `results/figures/fig_adoption_timing.png`

**Chart:** dual-axis — bars for new adopters per year (left axis) + cumulative line (right axis).

**Steps:**
1. Filter to `adoption_status == 'adopted'`, drop rows where `adoption_year_clean` is NaN.
2. Group by `adoption_year_clean`, count cities → `n_new`.
3. Sort by year, compute `cumulative = n_new.cumsum()`.
4. Plot bars (`n_new`) and line (`cumulative`) on shared x-axis, dual y-axes.
5. **Add annotation:** text box with the text computed as:
   ```python
   n_unknown = (df['adoption_status'] == 'adopted_unknown_year').sum()
   # → "Note: {n_unknown} additional cities confirmed 311 but launch year unknown; not shown."
   ```
   Do **not** hardcode 148 — compute from data so the note stays correct if upstream data changes.
6. X-axis: range driven by data (`year_min = timing['year'].min()`, `year_max = timing['year'].max()`). Currently 1996–2025. Label axes. No gridlines on bars.

**Expected takeaway:** staggered diffusion, not a single wave; large unknown-year gap underscores data limitation.

---

## Exhibit 2 — Regional adoption rates

**File:** `results/figures/fig_adoption_region.png`

**Chart:** horizontal stacked bar — one bar per Census region, three segments: `adopted`, `adopted_unknown_year`, `not_adopted`.

**Steps:**
1. Use full 301-city sample.
2. Group by `(region, adoption_status)`, count cities.
3. Pivot to wide: rows = regions, columns = the three statuses.
4. Compute shares within each region (divide each row by row total).
5. Plot horizontal stacked bar (shares, not counts).
   - **Region order** (bottom to top in the horizontal bar): `['West', 'South', 'Midwest', 'Northeast']` — West at bottom, Northeast at top.
   - **Segment order** (left to right within each bar): `['adopted', 'adopted_unknown_year', 'not_adopted']`
6. **Colors** (use exact hex codes — do not substitute with named colors):
   - `adopted`: `'#2c5f8a'` (dark blue)
   - `adopted_unknown_year`: `'#85b0d0'` (medium blue-grey)
   - `not_adopted`: `'#d9d9d9'` (light grey)
7. Label each segment with its share (e.g., "42%") if ≥ 10%; omit smaller segments. Text color: white for `adopted`, `'#333333'` for the other two.
8. **Legend order** matches segment order: Adopted — year known / Adopted — year unknown / Not adopted.

**Expected takeaway:** adoption is geographically uneven; unknown-year cities are concentrated in specific regions, which is itself informative.

---

## Table 1 — Adopters vs. non-adopters

**File:** `results/tables/tab_adopter_summary.tex` (LaTeX tabular) + `results/tables/tab_adopter_summary.csv`

**Rows:** N cities, mean `pop_2020` (thousands), median `pop_2020` (thousands), % in each Census region (4 rows).

**Column order:** `not_adopted` | `adopted` | `adopted_unknown_year` | All

**Column headers:** Not adopted | Adopted (year known) | Adopted (year unknown) | All cities

**Steps:**
1. Compute means/medians/counts per `adoption_status` group plus an "All" column.
2. **Regional shares** = within-group share: for each `adoption_status` column, what percentage of *that group's cities* are in each Census region. E.g. "14.1%" in the adopted / Northeast cell means 14.1% of adopted cities are in the Northeast. Do **not** compute across-group shares.
3. Format as LaTeX `tabular` with `booktabs`. In LaTeX, escape `%` in row labels as `\%`. Save both `.tex` and `.csv`.
4. **Row order:** N, Mean population (000s), Median population (000s), `\midrule`, Northeast (\%), Midwest (\%), South (\%), West (\%)

**Keep it to ≤ 8 rows.** Do not add demographic variables — population and region are sufficient.

---

## Output files

| File | Exhibit |
|---|---|
| `results/figures/fig_adoption_timing.png` | Exhibit 1 |
| `results/figures/fig_adoption_region.png` | Exhibit 2 |
| `results/tables/tab_adopter_summary.tex` | Table 1 |
| `results/tables/tab_adopter_summary.csv` | Table 1 |

---

## Code

Single notebook: `code/question2_descriptive.ipynb`

**Figure style requirements (publication quality):**
- `matplotlib` with `seaborn` style or clean custom style
- DPI ≥ 300 for saved figures
- Font size ≥ 11pt for axis labels, ≥ 9pt for tick labels
- No chart junk (no default grey background, no unnecessary gridlines)
- Figures should be self-contained: title or caption note embedded if needed
