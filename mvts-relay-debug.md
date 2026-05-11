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

## Root causes

| Finding | Root cause | Fix |
|---------|-----------|-----|
| IDC = 10000000 on leg 1 | HTM cannot reach relay point | Update IDC / fix map |
| IDC = 10000000 on leg 2 | Relay point cannot reach PPS | Update IDC / fix map |
| `can_assign_task = false` on PPS | PPS blocked or offline | Check PPS state in GMC |
| `available_capacity = 0` on bot | Bot is full | Wait for bot to drop tote |
| Task `status != to_be_assigned` | Task already assigned or completed | Check with GMC |
