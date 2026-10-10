# Methodology

Why the pipeline works the way it does, explained for someone new to the problem. Implementation details are in [pipeline.md](pipeline.md).

## Contents

1. [The problem](#1-the-problem)
2. [Core idea: spread as confidence](#2-core-idea-spread-as-confidence)
3. [Worked example](#3-worked-example)
4. [Cleaning rules and their reasons](#4-cleaning-rules-and-their-reasons)
5. [DBSCAN in five minutes](#5-dbscan-in-five-minutes)
6. [Tiers and thresholds](#6-tiers-and-thresholds)
7. [Validation approach](#7-validation-approach)
8. [Limits of the method](#8-limits-of-the-method)

---

## 1. The problem

Each subscriber has been visited by field agents many times, and each visit recorded a GPS position. Those positions are noisy: devices drift, agents read from the street instead of the meter, operators mistype coordinates, and some points are simply wrong. The meter's own coordinate is available only for some subscribers and is also error-prone.

The task is to turn these scattered readings into **one location per subscriber** and to say **how much that location can be trusted**.

## 2. Core idea: spread as confidence

If a subscriber's readings agree with each other, their mean is a good location estimate. If they disagree, it is not.

The pipeline measures disagreement as one number, the **combined standard deviation**:

```
MD_agent_std = round( sqrt( σx² + σy² ) )          # metres
```

* Small value → readings cluster tightly → high confidence.
* Large value → readings are spread out → low confidence.

Confidence is then turned into three **tiers** (parcel, block, regional) by fixed thresholds ([§6](#6-tiers-and-thresholds)).

A second idea rescues many subscribers: a large spread is often caused by **a few bad readings** among many good ones. Removing the strays with density-based clustering (DBSCAN) and recomputing the statistics can move a subscriber from a low tier to a high one.

## 3. Worked example

All numbers below are **synthetic**. Positions are offsets in metres from an arbitrary origin; they are not real data. Rule set: at least 6 readings, parcel threshold 15 m, DBSCAN `eps = 15`, `min_samples = 5`.

| Subscriber | Readings | σx / σy | `MD_agent_std` | What happens | Result |
|---|---|---|---|---|---|
| **A** | 8, all within about ±5 m | 3 / 2 | **4** | ≥ 6 readings and ≤ 15 m → *best accuracy* | **Parcel** |
| **B** | 8: six within about ±4 m, two roughly 100 m away | 47 / 30 | **56** | > 15 m → DBSCAN labels the six as cluster 0, the two far points as noise. Keep the six, recompute: σ 3 / 2 | **Parcel** (MD 4) |
| **C** | 8, spread over about 100 m, no dense group | 38 / 39 | **54** | > 15 m → DBSCAN labels **all** points as noise → keep original statistics | **Regional** (MD 54) |
| **E** | 8, spread over about ±30 m, no dense group | 19 / 18 | **26** | > 15 m → all noise → keep original statistics | **Block** (MD 26) |
| **D** | 4 readings | n/a | n/a | Fewer than 6 readings → no estimate attempted | *Insufficient data*, or, if D has a meter coordinate, **meter coordinate used as location** |

What this shows:

* **A** needs no help.
* **B** is the case clustering exists for: raw spread (56) would put it in the worst tier, but its true position is well determined by six agreeing readings.
* **C** and **E** have no dense group, so clustering cannot help. Their tier follows from the original spread.
* **D** has too few readings for any statistic to mean anything.

The estimated location is always the mean of the retained readings, e.g. for **B** the mean of the six cluster-0 points, not of all eight.

## 4. Cleaning rules and their reasons

| Rule | Reason |
|---|---|
| Zero → `NaN` | A coordinate of `0` is a placeholder for "missing". Left in, it would drag means and standard deviations toward the origin |
| Swap meter X/Y when `counter_x > 900000` and `counter_y < 1000000` | Operators sometimes entered the northing in the X field. A valid easting is below 1,000,000 and a valid northing is in the millions, so the pattern is unambiguous. Example: `(3900012, 500008)` becomes `(500008, 3900012)` |
| Blank meter coordinates outside the boundary | A meter outside the city is a data error; keeping it would place the subscriber outside the service area |
| Blank agent coordinates outside the boundary | Same reasoning for GPS points |
| Remove readings inside electricity offices | A reading taken at the office describes where the agent was, not where the subscriber is |
| Remove IDs already solved in the previous phase | They already have parcel-level accuracy; this phase works only on what remains |

**Order matters.** Zeros and swaps are fixed *before* the boundary test, otherwise swapped coordinates would be flagged as outside the city and discarded instead of repaired.

## 5. DBSCAN in five minutes

DBSCAN groups points that are densely packed and labels isolated points as noise. It needs two parameters:

| Parameter | Value here | Meaning |
|---|---|---|
| `eps` | 15 | Two points are neighbours if they are within 15 m |
| `min_samples` | 5 | A point is a *core point* if it has at least 5 points (itself included) within `eps` |

Output labels: `0, 1, 2, …` for clusters (numbered in the order they were discovered) and `−1` for noise.

How it is used here:

1. It runs **separately for each subscriber**. Subscribers are independent problems of a few dozen points, and running it on the whole table would merge clusters from different subscribers.
2. If at least one point gets label `0`, the subscriber has a dense group. Only label-`0` points are kept and the statistics are recomputed.
3. If every point is `−1`, there is no dense group. The original statistics are kept.

Because `min_samples` is 5, a cluster has at least 5 points. A subscriber that enters with 6 readings can therefore leave clustering with as few as 5.

## 6. Tiers and thresholds

| Tier | Rule on `MD_agent_std` | Reading |
|---|---|---|
| Parcel-level (قطعه‌ای) | ≤ 15 m | The estimate points to the subscriber's parcel |
| Block-level (بلوکی) | > 15 m and ≤ 45 m | The estimate is right to within a city block |
| Regional (منطقه‌ای) | > 45 m | The estimate only indicates the general area |

The thresholds (6 readings, 15 m, 45 m, `eps = 15`, `min_samples = 5`) are **hard-coded domain choices**, not values derived from the data in this notebook. Confirm their origin with the domain owner before changing them, because they define what each tier means for downstream users.

## 7. Validation approach

The validation compares this phase with the previous one:

1. Take the IDs the previous phase placed at **block** or **regional** level.
2. Intersect them with the IDs this phase places at **parcel** level.
3. The intersection is the number of subscribers **upgraded** by this phase.

In the recorded run, 19,545 of 26,315 previously block/regional IDs (about 74 %) became parcel-level. This measures improvement relative to the earlier phase, **not absolute correctness**: there is no ground-truth coordinate in the data.

## 8. Limits of the method

* **Precision is not accuracy.** A small spread means the readings agree with *each other*. If agents consistently read from the same wrong place (for example the street in front of the building, or a depot), the spread is small and the estimate is still off. The tiers measure consistency.
* **No ground truth.** Nothing in the data confirms the true location, so thresholds cannot be validated against reality from this notebook alone.
* **Cluster 0 is not necessarily the best cluster.** DBSCAN numbers clusters by discovery order. If a subscriber has two dense groups, the pipeline keeps whichever was found first and drops the other.
* **Few-readings subscribers are not estimated.** Fewer than 6 readings are treated as insufficient, even if those readings agree closely.
* **Rounding near thresholds.** Standard deviations are rounded to whole metres before being combined, so a subscriber very close to 15 m or 45 m can land on either side of the line.
* **The swap heuristic is range-based.** It assumes UTM zone 39N values. Data in another zone or another coordinate system needs a different rule.
