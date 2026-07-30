---
description: >
  MVTS Metric Correlation Analysis — build a correlation matrix across N InfluxDB metrics
  (e.g. R2R time, nav factor, breach %, queue depth, idle time) at any time granularity.
  TRIGGER when: user asks "is there a relationship/correlation between X and Y", "correlation
  matrix", "what's causing the disconnect between X and Y", "bifurcate this in Nhr intervals",
  "does this hold at finer granularity", or wants to compare planned vs actual metrics.
  SKIP: single-metric queries with no comparison — use /mvts/dev/influxdb instead.
allowed-tools: ["mcp__gor-global-mcp__query_metrics", "mcp__gor-global-mcp__describe_measurement",
  "mcp__gor-global-mcp__search_influxql_catalog", "mcp__gor-global-mcp__list_environments", "Bash"]
---

# MVTS Metric Correlation Analysis

> **Engineer-only tool**: This is a technical reference for MVTS engineers.
> If you are an operations user, use `/mvts/index` instead and describe the problem in plain English.

Computes a correlation matrix across any set of InfluxDB-backed metrics (mean-of-field metrics,
or computed ratios like breach %), aligned on shared timestamps, at whatever time granularity
fits the window being analyzed.

---

## Step 1 — Define the metrics

For each metric, resolve:
- **measurement** and **bucket/database** (`GreyOrange` is default; some tables like
  `operator_working_time_summary_airflow` live in `Alteryx` — check with
  `describe_measurement` or `search_influxql_catalog` first)
- **field** to aggregate (usually `MEAN(field)`)
- **outlier filter**, if the field can have extreme junk values (e.g. `< 300` seconds to
  drop >5min stalls before averaging) — decide this per-field, not globally
- **tag casing** — this varies per measurement and will silently return 0 rows if wrong.
  Check with `describe_measurement` before assuming: some use `"InstallationId"` (tag),
  others use `"installation_id"` (tag OR field, lowercase). Confirm the actual value too —
  don't assume `InstallationId` matches the environment alias; query
  `SHOW TAG VALUES FROM "<measurement>" WITH KEY = "InstallationId"` or
  `SELECT DISTINCT("installation_id") FROM ... WHERE time > now() - 7d` if unsure.

**Simple mean metric:**
```influxql
SELECT MEAN("field_name") FROM "measurement"
WHERE time >= '<start>' AND time <= '<end>'
  AND "InstallationId" = '<id>' AND "field_name" < <outlier_threshold>
GROUP BY time(<interval>) fill(none)
```

**Ratio metric** (e.g. "breach %" = % of rows where a computed value exceeds a threshold).
InfluxQL can't divide two aggregates across different WHERE clauses in one query — run two
COUNT queries over the same subquery and divide in Python:
```influxql
-- total
SELECT COUNT(x) FROM (SELECT (a - b) AS x FROM "measurement" WHERE time >= '<start>' AND time <= '<end>' AND "tag" = '<id>')
GROUP BY time(<interval>) fill(0)

-- subset (numerator)
SELECT COUNT(x) FROM (SELECT (a - b) AS x FROM "measurement" WHERE time >= '<start>' AND time <= '<end>' AND "tag" = '<id>')
WHERE x > <threshold> OR x < -<threshold>
GROUP BY time(<interval>) fill(0)
```
Then `ratio_pct[t] = subset[t] / total[t] * 100` for buckets where `total[t] > 0`.

---

## Step 2 — Pick granularity and time range

Rule of thumb: aim for **at least 12-20 data points** so the correlation isn't dominated by
noise. Match the interval to the window:

| Window | Suggested interval |
|--------|---------------------|
| Full month | `1d` |
| Single week | `6h` |
| Single day | `1h` |
| A few hours | `15m` |

If a relationship matters, **check it at more than one granularity** before trusting it.
A correlation that only shows up in daily aggregates but disappears at 1h can mean the
relationship is a slow, accumulated effect (real, but not a moment-to-moment trigger) — or
it can mean the daily number was a coincidental trend, not causal. Compare both interpretations
against what you already know about the system rather than assuming either by default.

Some measurements return extra empty (`fill(0)`) buckets past your requested range (an
InfluxDB/tool quirk, not a real symptom) — harmless, just ignore/drop count=0 buckets outside
the range you asked for.

---

## Step 3 — Align series on shared timestamps

Different measurements can have different available buckets (missing data, different
retention, different write cadence). **Inner-join on timestamp — only keep buckets present
in every series** — before computing correlation. Do this in Python:

```python
keys = set(series_a) & set(series_b) & set(series_c)  # ... & all series
keys = sorted(keys)
aligned = {name: [s[k] for k in keys] for name, s in all_series.items()}
```

Don't try to eyeball-align by counting rows — always intersect explicit timestamp keys.

---

## Step 4 — Compute the correlation matrix

```python
import numpy as np

names = list(aligned.keys())
for n1 in names:
    for n2 in names:
        r = np.corrcoef(aligned[n1], aligned[n2])[0, 1]
```

**Pearson (`np.corrcoef`) is the default** — measures linear correlation, sensitive to outliers.

**Use Spearman instead/also when a series has known extreme outliers** (e.g. a "planned"
estimate that occasionally spikes 10-50x baseline due to a rare edge case) — Spearman only
cares about rank order, so it won't get dominated by a couple of extreme points:
```python
from scipy.stats import spearmanr
rho, _ = spearmanr(aligned[n1], aligned[n2])
```
If Pearson and Spearman disagree a lot, the Pearson number is probably an outlier artifact —
say so rather than reporting just one number.

Present as a markdown table, most-correlated pairs called out in prose. Note the `n` (number
of aligned buckets) alongside the matrix — a correlation from 8 points means something very
different from one computed over 400.

---

## Common pitfalls (found the hard way)

| Symptom | Likely cause |
|---------|-------------|
| Two metrics that *should* relate show ~0 correlation | They may be unpaired — averaging two independently-computed metrics into the same time bucket destroys per-event pairing. If you have a shared key (`task_key`, `request_id`, `pps_id`), join on that instead of blind time-bucket means. |
| A metric has wild swings (10x+) another doesn't | Likely a sparse/event-triggered metric (fires rarely, e.g. only on new-assignment events) vs a dense per-task average — the sparse one will have much higher variance and can dominate/mislead an aggregate correlation. Check sample count (`n`) per bucket for each metric, not just the mean. |
| Correlation strong at 1d, weak at 1h | The relationship may be real but *cumulative* (builds up over hours), not a moment-to-moment trigger — don't conclude "no relationship," conclude "not an instant one." |
| A "planned/predicted" metric doesn't correlate with its "actual" counterpart | Consider whether the planned metric is a **decision trigger** the system acts on (e.g. used internally to decide when to reassign) rather than a passive forecast — if the system corrects on high predicted values, the actual outcome will be suppressed relative to the plan by design, not by measurement error. |
| Query returns 0 rows | Wrong tag-name casing (`InstallationId` vs `installation_id`) or wrong `InstallationId` value for this measurement — verify with `describe_measurement` / `SHOW TAG VALUES` before assuming the metric doesn't exist. |

---

## Worked example (from a real investigation)

Metrics: `planned_r2r` (MEAN `new_idle_time_secs`, <300s filter), `actual_r2r` (MEAN
`time_diff_b_w_pptl_press_to_rack_arrival` from `Alteryx.operator_working_time_summary_airflow`,
<300000ms filter), `nav_factor` (MEAN `congestion` from `mvts_multiplier_msu_to_pps`),
`breach_pct` (ratio from `mvts_pred_vs_real_times`, ±30000ms threshold), `htm_tpq` (MEAN `tpq`
from `mvts_htm_tpq_details`).

Result across granularities (1d → 6h → 3h → 1h): `nav_factor` ↔ `breach_pct` held steady at
~0.72 at every scale — the one robust, causal-adjacent finding. `actual_r2r` ↔ `nav_factor`
dropped from 0.71 (1d) to 0.38 (1h) — real but cumulative, not instant. `planned_r2r` never
correlated with anything real (r ≈ 0.04 to -0.4 at every scale) — the planning side was
decoupled from actual outcomes entirely. That asymmetry (dense/stable actual metric vs.
sparse/spiky planned metric, near-zero Pearson r) was itself the diagnostic signal pointing at
a measurement/pipeline mismatch, not just "no relationship."
