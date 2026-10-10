# Data Dictionary

Reference for every table, intermediate file and output of the pipeline. For the logic that produces them see [pipeline.md](pipeline.md).

## Contents

1. [Conventions](#1-conventions)
2. [Inputs](#2-inputs)
3. [Intermediate files](#3-intermediate-files)
4. [Final outputs](#4-final-outputs)
5. [Report variables (`R_*`)](#5-report-variables-r_)
6. [Reconciliation checks](#6-reconciliation-checks)

---

## 1. Conventions

| Topic | Convention |
|---|---|
| Coordinate system | UTM zone 39N, WGS84 (EPSG:32639). `x` = easting, `y` = northing, both in **metres** |
| Missing coordinates | `NaN`. Raw zeros are converted to `NaN` early in the pipeline |
| `bill_ID` / `ID` | 13-digit subscriber identifier. The raw column is `bill_ID`; later files call it `ID` |
| Record | One row = one agent reading for one subscriber |
| Grain | **record** (one reading per row) or **ID** (one row per subscriber). Stated for every file below |
| `MD_agent_std` | `round(sqrt(σx² + σy²))`, where σx and σy are the pandas sample standard deviations (`ddof=1`) of the agent coordinates, each rounded to whole metres first |
| Averages | `agent_x_avg`, `agent_y_avg` are the arithmetic means, rounded to whole metres |
| `counter_*` | Coordinates of the subscriber's **meter** (کنتور) |
| Notation | Folder names `counter_false/` and `counter_true/` hold subscribers **without** / **with** a meter coordinate |

---

## 2. Inputs

### 2.1 Readings table (`d10.csv`)

Exported from the Access database.

| Column | Type | Required | Description |
|---|---|---|---|
| `district` | integer | **yes** | Dropped immediately; the notebook fails with `KeyError` if the column is absent |
| `bill_ID` | integer (13 digits) | yes | Subscriber identifier |
| `agent_x`, `agent_y` | float | yes | Agent position at reading time |
| `counter_x`, `counter_y` | float | yes | Meter position; frequently empty, sometimes swapped or `0` |

### 2.2 Previous-phase results (DBF)

| File | Used for | Required column |
|---|---|---|
| `دقت_قطعه_ای.dbf` (parcel-level) | Pre-filter: these IDs are removed from the raw table | `BILL_IDENT` |
| `دقت_بلوکی.dbf` (block-level) | Validation only (Stage VII) | `BILL_IDENT` |
| `دقت_منطقه_ای.dbf` (regional) | Validation only (Stage VII) | `BILL_IDENT` |

`BILL_IDENT` is cast to `int64` before comparison with `bill_ID`.

### 2.3 Shapefiles

| Variable | Default path | Geometry | Use |
|---|---|---|---|
| `dist_shape` | `T6/T6.shp` | Polygon(s) | Valid area. Points outside are outliers. Must have a CRS or be in EPSG:32639 already |
| `offices` | `offices/offices.shp` | Polygon(s) | Electricity offices. Agent readings inside are discarded |

---

## 3. Intermediate files

All paths are relative to the run directory (`workspace/<district>/`).

### 3.1 Cleaning stage (Stage I), grain: record

All files have the columns `bill_ID, agent_x, agent_y, counter_x, counter_y`.

| File | Contains | Written by |
|---|---|---|
| `filtered_by_dbscan_zer2null.csv` | Raw rows minus previous-phase IDs; zeros → `NaN` | I.3 |
| `filtered_by_dbscan_zer2null_counterFixed.csv` | Same, with swapped meter X/Y corrected | I.4 |
| `Jst_fixed_counter_data.csv` | **Only** the rows whose meter X/Y were swapped (after correction) | I.4 |
| `filtrd_goodCounters.csv` | Meter coordinates outside the boundary set to `NaN` | I.5 |
| `filtrd_badCounters.csv`, `nan_replaced_rows.csv` | Rows whose meter coordinates were newly blanked (identical content) | I.5 |
| `fltrd_good_agentANDcounter.csv` | Agent coordinates outside the boundary set to `NaN` | I.6 |
| `removed_rows.csv` | Rows whose agent point is inside an office polygon | I.7 |
| `fltrd_good_agentANDcounter_office_removed.csv` | Cleaned table: input to aggregation | I.7 |

### 3.2 Aggregation (Stage II), grain: ID

| File | Structure |
|---|---|
| `result_with_combined_std_and_counters_and_averages.json` | `{ID: {agent_x[], agent_y[], counter_x, counter_y, agent_x_std, agent_y_std, MD_agent_std, agent_x_avg, agent_y_avg}}` |
| `preprocessed_reorginized.json` | `{ID: {agent_coordinates[[x,y],…], counter_x, counter_y, agent_x_std, agent_y_std, MD_agent_std, agent_x_avg, agent_y_avg, agent_count, counter_coor}}`. `NaN` pairs removed; `counter_coor` is `true` when both meter coordinates exist |
| `data_counter_coor_true.json` | Subset with `counter_coor = true` |
| `data_counter_coor_false.json` | Subset with `counter_coor = false` |

### 3.3 Branch files (Stage III)

Each branch folder uses the same names, except where noted.

| File (in `counter_false/` or `counter_true/`) | Grain | Columns / structure | Branch |
|---|---|---|---|
| `filtered_data_agent_count_ge6_MD_agent_std_le15.{json,csv}` (B: `filtered_data_with_counter_…`) | ID | CSV: `ID, agent_x_avg, agent_y_avg, MD_agent_std, agent_count` (B adds `counter_x, counter_y`) | A, B |
| `filtered_data_agent_count_le6.{json,csv}` | ID | Same columns as above. Rule is `agent_count < 6` | A only |
| `filtered_data_counter_as_cooridate.{json,csv}` | ID | `ID, counter_x, counter_y, agent_count` | B only |
| `filtered_data_2_DBSCAN_…ge6_MD_agent_std_ge15.json` (B: `…with_counter_2_DBSCAN…`) | ID | Same structure as `preprocessed_reorginized.json` | A, B |
| `Clusterd_filtered_data_2_DBSCAN_….csv` (B: `clustered_filtered_data_with_counter_2_DBSCAN_….csv`) | record | `ID, agent_x, agent_y, [counter_x, counter_y,] Cluster` | A, B |
| `has_cluster.csv` | record | IDs with at least one label `0`; all their readings including noise (`−1`) | A, B |
| `no_cluster.csv` | record | IDs whose readings are all `−1` | A, B |
| `cleaned_has_cluster.csv` | record | `has_cluster` rows with `Cluster == 0` only | A, B |
| `has_cluster_result_with_combined_std_and_averages.{json,csv}` | ID | **Before** clustering. CSV columns renamed `old_agent_x_avg, old_agent_y_avg, old_MD_agent_std, old_agent_count` | A, B |
| `cleaned_has_cluster_result_with_combined_std_and_averages.{json,csv}` | ID | **After** clustering: `ID, agent_x_avg, agent_y_avg, MD_agent_std, agent_count` | A, B |
| `no_cluster_result_with_combined_std_and_averages.{json,csv}` | ID | Statistics on the original points. B adds `counter_x, counter_y` | A, B |
| `DBSCAN_results.csv` | ID | After-columns merged with `old_*` columns | A, B |
| `temp.json` | ID | Scratch file, overwritten several times. Do not rely on it | A, B |

**`Cluster` label:** `0` = first dense cluster found, `−1` = noise. Labels greater than 0 may exist but are discarded by the pipeline.

---

## 4. Final outputs

All in `output/`, grain: **ID** unless noted.

| File | English meaning | Columns | Rule |
|---|---|---|---|
| `دقت_قطعه_ای.csv` | Parcel-level | `ID, agent_x_avg, agent_y_avg, MD_agent_std, agent_count` (+ `counter_x, counter_y` where available) | Best-accuracy IDs + DBSCAN/no-DBSCAN rows with `MD_agent_std ≤ 15` |
| `دقت_بلوکی.csv` | Block-level | same | `15 < MD_agent_std ≤ 45` |
| `دقت_منطقه_ای.csv` | Regional | same | `MD_agent_std > 45` |
| `مشترکین_با_داده_ناکافی.csv` | Insufficient data | same | Branch A, `agent_count < 6` |
| `مختصات_کنتور_به_عنوان_مکان_مشترکین.csv` | Meter coordinate used as location | `ID, counter_x, counter_y, agent_count` | Branch B, `agent_count < 6` |
| `برداشت_های_روی_اداره_برق.csv` | Readings located on electricity offices | raw record columns | grain: **record** |
| `مشترکین_با_دقت_بهبود_یافته_در_فاز_جدید.csv` | IDs upgraded vs previous phase | `Identical Numbers` (the IDs) | Present in previous block/regional list **and** in the new parcel-level list |
| `*.png` | Report charts | n/a | Donut/pie charts |

---

## 5. Report variables (`R_*`)

Counters collected during the run and used by the charts. `%who_ls` lists them.

| Variable | Meaning |
|---|---|
| `R_access_rows`, `R_access_ID_unique` | Raw table: records, unique IDs (see known issue on the off-by-one in `R_access_rows`) |
| `R_dbscan_rows_uniqueID` | IDs in the previous-phase parcel-level DBF |
| `R_flt_by_dbscan_rows`, `R_flt_by_dbscan_ID_unique` | Records / IDs left after the pre-filter |
| `R_counters`, `R_counters_ID_unique` | Records / IDs with a meter coordinate |
| `R_counters_inverted`, `R_counters_inverted_ID_unique` | Records / IDs whose meter X/Y were swapped |
| `R_counters_outlayer`, `R_counters_outlayer_ID_unique`, `R_outside_counters` | Meter coordinates outside the boundary |
| `R_outside_agent` | Agent readings set to `NaN` by the boundary test |
| `R_count_true`, `R_count_false` | IDs with / without a meter coordinate |
| `R_2_counter_as_coor` | IDs whose meter coordinate is used as location |
| `R_1_best_accuracy_ID_unique` | Best-accuracy IDs before DBSCAN |
| `R_old_not_good_IDs`, `R_new_good_IDs` | Validation: previous block + regional IDs; new parcel-level IDs |

---

## 6. Reconciliation checks

The notebook does not verify these. They are expected to hold and are worth adding as assertions or a final cell.

1. **Every aggregated ID lands in exactly one tier or bucket.**
   `|parcel| + |block| + |regional| + |insufficient data| + |meter as location|` should equal the number of IDs in `preprocessed_reorginized.json`.
2. **Tiers are disjoint.** The three tier files share no `ID`.
3. **Aggregation covers the cleaned table.** The number of IDs in `preprocessed_reorginized.json` equals the unique IDs in `fltrd_good_agentANDcounter_office_removed.csv`.
4. **Branch split is complete.** `R_count_true + R_count_false` equals the number of IDs in `preprocessed_reorginized.json`.
5. **Office readings are row-level.** An ID can appear in `برداشت_های_روی_اداره_برق.csv` *and* in a tier file when only some of its readings were inside an office. This is not an error.
6. **Upgrade list is a subset.** Every ID in the upgraded list is in the parcel-level file.
