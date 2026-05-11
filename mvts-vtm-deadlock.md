---
description: >
  MVTS VTM Cyclic Deadlock Debugging — diagnoses and traces VTM bot aisle deadlocks.
  TRIGGER when: no VTM bots are getting assigned tasks, VTM deadlock suspected, bots stuck in aisles,
  mvts_relay_vtm_bot_assignment_details empty for 3+ hours.
  SKIP: HTM relay assignment issues — use mvts-relay-debug instead.
---

# MVTS VTM Cyclic Deadlock Debugging

**Symptom**: No VTM bots are getting assigned tasks.

---

## Prerequisites

Resolve the namespace, ensure pod is running, and have `/tmp/ps.json` saved. See `mvts-debug` for access patterns.

Find the `InstallationId` for InfluxDB queries:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep -i "installation\|influx" /app/data/config/local.application.properties'
```

---

## Step 1 — Check VTM bot state in the problem statement

First check what VTM version is present:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq "[.ranger_list[].version] | unique" /tmp/ps.json'
```

Extract current vs assigned aisle for all VTM bots:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".ranger_list[] | select(.version == \"VTM_QT_C56_GT_2D\") | {id, current_aisle: .current_aisle_info, assigned_aisle: .ranger_schedule[0].aisle_info, status, available_capacity}" /tmp/ps.json'
```

---

## Step 2 — Identify deadlock cycles

Build a mapping: `bot → current_aisle → assigned_aisle`

Look for:
1. **Direct cycle**: Bot A at aisle X wants aisle Y, Bot B at aisle Y wants aisle X → neither can move
2. **Cascading blocks**: Other bots waiting to enter aisles involved in the cycle

---

## Step 3 — Confirm via InfluxDB

**Check if any VTM assignments are happening at all:**

If `mvts_relay_vtm_bot_assignment_details` has no rows for 3+ hours, no assignments have been made → strong deadlock signal.

**Visualize the deadlock graph:**
```
mvts_relay_vtm_transitive_movements_nodes   -- aisles as nodes with color/state
mvts_relay_vtm_transitive_movements_edges   -- bot movements as directed edges
```

Node color legend:
| Color | Hex | Meaning |
|-------|-----|---------|
| Purple | `#A78BFA` (`highlighted=true`) | Deadlocked aisle |
| Amber | `#FBBF24` | Occupied (not deadlocked) |
| Green | `#22C55E` | Free-moving |
| Gray | `#6B7280` | Empty aisle |

**Key signal**: Same purple nodes unchanged across multiple consecutive `request_id` values (~13s apart) = persistent deadlock.

---

## Step 4 — Trace the full cascade

For each blocked bot, check what aisle it's waiting for and whether that aisle's occupant is also blocked. Build the full dependency chain to find the root cycle.

---

## Step 5 — Traceback via InfluxDB aisle switches

```influxql
-- All aisle switches for deadlocked bots
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 24h
AND "InstallationId" = '<installation_id>'
AND ("bot_id_tag" = '<bot_a>' OR "bot_id_tag" = '<bot_b>')
ORDER BY time ASC

-- All switches in/out of a specific aisle
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 24h
AND "InstallationId" = '<installation_id>'
AND (to_aisle = <N> OR from_aisle = <N>)
ORDER BY time ASC

-- Bots coming from dummy aisles (negative from_aisle = bot in transit/idle)
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 48h
AND "InstallationId" = '<installation_id>'
AND from_aisle < 0
ORDER BY time ASC
```

**Confirm deadlock** — both bots repeat the exact same `task_chain` every ~12s:
```influxql
SELECT * FROM mvts_relay_vtm_bot_assignment_details
WHERE time > now() - 12h
AND "InstallationId" = '<installation_id>'
AND (bot_id = <A> OR bot_id = <B>)
ORDER BY time ASC
```

---

## Root cause pattern

1. Bot A finishes tasks in aisle X → MVTS sends it to aisle Y
2. Bot B is in a dummy aisle (in transit); aisle X gets new tasks → MVTS assigns Bot B to aisle X
3. Bot B physically passes through aisle Y on its way to X
4. Now: Bot A at X wants Y (Bot B is there), Bot B at Y wants X (Bot A is there) → deadlock
5. MVTS made two independently valid decisions without detecting the physical path conflict

---

## Resolution

**MVTS cannot resolve cyclic deadlocks** — this requires GMC intervention. Escalate to the GMC team with:
- The deadlocked bot pair and their aisles
- The aisle switch history showing how it formed
- The `request_id` when the deadlock first appeared

---

## Config flags (for investigation)

| Flag | Default | Effect |
|------|---------|--------|
| `ENABLE_VTM_TASK_REASSIGNMENT` | `true` | `false` = every cycle assigns from scratch, no deadlock detection runs |
| `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM` | `false` | `true` = cascade deassignment on TAM failure (not yet wired in production as of f4 branch) |

Check/set via config API:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all | python3 -m json.tool | grep -i "vtm\|tam"'
```

---

## Notes

- `current_aisle_info` can be `null` if bot is in transit between aisles
- Multiple bots assigned to the same aisle = contention, not necessarily deadlock
- The core deadlock is always a cycle — fixing it unblocks the entire cascade
