# Known Issues and Roadmap

Caveats a developer should know before extending the code, and a suggested path for improving it. The repository restructure cleared saved outputs and embedded subscriber IDs, added the run-directory setup, replaced the shell-specific working-directory command, and added an MIT license. The processing caveats below still describe the recorded algorithm.

## Contents

1. [Code and method caveats](#1-code-and-method-caveats)
2. [Repository and process issues](#2-repository-and-process-issues)
3. [Performance notes](#3-performance-notes)
4. [Roadmap](#4-roadmap)

---

## 1. Code and method caveats

1. **Misleading comment:** the DBSCAN cell says `epsilon = 15  # 100 meters`; the actual radius is **15 m**.
2. **Cluster selection:** only `Cluster == 0` is kept. DBSCAN numbers clusters in discovery order, not by size, so if an ID has several clusters the retained one is not necessarily the largest.
3. **Minimum points after clustering:** `min_samples=5` while the entry rule is `agent_count ≥ 6`. After cleaning, a refined ID may have only 5 points, so the "≥ 6" rule is not enforced on DBSCAN-refined results.
4. **Different joins in the two branches:** Branch A merges `cleaned` and `old` statistics with `how='outer'`; Branch B uses the default inner join. Verify that this is intended.
5. **Boundary test counts `NaN` rows:** for agents, `Point(nan, nan).within(...)` is `False`, so `R_outside_agent` (18,663) probably includes readings that were already missing. The meter step guards against this (`originally_non_nan`); the agent step does not.
6. **Office filter is row-level**, not ID-level, despite the heading "remove entire record". An ID can lose some readings but stay in the pipeline.
7. **Off-by-one in the report:** `R_access_rows = shape[0] - 1` subtracts one row although `read_csv` already consumed the header, so the printed count is one lower than the true row count.
8. **Single meter coordinate assumed per ID.** If an ID has both `NaN` and valid meter rows, the join in Stage II can produce duplicate index entries and the last one wins in the dictionary conversion.
9. **Rounding before combining:** `σx` and `σy` are rounded to whole metres before `MD_agent_std` is computed, which introduces up to ≈1 m of error near the 15 m / 45 m thresholds.
10. **Typos in names:** `Clusterd_…`, `…_cooridate`, `filtered_data_agent_count_le6` (rule is `< 6`), `print(R_count_true, R_count_true)` (prints the true count twice). Rename carefully, because downstream cells use these names.
11. **Environment coupling:** the first code cell now selects a repo-relative run directory and the shell-specific `!pwd` has been replaced by `Path.cwd()`. Input file names are still hard-coded and `%who_ls` still needs an IPython kernel.
12. **Deprecated API:** `pyproj.transform` is deprecated; use `Transformer.from_crs(32639, 4326, always_xy=True)`.
13. **Maps are not persisted:** the Folium map objects are created but the display lines are commented out and nothing is saved as HTML.

---

## 2. Repository and process issues

1. **Historical data exposure remains.** The current notebooks have been cleared of saved outputs and embedded subscriber IDs. Earlier commits still contain subscriber IDs and coordinates. The owner chose to keep the repository public and handle history cleanup separately. Deleting data in a later commit does not remove it from history; see the [runbook](runbook.md#7-committing-safely).
2. **Notebook output hygiene.** Saved outputs have been stripped, substantially reducing notebook size. `.gitattributes` marks notebooks for the `nbstripout` filter; install it once in every clone as described in the runbook.
3. **Two notebooks that diverge.** `prototype_preprocessing.ipynb` (formerly `automated.ipynb`) is an earlier attempt. Observed differences from the full pipeline:
   * it ends after per-subscriber aggregation and writes `reorg.json`; there is no DBSCAN, no tiering and no output files;
   * previous-phase IDs are read from `dbscan.csv`, not from the DBF;
   * the boundary is applied with `gpd.sjoin(..., op="within")` against a city shapefile, while the full pipeline uses `within(unary_union)`;
   * the data is split by meter-coordinate availability before the swap fix, while the full pipeline fixes swaps on the whole table first;
   * statistics are not rounded;
   * it defines small helper functions (for example `drop_zeros`), the only place where logic is function-based.
   Keep it as an archive and do not treat it as a second source of truth.
4. **No automated tests.** Behaviour is only checked by eye (printed counts and maps). The [reconciliation checks](data-dictionary.md#6-reconciliation-checks) are a cheap first step.
5. **Hard-coded names and paths.** Input names, three DBF names and all intermediate file names are literals scattered across cells, and the notebook relies on its working directory.
6. **License resolved.** The repository now includes an [MIT License](../LICENSE), and notebook headers link to it.
7. **No recorded environment.** Package versions were not saved with the notebook.

---

## 3. Performance notes

* Statistics are computed with `groupby` and then **re-built through JSON and row-wise `apply`** several times. This is slow and memory-heavy on millions of rows.
* DBSCAN runs in a **Python loop over IDs**; the cost scales with the number of scattered subscribers, not with the number of rows.
* The Folium cells add one marker per row in a Python loop. Keep them sampled.
* Many stages read a CSV that the previous stage just wrote. Passing DataFrames between stages removes most of the I/O.

---

## 4. Roadmap

* **Refactor repeated blocks into functions.** The "group → std → reorganise → write CSV" block appears about six times with small variations (`compute_stats(df, id_col, with_counter=False)`).
* **Pass DataFrames instead of round-tripping through JSON/CSV** between stages; keep file output only for deliverables.
* **Vectorise statistics** with `groupby().agg(['mean','std','count'])` and avoid per-ID Python loops except for DBSCAN.
* **Parameterise thresholds** in one config cell or YAML (`MIN_READINGS`, `PARCEL_M`, `BLOCK_M`, `EPS_M`, `MIN_SAMPLES`).
* **Choose the largest cluster** instead of cluster `0`, and enforce the minimum readings after clustering.
* **Add data-quality assertions** (unique ID counts per stage should sum to the total) so the tier files can be reconciled automatically.
* **Use a single CRS helper** and `pyproj.Transformer`, and write QA maps to HTML.
* **Package it as a script / CLI** (`python run.py --district 10`) with logging, keeping the notebook for exploration only.
* **Pin dependencies** and remove unused imports.

### Suggested order

1. **Safety:** current outputs and embedded IDs are cleared, data is git-ignored, and MIT licensing is in place. The repository remains public; historical data cleanup is pending by owner choice.
2. **Reproducibility:** record `requirements.lock.txt` after a successful run and add the reconciliation assertions. The working-directory cell is already included.
3. **Structure:** extract the repeated aggregation block into one function, then move functions into a package under `src/` with a thin notebook on top.
4. **Behaviour fixes:** choose the largest cluster, enforce the minimum readings after clustering, align the two join types, fix the off-by-one.
5. **Automation:** a command-line entry point and a small synthetic dataset for tests.
