# Relay Assignment Debug

Diagnoses why HTM assignments are not happening for a PPS. Walks through the full chain: HTM task pool → order promotion → tote locations → VTM assignment state.

---

## When to use

- `promoted_on_relay_count = 0` persisting across multiple PS cycles
- HTM bots idle but no relay tasks being assigned
- Totes not reaching relay despite orders being present

---

## Step 1 — Check the HTM task pool

Query `mvts_relay_incoming_tasks_per_pps`. The key column is `promoted_on_relay_count`.

```
query_metrics({
  environment: "<env>",
  measurement: "mvts_relay_incoming_tasks_per_pps",
  timeRange: "30m",
  limit: 50
})
```

**What to look for (per PPS row):**

| Field | Meaning |
|-------|---------|
| `promoted_on_relay_count` | Tasks available for HTM assignment. **If 0, no assignments can happen.** |
| `at_relay` | Totes currently sitting at a relay point, eligible for promotion |
| `incoming_tasks` | Total orders pending for this PPS |
| `totes_invalid_aisle` | Totes whose aisle MVTS can't resolve — VTM cannot be assigned |
| `totes_unavailable` | Totes blocked: have an active VTM task, or otherwise ineligible |
| `on_vtm_bot` | Totes currently being carried by a VTM |
| `stored` | Totes sitting in store waiting for VTM pick |

If `promoted_on_relay_count = 0`, proceed to Step 2 to find why.

---

## Step 2 — Check order promotion state

Query `mvts_relay_order_details` to see which orders are not promoted and why.

```
query_metrics({
  environment: "<env>",
  measurement: "mvts_relay_order_details",
  timeRange: "10m",
  limit: 30
})
```

**Key fields:**

| Field | Meaning |
|-------|---------|
| `is_promoted` | `true` = order has an HTM task; `false` = blocked |
| `is_in_promotion_cache` | `true` = MVTS is aware of order but waiting on tote(s) |
| `msu_ids` | `toteid:availability:location:botid` — one entry per tote in the order |
| `locations` | Where the order sits: `task_list` (pending), `pps` (at station), `ranger` (on HTM) |

**Decode `msu_ids`:**
- `HA20004910:true:STORABLE:null` → available, in store, no bot assigned
- `HA20004910:false:STORABLE:null` → **unavailable**, in store, no bot assigned
- `HA20004910:true:RELAY:null` → at relay, eligible for HTM pick
- `HA20004910:true:RANGER:129` → on HTM bot 129, en route to PPS

**Decision:**
- All totes show `location=RELAY` and `availability=true` → **Pick (MVTS) is at fault** — order should be promoting but isn't (escalate to MVTS team)
- Any tote NOT at relay → VTM hasn't brought it yet → proceed to Step 3

---

## Step 3 — Check VTM assignment state

Query `mvts_relay_vtm_bot_assignment_details` to see what VTM bots are doing.

```
query_metrics({
  environment: "<env>",
  measurement: "mvts_relay_vtm_bot_assignment_details",
  timeRange: "30m",
  limit: 50
})
```

**Task chain format:** `↑TOTE_ID(ACTION, LOCATION, aisle) -> ↓TOTE_ID(ACTION, LOCATION, aisle)`
- `↑` = pick up tote
- `↓` = put down tote
- `STORE→RELAY` direction: VTM bringing tote to relay (correct for PICK orders)
- `RELAY→STORE` direction: VTM returning tote to store (post-pick / put-away)
- `*` prefix on task: task sent to GMC and in-progress

**Diagnose each blocked tote:**

### Tote has `invalid_aisle`
MVTS cannot find a valid aisle for the tote → no VTM assignment possible.
- Usually transient: GMC/butler-core may not have updated the tote's bin location yet
- Check PS for the tote's current aisle: `jq ".task_list[] | select(.task_key =~ \"<TOTE_ID>\")" /tmp/ps.json`
- If aisle is missing/null in PS, this is a GMC data staleness issue — wait or escalate to GMC team

### Tote has no VTM assignment (and is in STORABLE)
No VTM has been tasked to bring this tote to relay.
- Check whether a VTM is assigned in `mvts_relay_vtm_bot_assignment_details` with a `STORE→RELAY` chain for this tote
- If no assignment exists: MVTS is not planning a task — check `mvts_relay_incoming_tasks_per_pps` for `not_received` or `inconsistent_data` flags
- If assignment exists but no `*` prefix after multiple cycles → VTM task stuck (see below)

### VTM task repeating without `*` prefix (STUCK)
Same task chain logs every ~11 seconds (one per PS cycle) with no `*` prefix = MVTS plans it but GMC hasn't executed it.

```
# Find first occurrence to determine how long it's been stuck
query_metrics({
  environment: "<env>",
  influxqlQuery: "SELECT * FROM \"mvts_relay_vtm_bot_assignment_details\"
                  WHERE time > now() - 2h
                  AND task_chain =~ /<TOTE_ID>/
                  ORDER BY time ASC LIMIT 5"
})
```

If stuck for >5 minutes, this is a **GMC-side execution issue** — MVTS is planning correctly but GMC is not accepting/running the task. Escalate to GMC team with: bot_id, tote_id, task chain, and first-seen timestamp.

### VTM task has `*` prefix
Task is in-progress. Wait 1–2 PS cycles for it to complete. The tote should transition from `unavailable` → `on_vtm_bot` → `at_relay`.

---

## Step 4 — Check the problem statement (if pod is accessible)

If VTM assignment exists but the aisle seems wrong:

```bash
CTX="gke_<project>_<region>_<cluster>"
NS="<namespace>"
kubectl exec mvts-0 -n $NS --context $CTX -- bash -c \
  'grep " Message:" /app/data/logs/scheduler.log | tail -1 | sed "s/.*Message: //" > /tmp/ps.json'

# Check tote's aisle in task_list
kubectl exec mvts-0 -n $NS --context $CTX -- bash -c \
  'jq ".task_list[] | select(.task_key | test(\"<TOTE_ID>\"))" /tmp/ps.json'

# Check relay points for this tote
kubectl exec mvts-0 -n $NS --context $CTX -- bash -c \
  'jq ".relay_point_list[] | select(.reserving_tote_id == \"<TOTE_ID>\")" /tmp/ps.json'
```

---

## Decision tree summary

```
promoted_on_relay_count = 0?
├── incoming_tasks = 0 → No orders from Pick. Check Pick system.
└── incoming_tasks > 0
    ├── All totes at_relay=1, available=true → MVTS promotion bug. Escalate.
    └── Totes NOT at relay
        ├── Check order_details for these totes first:
        │   ├── is_promoted=true + location=CONVEYOR → Tote mid-pick-cycle on conveyor.
        │   │   on_unknown_bot flag is also expected. Normal — wait for conveyor to finish.
        │   └── is_promoted=false → Tote genuinely blocked, continue below.
        ├── invalid_aisle (not CONVEYOR) → GMC aisle data stale. Wait or escalate GMC.
        ├── No VTM assignment → MVTS not planning. Check not_received/inconsistent flags.
        └── VTM assigned
            ├── task repeating without * for >5 min → GMC not executing. Escalate GMC.
            └── task has * prefix → In progress. Wait.
```

---

## Common patterns seen

| Pattern | Symptom | Resolution |
|---------|---------|------------|
| Transient invalid_aisle | Tote aisle unknown for a few cycles, then clears | Self-resolves when GMC syncs location |
| VTM relay→store stuck 10–15 min | Same relay→store chain repeats, no `*`, tote stays `unavailable` | GMC task execution delay; self-resolves or requires GMC team intervention |
| No VTM assignments at all | `mvts_relay_vtm_bot_assignment_details` empty for >5 min while tasks pending | Check if relay is enabled: `ENABLE_RELAY=true` in config |
| Tote on CONVEYOR (false alarm) | Tote appears in `invalid_aisle` / `unavailable` / `on_unknown_bot` but order_details shows `is_promoted=true` with `location=CONVEYOR` | Tote is mid-pick-cycle on the conveyor belt — not a real blockage. Normal queuing for subsequent orders needing the same tote. No action needed; wait for conveyor cycle to complete. |
