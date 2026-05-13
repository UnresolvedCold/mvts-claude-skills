---
description: >
  MVTS VTM bot stall — diagnoses why VTM storage robots are not moving or getting assignments.
  Covers both cyclic deadlocks (bots blocking each other) and aisle stalls (nothing to assign).
  TRIGGER when: no VTM bots are getting assigned tasks, VTM deadlock suspected, bots stuck in
  aisles, mvts_relay_vtm_bot_assignment_details empty for 3+ hours, all VTM bots log
  "No eligible aisle found", aislesWithTasks empty, PPS throughput dropped, single bot stuck
  at max task capacity for >30 min.
  TRIGGER also when: user says "storage bots are stuck", "robots blocking each other in aisles",
  "no bots are moving in storage", "VTM bots aren't doing anything", "bots stuck facing each
  other", "bots are idle but there's work to do", "pick stations running out of totes",
  "throughput has dropped", "one robot seems stuck and everything stopped",
  "warehouse storage area is not working".
  SKIP: HTM relay assignment issues — use /mvts/ops/relay instead.
---

# MVTS VTM Bot Stall Debugging

Diagnoses two related conditions that both appear as "storage bots not doing anything":

1. **Cyclic Deadlock** — Two bots are blocking each other in a circular standoff. Neither can move.
2. **Aisle Stall** — Bots are ready but can't get an assignment (no eligible aisles, relay points blocked, single bot stuck).

---

## Communication Protocol (always follow)

- Run ALL diagnostic steps silently. Show NO kubectl commands, jq queries, JSON, or InfluxDB rows to the user.
- After completing diagnostics, present ONE plain-English Diagnosis Summary.
- Translate every finding:
  - "assigned_aisle pointing to other bot's aisle" → "Two robots are waiting for each other's spot"
  - "aislesWithTasks=[]" → "The system sees no work available for storage robots"
  - "bot stuck at threshold >30 min" → "One robot has been sitting in the same aisle for over 30 minutes with tasks that haven't been confirmed complete"
  - "relay points blocked, tote: null" → "The pickup/dropoff areas in that aisle are locked at the warehouse level"
- Always provide the GMC escalation message when intervention is needed.
- Never ask the user to interpret technical output.

---

## Step 1 — Resolve environment and confirm pod running (silently)

```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

If pod is not Running → switch to `/mvts/ops/crash`.

---

## Step 2 — Save the problem statement (silently)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \" Message:\" /app/data/logs/scheduler.log | tail -1 | sed \"s/.*Message: //\" > /tmp/ps.json && echo done'"
```

---

## Step 3 — Determine which type of stall this is (silently)

Check the TAM strategy init log — this single line tells you everything:

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"TAM strategy init\" /app/data/logs/scheduler.log | tail -3'"
```

Then extract VTM bot state:

```bash
# Check VTM version present
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.ranger_list[].version] | unique\" /tmp/ps.json'"

# Extract current vs assigned aisle for all VTM bots
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".ranger_list[] | select(.version | startswith(\\\"VTM\\\")) | {id, current_aisle: .current_aisle_info, assigned_aisle: .ranger_schedule[0].aisle_info, status, available_capacity}\" /tmp/ps.json'"
```

**Decision rule:**
- If any bot has `assigned_aisle` pointing to the same aisle as another bot's `current_aisle`, AND vice versa → **Cyclic Deadlock** → go to Section A
- If all bots have `assigned_aisle: null` and are `ready` → **Aisle Stall** → go to Section B
- If only one bot is `processing` with the same tasks for >30 min → **Stuck Bot Stall** → go to Section B

---

## Section A — Cyclic Deadlock

### A1 — Identify the deadlocked pair (silently)

Build the mapping: `bot → current_aisle → assigned_aisle`

Look for:
1. **Direct cycle**: Bot A at aisle X wants aisle Y, Bot B at aisle Y wants aisle X
2. **Cascading blocks**: Other bots waiting to enter the same aisles

### A2 — Confirm via InfluxDB (silently)

Get the `InstallationId`:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep -i \"installation\" /app/data/config/local.application.properties'"
```

Check if assignments have stopped (deadlock signal):
```influxql
SELECT * FROM mvts_relay_vtm_bot_assignment_details
WHERE time > now() - 3h
AND "InstallationId" = '<installation_id>'
ORDER BY time DESC LIMIT 5
```

If same `task_chain` repeats every ~12s → confirmed deadlock.

Aisle switch history:
```influxql
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 24h
AND "InstallationId" = '<installation_id>'
AND ("bot_id_tag" = '<bot_a>' OR "bot_id_tag" = '<bot_b>')
ORDER BY time ASC
```

### A3 — Explain the deadlock to the user

When a deadlock is confirmed:
> "I found that [N] warehouse robots are caught in a circular block. Robot [A] is in aisle [X] and needs to move to aisle [Y], but Robot [B] is already in aisle [Y] and needs to move back to aisle [X]. Neither can move because the other is in the way. The scheduling system cannot resolve this on its own — the warehouse control system (GMC) needs to manually reposition one of the robots."

### A4 — Escalation message (always show for deadlock)

```
To: GMC Operations Team
Environment: [env name]
Time detected: [timestamp]
Priority: High — no VTM bot assignments for [X] hours

MVTS VTM Deadlock Alert

Two warehouse robots are caught in a circular block in the storage area and cannot resolve it automatically.

Deadlocked robots:
  • Robot [Bot A ID] is in aisle [X], assigned to move to aisle [Y]
  • Robot [Bot B ID] is in aisle [Y], assigned to move to aisle [X]

How the deadlock formed:
  [Time]: Robot [A] was sent to aisle [Y]
  [Time]: Robot [B] was in transit and assigned to aisle [X]
  [Time]: Both robots reached destination aisles simultaneously, creating the block

What is needed: Please reposition Robot [B] out of aisle [Y] to any free aisle.
After repositioning, MVTS will detect the change in the next solver cycle (~13 seconds) and resume assignments automatically.

Impact: All VTM storage bot assignments are paused. Totes are not moving from storage to relay points.
Duration: Deadlock confirmed since [time from InfluxDB aisle switch data].
```

---

## Section B — Aisle Stall (no deadlock, bots just not getting work)

### B1 — Read the TAM strategy log (silently)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"TAM strategy init\" /app/data/logs/scheduler.log | tail -3'"
```

What to look for:
```
TAM strategy init: 31 bots, 3 old assignments,
  aislesWithTasks=[30],     ← only aisles MVTS will assign bots to
  aislesOccupied=[30],      ← aisles already held by a bot
  aislesAvailable=[1,2,5...]
```

Key signals:
| Signal | Meaning |
|--------|---------|
| `aislesWithTasks=[]` | No VTM tasks at all — GMC upstream issue |
| `aislesWithTasks=[X]` narrow | Only one or two aisles have work; others blocked/empty |
| `aislesOccupied` ⊇ `aislesWithTasks` | Every aisle with tasks is held by a bot at max capacity |

Also check:
```
Greedy VTM assignment start: N eligible bots, M store-to-relay tasks, K relay-to-store totes
```
If `M + K = 0` → no VTM work exists. If `M + K > 0` but assignments = 0 → stuck bot blocking.

### B2 — Find stuck bot (silently)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"reached its assignment threshold\" /app/data/logs/scheduler.log | tail -5'"
```

If a bot hits threshold every cycle, it's stuck — it has tasks that GMC hasn't confirmed complete.

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".ranger_list[] | select(.version | startswith(\\\"VTM\\\") and .status == \\\"processing\\\") | {id, current_aisle: .current_aisle_info.aisle_id, available_at_time, tasks: [.ranger_schedule[] | {task_subtype, task_status, tote: .transport_entity_id}]}\" /tmp/ps.json'"
```

Verify via InfluxDB — when did the stuck bot last make a NEW assignment:
```influxql
SELECT * FROM mvts_relay_vtm_bot_assignment_details
WHERE time > now() - 12h
AND "InstallationId" = '<installation_id>'
AND bot_id = <bot_id>
ORDER BY time DESC LIMIT 10
```

If same `task_chain` totes appear repeatedly → tasks looping, not completing.

### B3 — Check relay point status per aisle (silently)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.relay_point_list[] | {aisle: .aisle_info.aisle_id, status, reserved: (.reserving_tote_id != null)}] | group_by(.aisle) | map({aisle: .[0].aisle, total: length, available: (map(select(.status == \\\"available\\\")) | length), occupied: (map(select(.status == \\\"occupied\\\")) | length), blocked: (map(select(.status == \\\"blocked\\\")) | length)}) | sort_by(.aisle)\" /tmp/ps.json'"
```

| Pattern | Meaning |
|---------|---------|
| All relay points `blocked`, no tote IDs | Aisle locked at GMC level — physically inaccessible or GMC bug |
| `occupied` relay points with same totes for hours | VTM task stuck executing; GMC not completing |
| `reserved` but 0 `occupied` | Tote reserved, waiting for HTM pickup — healthy |

### B4 — Config issue check (silently)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'curl -s localhost:8080/mvts/config/all | python3 -m json.tool | grep -iE \"vtm|threshold|static|aisle\"'"
```

Check: `ENABLE_STATIC_BOT_AISLE_MAP`. If `true`, compare #VTM bots vs #aisles:
```bash
# Count aisles
jq "[.relay_point_list[].aisle_info.aisle_id] | unique | length" /tmp/ps.json
# Count VTM bots
jq "[.ranger_list[] | select(.version | startswith(\"VTM\"))] | length" /tmp/ps.json
```

If aisles > bots and `ENABLE_STATIC_BOT_AISLE_MAP=true` → some aisles are permanently skipped.
Fix: `curl "localhost:8080/mvts/config/set?property=ENABLE_STATIC_BOT_AISLE_MAP&value=false"`

---

## Plain-English Root Cause Translations

| Technical finding | Tell the user |
|---|---|
| `aislesWithTasks=[]` | "The warehouse control system (GMC) has generated no work for storage robots. There may be no items queued for those aisles, or task generation may be paused upstream." |
| Single bot stuck `processing` same totes >30 min | "Robot [ID] has been sitting in aisle [X] for over 30 minutes with the same tasks assigned, but the warehouse control system hasn't confirmed those tasks as complete. While that robot is stuck, no other robot can be assigned to that aisle." |
| All relay points `blocked` (no tote), aisle excluded from `aislesWithTasks` | "The pickup/dropoff points in aisle [X] are locked at the warehouse control level — possibly due to physical obstruction or a system flag. Storage robots cannot operate in that aisle." |
| `ENABLE_STATIC_BOT_AISLE_MAP=true` and #aisles > #bots | "The system is configured to assign each robot to a fixed aisle, but there are more aisles than robots. Some aisles never get visited. This can be fixed with a config change." |
| `VTM_TASK_ASSIGNMENT_THRESHOLD` too high | "The minimum task count required before sending a robot to an aisle is set higher than the number of tasks available. Some aisles with work are being skipped." |

---

## Escalation Message Template — GMC Team (Aisle Stall)

```
To: GMC Operations Team
Environment: [env name]
Time detected: [timestamp]
Priority: [High if >1h / Medium if <1h]

MVTS VTM Aisle Stall

Storage robots (VTM) in [env] have stopped receiving assignments.
Pick station throughput is degrading.

Root cause found:
[Check all that apply]

□ Stuck robot: Robot [ID] has been in aisle [X] since [time] with the same tasks unacknowledged by GMC.
  Totes involved: [list tote IDs from ranger_schedule]
  → Please check task completion status for bot [ID] and those totes.

□ Blocked aisles: Aisle(s) [X, Y] have all relay IO points in a "blocked" state with no tote present.
  Relay IO point IDs: [TLOC IDs]
  → Please check the physical/logical status of those relay points.

□ No tasks generated: MVTS sees zero VTM tasks in the current problem statement.
  GMC may have paused VTM task generation, or no items are queued for these aisles.

Impact on pick stations: Pick stations [IDs] are running low on totes.
Stall confirmed since: [timestamp from InfluxDB aisle switch data]
```

---

## Diagnosis Summary (show to user at the end)

```
**Environment**: [env name]
**Type of issue**: [Circular deadlock between robots / Single robot stuck / No tasks available / Aisles blocked]
**What I found**: [1-2 plain-English sentences]
**Impact**: [e.g. "No storage robot assignments have been made for 2 hours. Pick stations 12 and 13 are running low on totes."]
**What needs to happen**: [one of:]
  → "The GMC team needs to manually reposition Robot [X]. Escalation message above."
  → "I've applied a config fix — the system should recover in the next 30 seconds."
  → "The GMC team needs to check the stuck robot's task completion. Escalation message above."
  → "This needs the GMC team to unblock aisles [X, Y]. Escalation message above."
```
