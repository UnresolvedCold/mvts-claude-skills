---
description: >
  MVTS Relay Assignment Debugging — diagnoses why HTM bots are not being assigned to relay tasks.
  TRIGGER when: totes are at relay points but HTM bots are not getting assigned, relay assignment stuck,
  RelaySubsystemInitializer warnings in logs, bots idle with pending relay tasks.
  SKIP: VTM deadlock issues — use mvts-vtm-deadlock instead.
---

# MVTS Relay Assignment Debugging

**Issue pattern**: Totes are at relay points but HTM bots are not getting assigned.

---

## Prerequisites

Resolve the namespace and ensure pod is running. See `mvts-debug` for environment resolution and access patterns.

Key file locations inside pod:
- Live log: `/app/data/logs/scheduler.log`
- Config: `/app/data/config/local.application.properties`

---

## Step 1 — Get a reference bot and task

Grep for the relay initializer warning:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep "RelaySubsystemInitializer: Unable to find a time sample" /app/data/logs/scheduler.log | tail -5'
```

This gives: `bot <id>` and `task <task_key>`.

---

## Step 2 — Save the problem statement

```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep " Message:" /app/data/logs/scheduler.log | tail -1 | sed "s/.*Message: //" > /tmp/ps.json'
```

Always write to `/tmp/ps.json` first — the JSON is ~950KB and piping directly causes truncation.

---

## Step 3 — Sanity check the problem statement

**Bot** (field is `id`, not `ranger_id`):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".ranger_list[] | select(.id == <bot_id>) | {id, version, available_capacity, available_at_coordinate, status}" /tmp/ps.json'
```
- `version` should be `HTM_QT_M5F`
- `available_capacity` should be > 0

**Task**:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".task_list[] | select(.task_key == \"<task_key>\")" /tmp/ps.json'
```
- `status` should be `to_be_assigned`
- Note `transport_entity_id` (the tote, e.g. `RELAYY_XXXX`) and `destination_id` (PPS)

**Relay point** (find by tote ID):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".relay_point_list[] | select(.reserving_tote_id == \"<RELAYY_XXXX>\")" /tmp/ps.json'
```
- Note `htm_io_point` coordinates and `aisle_info`

**PPS**:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".pps_list[] | select(.id == <pps_id>)" /tmp/ps.json'
```
- `pps_status` should be active
- `can_assign_task` should be `true`
- `pps_type` should be `RELAY`
- `supported_butler` should include the HTM bot version

---

## Step 4 — Check IDC for both relay legs

**Infinite IDC = `10000000`** — no valid path exists between those coordinates for that bot version.

IDC endpoint runs on port 8383 inside the pod:

**Leg 1** — bot's current position → relay point's `htm_io_point`:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8383/idc/calculate?x1=<bot_x>&y1=<bot_y>&x2=<relay_x>&y2=<relay_y>&floor=1&liftState=down&botVersion=HTM_QT_M5F"'
```

**Leg 2** — relay point's `htm_io_point` → PPS `ranger_dock_coordinates`:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8383/idc/calculate?x1=<relay_x>&y1=<relay_y>&x2=<pps_x>&y2=<pps_y>&floor=1&liftState=down&botVersion=HTM_QT_M5F"'
```

Returns travel time in milliseconds. `10000000` = infinite = no path.

**To find which relay point a coordinate belongs to:**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".relay_point_list[] | select(.htm_io_point.x == <x> and .htm_io_point.y == <y>)" /tmp/ps.json'
```

---

## Step 5 — Check order promotion (new-framework PICK)

In the new framework, **PICK promotes an order only when ALL totes required by that order are at relay**. If even one sibling tote is still in storage, the order stays in `OrderForPromotionCache`, no HTM relay→PPS extraction task is generated, and the tote at relay looks "stuck" even though bots, IDC, and PPS state are all fine.

**Telltale log signature** (every planning cycle, no assignment):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep -E "MSIOOrderSelectionStrategy|Selected orders for promotion|PPS Promoted Totes" /app/data/logs/scheduler.log | grep -v "QueueManager: Message:" | tail -20'
```

Diagnostic — orders cached but never promoted:
```
MSIOOrderSelectionStrategy: BinTagGroup: <bg>, AllowedPromotions: 0
MSIOOrderSelectionStrategy: For bin tag group <bg>, pps <id>, number of bins = N, unassigned orders = M, cached orders = K, new promotable orders = 0
No new orders to promote for bin tag group <bg>
Selected orders for promotion for pps <id> and Bin Tag Group <bg> are []
PPS Promoted Totes: {}
```

When you see this AND bots/IDC/PPS look healthy, the issue is upstream: PICK hasn't promoted the orders because not all required totes have reached relay.

**Find the sibling totes** — extract orders on the stuck tote, then find the other totes serving the same orders:
```bash
# Orders on the stuck tote
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".task_list[] | select(.transport_entity_id == \"<tote_id>\") | .serviced_orders[].order_id" /tmp/ps.json'

# All tasks that mention those order IDs (sibling totes for the same orders)
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq --arg oid "<order_id>" "[.task_list[] | select(.serviced_orders[]?.order_id == \$oid) | {task_key, transport_entity_id, status, aisle_info, dest: .destination_id}]" /tmp/ps.json'
```

Then check `transport_entity_list` (or `relay_point_list[].reserving_tote_id`) to see which sibling totes are already at relay vs still in storage. Any sibling NOT at relay = the reason PICK hasn't promoted.

**Also check** the PPS `bin_details`:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".pps_list[] | select(.id == <pps_id>) | .bin_details" /tmp/ps.json'
```
If all bins show `is_virtual_bin_used: true` and `PRIORITIZE_VIRTUAL_BIN_TASKS_OVER_MSIO_TASKS=true`, virtual-bin tasks may be holding the slots — confirm none are stuck.

---

## Root causes

| Finding | Root cause | Fix |
|---------|-----------|-----|
| `AllowedPromotions: 0` / `Selected orders for promotion … are []` while bots & IDC are healthy | New-framework: PICK has not promoted the order because not all sibling totes for it are at relay yet | Trace sibling totes; resolve whatever is keeping them in storage (VTM stuck, IDC, aisle constraint) |
| All `bin_details[].is_virtual_bin_used = true` + promotion blocked | Virtual-bin tasks are holding all bin slots and blocking MSIO promotion (`PRIORITIZE_VIRTUAL_BIN_TASKS_OVER_MSIO_TASKS=true`) | Investigate stuck virtual-bin tasks upstream |
| IDC = 10000000 on leg 1 | HTM cannot reach relay point | Update IDC / fix map |
| IDC = 10000000 on leg 2 | Relay point cannot reach PPS | Update IDC / fix map |
| `can_assign_task = false` on PPS | PPS blocked or offline | Check PPS state in GMC |
| `available_capacity = 0` on bot | Bot is full | Wait for bot to drop tote |
| Task `status != to_be_assigned` | Task already assigned or completed | Check with GMC |

---

## Things MVTS currently ignores (don't be misled)

- `relay_point_list[].available_at_time` — even a far-future value (e.g. `start_time + 24h`) does NOT filter the tote from HTM planning. Don't treat a "+24h" sentinel here as the cause.
