# Insights & Assumptions — Part 1 Data Pipeline

This documents what the pipeline in `Part1DataPipelines.ipynb` actually does,
what was observed from its own saved run output, the assumptions baked into
each step, and where the current implementation diverges from the technical
test brief. It is written from the notebook as checked in — not from an
idealised description of what the pipeline "should" do.

## 1. Source data observations

The `Raw/` folder is read with `Path("Raw").glob("*.csv")` — every CSV file
present is loaded, with no filename pattern or date restriction. As
exercised in the saved notebook run, `Raw/` contained the five files data.gov.sg
publishes for this dataset collection:

| File | Rows |
|---|---:|
| Resale Flat Prices (Based on Approval Date), 1990 - 1999.csv | 287,196 |
| Resale Flat Prices (Based on Approval Date), 2000 - Feb 2012.csv | 369,651 |
| Resale Flat Prices (Based on Registration Date), From Mar 2012 to Dec 2014.csv | 52,203 |
| Resale Flat Prices (Based on Registration Date), From Jan 2015 to Dec 2016.csv | 37,153 |
| Resale flat prices based on registration date from Jan-2017 onwards.csv | 239,467 |
| **Total** | **985,670** |

This total matches the master dataset's row count exactly (see 2. Data Profiling) —
confirming no row is dropped or filtered before the union step. Note the
date coverage runs from **1990 through at least 2017+**, not just the
Jan 2012–Dec 2016 window named in the brief.

**Schema differences reconciled across the five files:**
- `resale_price` is `int` in the 1990–1999 file but `double` everywhere
  else. `infer_combined_schema()` resolves this by ranking types
  (`Integer < Long < Float < Double < Date < Timestamp < String`) and
  picking the widest type seen for each column — so the master schema
  uses `double`.
- `remaining_lease` is present as a **source-provided** column in only 2 of
  the 5 files (Jan 2015–Dec 2016, and Jan 2017 onward). `align_dataframes()`
  fills it with `NULL` for the other three files so `unionByName` succeeds.
  This source column is later **overwritten** by the pipeline's own computed
  `remaining_lease` (4. Remaining lease) — the two are not the same value.

## 2. Data Profiling

- **985,670 rows, 11 columns** after combining and aligning all five files.
- **1,929 exact full-row duplicates** (via `dropDuplicates()`) present at
  the raw, unvalidated stage — before any business-key deduplication (5. 
  Composite key & deduplication) is applied.
- Final unioned schema: `month` (string, `yyyy-MM`), `town`, `flat_type`,
  `block`, `street_name`, `storey_range`, `floor_area_sqm` (double),
  `flat_model`, `lease_commence_date` (int), `resale_price` (double),
  `remaining_lease` (string).

## 3. Data Validation Approach

Town, flat type, flat model and storey range are validated against the
**distinct values seen in the January 2012 records only**, per the brief's
"Jan 2012 dataset as the authoritative set" instruction. `valid_date`
checks the `month` column matches `^\d{4}-(0[1-9]|1[0-2])$` — it confirms
the format is a valid year-month, it does **not** restrict values to
2012–2016.

**Validation is not case-sensitive and is exact-string-match.** 
`upper()` is used on both sides of the Town, flat type, flat model and 
storey range before matching.

## 4. Remaining lease

- **Assumption:** the lease commenced 1 January of the year in
  `lease_commence_date`, and the lease term is 99 years.
- Computed as `add_months(lease_commence_full_date, 99*12)` minus
  `current_date()`, floored to whole months, formatted as
  `"X years Y months"`.
- **Important:** this is computed from the *date the notebook is executed*,
  not the resale transaction date. Re-running the notebook on a different
  day produces different `remaining_lease` values for the exact same input
  row — the output is not reproducible across run dates. It also means the
  computed value has no fixed relationship to the transaction's own month;
  a more typical implementation would calculate remaining lease as-of the
  transaction month, not as-of "today."
- The computed column has the same name as the source-provided
  `remaining_lease` column present in 2 of the 5 raw files (1. Source data 
  observations), and `withColumn` silently overwrites it — the source-provided 
  value for those rows is discarded, not compared or reconciled.

## 5. Composite key & deduplication

- The composite key is **all columns except** `resale_price` and the five
  validation flag columns (`valid_record`, `valid_date`, `valid_town`,
  `valid_flat_type`, `valid_flat_model`, `valid_storey_range`) — matching
  the brief's "composite key excluding resale price" instruction.
- A window, partitioned by the composite key and ordered by
  `resale_price desc`, assigns `row_num`; keeping `row_num == 1` retains the
  **higher-priced** record in each duplicate group, per the brief.
- This runs on the **full, not-yet-validated** dataset. A duplicate group
  can therefore contain a mix of valid and invalid rows; the higher-priced
  row is kept regardless of its own validity, and validity filtering is
  applied as a separate, later step.

## 6. Anomalous price detection

- Grouped by `town`, `flat_type`, `flat_model`. Computes `q1`/`q3` via
  `percentile_approx`, then Tukey's fences: `lower = q1 - 1.5*iqr`,
  `upper = q3 + 1.5*iqr`. Groups smaller than `MIN_GROUP_SIZE = 10` are
  never flagged (`is_anomalous_price = 0` unconditionally), since a
  benchmark from fewer than 10 comparable transactions was judged too
  unreliable to test against.
- **This is reporting metadata, not a routing decision.**
  `is_anomalous_price` does **not** determine whether a row lands in
  `Cleaned` or `Quarantined` — that split is driven entirely by
  `valid_record`, `add_valid_record`, and `row_num`. An anomalous-price
  row that otherwise passes validation still goes to `Cleaned`; an
  anomalous-price row that also fails validation goes to `Quarantined`
  either way, anomaly or not.
- **The flag is preserved for visibility, not used as a filter.**
  `is_anomalous_price` is dropped from `Cleaned`/`Transformed`/`Hashed`
  (consistent with those being the "this row is good" outputs), but it
  **is kept as a column in `Quarantined`** — so if a quarantined row also
  happens to be anomalous, a reviewer can see that at a glance without it
  being what caused the row to be quarantined in the first place.

## 7. Additional validation rules

`add_valid_record` requires all three of:
- `floor_area_sqm > 0`
- `resale_price > 0`
- `lease_commence_date` between 1900 and 2100

`Cleaned` is defined as: `valid_record == 1` **and** `add_valid_record == 1`
**and** `row_num == 1` — i.e. passes every business-rule check, passes the
extra sanity checks, and is the highest-priced survivor of its duplicate
group. There are 941,137 records in `Cleaned` dataset while there are 44,533
records in `Quarantined` dataset

## 8. Resale Identifier & hashing

Built as: `S` + 3-digit block number (non-numeric characters stripped,
zero-padded to 3 digits) + first 2 digits of the **floor of the average
resale price** for that record's `(month, town, flat_type)` group + 2-digit
month + first letter of `town`.

Verified against the notebook's own sample output — e.g. block `104A`,
January 2000, Ang Mo Kio, Executive flats → `S1045501A`. This matches the
brief's worked example exactly (block → 3 digits, avg price → first 2
digits, month → 2 digits, town → 1 letter).

`Hashed_Resale_Identifier = sha2(Resale_Identifier, 256)` — a standard,
irreversible SHA-256 hash, 64 hex characters.

## 9. Other Assumptions

These are worth reading before treating the notebook as a complete,
brief-compliant deliverable:

1. **Anomalous price does not affect routing.**
   `is_anomalous_price` is deliberately **not** one of the OR conditions
   above. This was a genuinely ambiguous call: the brief says to
   "**identify** potentially anomalous resale price," which reads as
   detection/reporting rather than "exclude anomalous rows." Under this
   reading, an anomalous-price row that's otherwise valid still belongs in
   `Cleaned`, not `Quarantined` — routing is driven purely by validity and
   duplicate status. The flag itself is not lost: `Quarantined` retains
   `is_anomalous_price` as a column (unlike `Cleaned`/`Transformed`/
   `Hashed`, which drop it), so a row that's quarantined for a
   validity reason *and* happens to be anomalous is still visibly flagged
   as such. If the intended reading is instead "exclude anomalous rows from
   Cleaned," add `(col('is_anomalous_price') == 1)` as a fourth OR
   condition to `Quarantined` and `.filter(col('is_anomalous_price') == 0)`
   to `Cleaned`.
2. **No Jan 2012–Dec 2016 date filter is applied.** The pipeline processes
   every file found in `Raw/` regardless of date (1. Source data observations); 
   the profiling step reports how many rows fall outside 2012–2016 but nothing 
   removes them. Either restrict which files are placed in `Raw/`, or add an 
   explicit `.filter(col("month").between("2012-01", "2016-12"))` after building
   `master_df`.
3. **`remaining_lease` is not reproducible run-to-run** (4. Remaining Lease) 
   — it depends on the execution date.
4. **Resale Identifier uniqueness is not verified.** The brief asks to hash
   "while preserving uniqueness"; the notebook builds and hashes the
   identifier but never checks for collisions (e.g. two different
   transactions whose block digits, average-price digits, month, and town
   initial all happen to coincide).
