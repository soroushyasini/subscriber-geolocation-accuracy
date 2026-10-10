# Runbook

How to set up, run, check and troubleshoot the pipeline.

## Contents

1. [Set up the environment](#1-set-up-the-environment)
2. [Prepare the inputs](#2-prepare-the-inputs)
3. [Prepare the run directory](#3-prepare-the-run-directory)
4. [Run the notebook](#4-run-the-notebook)
5. [Checkpoints](#5-checkpoints)
6. [After the run](#6-after-the-run)
7. [Committing safely](#7-committing-safely)
8. [Adapting to another district](#8-adapting-to-another-district)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Set up the environment

Python 3.12 was used for the recorded run.

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

`geopandas` reads Shapefiles and DBF files through GDAL. Recent wheels bundle it. If installation fails on your platform, install GeoPandas with conda or your platform's package manager instead.

After a successful run, record the exact versions so the environment can be reproduced:

```bash
pip freeze > requirements.lock.txt
```

## 2. Prepare the inputs

### 2.1 Export the Access database to CSV

The source data lives in a Microsoft Access database and **must be exported to CSV first**. Any of these work:

* **Access:** *External Data → Export → Text File*, comma-delimited, with field names in the first row, UTF-8.
* **Linux/macOS with `mdbtools`:**

  ```bash
  mdb-export database.accdb <TableName> > d10.csv
  ```
* **Python with `pyodbc`/`pandas`** against the Access ODBC driver (Windows).

### 2.2 Required CSV format

| Requirement | Detail |
|---|---|
| Header row | `district, bill_ID, agent_x, agent_y, counter_x, counter_y` (exact names) |
| Delimiter | Comma |
| `district` | Must exist; the notebook drops it |
| Missing values | Empty cells or `0`. Both are treated as missing |
| `bill_ID` | Numeric, 13 digits, no quotes or spaces |
| Coordinates | UTM 39N, metres, decimal point (not comma) |

Quick sanity check before running:

```python
import pandas as pd
df = pd.read_csv("d10.csv", nrows=5)
print(df.dtypes)
print(df.head())
```

### 2.3 Other inputs

| Input | Notes |
|---|---|
| City boundary shapefile | All sidecar files (`.shp .shx .dbf .prj`) must be present. A `.prj` is needed for `to_crs` |
| Electricity-office shapefile | Same |
| Previous-phase DBFs | `دقت_قطعه_ای.dbf` (required), `دقت_بلوکی.dbf` and `دقت_منطقه_ای.dbf` (needed for the validation stage). Each needs a `BILL_IDENT` column |

## 3. Prepare the run directory

The notebook reads and writes everything **relative to its working directory** and creates many files, so run it from a dedicated, git-ignored directory.

```
workspace/d10/
├── d10.csv
├── T6/T6.shp (+ sidecar files)
├── offices/offices.shp (+ sidecar files)
├── دقت_قطعه_ای.dbf  (+ .dbt/.cpg if present)
├── دقت_بلوکی.dbf
└── دقت_منطقه_ای.dbf
```

Both notebooks already include this **first code cell**. It finds the repository from the current directory and selects `workspace/d10/`, including when Jupyter starts from the repo root, `notebooks/`, `notebooks/archive/`, or the run directory. Running it again keeps the same location:

```python
import os
from pathlib import Path

# Locate the repository from the repo, notebooks, archive, or run directory.
START_DIR = Path.cwd().resolve()
REPO_ROOT = next(
    (path for path in (START_DIR, *START_DIR.parents)
     if (path / "notebooks" / "01_full_pipeline.ipynb").is_file()
     and (path / "requirements.txt").is_file()),
    None,
)
if REPO_ROOT is None:
    raise FileNotFoundError("Start Jupyter from inside the repository.")

RUN_DIR = REPO_ROOT / "workspace" / "d10"  # Change per district.
RUN_DIR.mkdir(parents=True, exist_ok=True)
os.chdir(RUN_DIR)
print("Working directory:", Path.cwd())
```

Then confirm the three path variables in Stage 0 (`dist_shape`, `offices`, `access_db`) match your files.

## 4. Run the notebook

* Start Jupyter: `jupyter lab notebooks/01_full_pipeline.ipynb`
* **Run all cells from top to bottom, once.** Later cells read files written by earlier cells, and some scratch files (`temp.json`) are overwritten as the notebook proceeds.
* To re-run, empty the run directory of everything except the inputs listed in [§3](#3-prepare-the-run-directory), then run again from the top. Re-running a single cell in the middle can silently mix old and new files.
* The Folium map cells end with a commented-out display line. Uncomment `m_…` only for a quick visual check on a sampled subset, because drawing hundreds of thousands of markers is slow and memory-hungry.

## 5. Checkpoints

Look at these printed values as the run proceeds. Wildly different numbers usually mean an input problem.

| After stage | Check |
|---|---|
| Load | Printed record count and unique IDs are plausible for the district |
| I.3 pre-filter | Unique IDs left = raw IDs minus previous-phase IDs (not exactly, as the two sources need not share every ID) |
| I.4 swap fix | Count of swapped meter coordinates is reported. A large share is normal for this dataset; a share near 0 % or 100 % deserves a look |
| I.5 / I.6 boundary | Counts of blanked meter / agent coordinates. If almost everything is blanked, the shapefile CRS or the UTM zone is wrong |
| I.7 offices | `removed_rows.csv` is not empty if offices exist in the district |
| II / split | `R_count_true + R_count_false` equals the number of aggregated IDs |
| III DBSCAN | Both `has_cluster.csv` and `no_cluster.csv` exist and are non-empty |
| IV join | The three tier files exist in `output/` |
| VII validation | Intersection count ≤ both list sizes |

## 6. After the run

1. Run the [reconciliation checks](data-dictionary.md#6-reconciliation-checks): the tier and bucket counts must add up.
2. Open `output/` and confirm the seven CSV files and the PNG charts exist.
3. Spot-check a few subscribers from each tier on a map (QGIS or the Folium cells).
4. Save the environment versions (`pip freeze`) and the printed `R_*` values with the results, so the run can be reproduced and compared with future runs.

## 7. Committing safely

The current notebooks have no saved outputs or embedded subscriber IDs. Earlier commits contain subscriber data and remain public by owner choice; history cleanup is pending separately. Before every commit:

1. **Strip notebook outputs.** The simplest way is to automate it:

   ```bash
   python -m pip install nbstripout
   python -m nbstripout --install --attributes .gitattributes  # once per clone
   python -m nbstripout notebooks/01_full_pipeline.ipynb notebooks/archive/prototype_preprocessing.ipynb
   ```

   or manually: *Kernel → Restart & Clear All Outputs*, then save.

   The Git filter clears outputs in committed notebooks; it does not remove subscriber data typed into source cells or comments. Inspect those manually too. The filter configuration is local, so every new clone needs the installation command.

2. Confirm that `workspace/` is git-ignored (`git status` should not list data files).
3. Never commit CSV, JSON, Shapefile, DBF or Access files that contain real data.
4. Put charts that are safe to publish (aggregate pies only) under `docs/img/`.

If real data was already committed, deleting it in a new commit is **not enough**: it stays in the git history. Rewrite the history (for example with `git filter-repo`) or make the repository private, and treat any identifiers already published as exposed.

## 8. Adapting to another district

1. Create `workspace/<district>/` and copy that district's inputs in.
2. Change `RUN_DIR` and the three path variables.
3. Confirm the data is in **EPSG:32639**. If not, change `epsg=32639` and the UTM zone in every `pyproj.Proj(...)` call.
4. Check the swap heuristic (`counter_x > 900000` and `counter_y < 1000000`) against the new coordinate ranges.
5. Make sure the previous-phase DBF belongs to the same district.
6. Leave the thresholds alone unless the domain owner agrees; see [methodology.md](methodology.md#6-tiers-and-thresholds).

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `KeyError: 'district'` | Column missing in the CSV | Export it with the `district` column, or remove the `drop(columns=['district'])` call |
| `FileNotFoundError` for a CSV/JSON | A cell was skipped or run out of order | Empty the run directory and run from the top |
| `FileNotFoundError` for `counter_false/…` | Output folder not created | Run the cell that creates it (it uses `os.makedirs`) |
| Everything is "outside the boundary" | Shapefile in a different CRS, no `.prj`, or data in a different UTM zone | Check `gdf.crs`, add the `.prj`, or fix the zone |
| `to_crs` fails | Shapefile has no CRS | Set it with `gdf.set_crs(epsg=…)` using the true CRS |
| `Number of records` off by one | Known issue: `R_access_rows = shape[0] - 1` | See [known-issues.md](known-issues.md) |
| Run directory cannot be found | Jupyter started outside the repository | Start Jupyter from inside the repository; the first cell uses `Path.cwd()` and creates `workspace/d10/` |
| `%who_ls` errors | Running the code as a plain script, not in Jupyter | Run in Jupyter, or remove the cell |
| `PyTables is not installed` / `SQLalchemy is not installed` | Notices printed by `simpledbf` on import | Harmless. `simpledbf` is not used; remove the import |
| `FutureWarning` / deprecation for `pyproj.transform` | Deprecated API | Harmless for now. See [known-issues.md](known-issues.md) |
| `sjoin(... op=...)` error (prototype notebook) | `op` was renamed to `predicate` in newer GeoPandas | Use `predicate="within"` |
| Persian file names fail to read or write | Locale or file-system encoding | Use UTF-8 and a modern file system, or rename files and update the paths |
| Notebook becomes slow or kills the kernel | Too many Folium markers, or the full table in memory | Sample the points for maps, close unused figures, restart the kernel and run from the top |
| DBSCAN cell takes very long | Per-ID loop in pure Python on tens of thousands of IDs | Expected; see the performance notes in [known-issues.md](known-issues.md) |
