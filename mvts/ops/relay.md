---
description: >
  MVTS HTM relay assignment — diagnoses why HTM bots are not being assigned to relay tasks.
  TRIGGER when: totes are at relay points but HTM bots are not getting assigned, relay assignment
  stuck, RelaySubsystemInitializer warnings, bots idle with pending relay tasks.
  TRIGGER also when: user says "totes are waiting but no bots come", "HTM bots are idle",
  "totes stuck at the relay area", "orders stuck waiting for a bot", "bots aren't going to the
  relay points", "things are piling up at relay", "pick stations have no totes arriving".
  SKIP: VTM deadlock issues — use /mvts/ops/vtm-stuck instead.
---

# MVTS Relay Assignment Debugging

**Issue pattern**: Totes are at relay points but HTM bots are not being assigned to pick them up.

---

## Communication Protocol (always follow)

- Run ALL diagnostic steps silently. Show NO kubectl commands, jq queries, JSON, or log lines to the user.
- After completing all steps, present a single plain-English Diagnosis Summary.
- Translate every finding. Examples:
  - "IDC = 10000000" → "No navigable route exists between those two points"
  - "available_capacity = 0" → "Bot is fully loaded and cannot take another tote"
  - "can_assign_task = false" → "Pick station is marked unavailable"
  - "AllowedPromotions: 0" → "The system is waiting for all totes in the same order to arrive before assigning a bot"
- When the root cause requires GMC intervention, show the escalation message at the end.
- Never ask the user to interpret output. All reasoning is done by you.

---

## Diagnostic Steps (run silently — do not show output to user)

### Step 1 — Resolve environment and confirm pod running

```bash
# On-prem
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"

# GCP
mcp__gor-global-mcp__list_environments
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

If pod is not Running → switch to `/mvts/ops/crash` immediately.

### Step 2 — Get a reference bot and task from logs

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"RelaySubsystemInitializer: Unable to find a time sample\" /app/data/logs/scheduler.log | tail -5'"
```

This gives: `bot <id>` and `task <task_key>`.

### Step 3 — Save the problem statement

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \" Message:\" /app/data/logs/scheduler.log | tail -1 | sed \"s/.*Message: //\" > /tmp/ps.json && echo done'"
```

Always write to `/tmp/ps.json` first — JSON is ~950KB and piping directly causes truncation.

### Step 4 — Inspect bot, task, relay point, and PPS state

**Bot** (field is `id`, not `ranger_id`):
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".ranger_list[] | select(.id == <bot_id>) | {id, version, available_capacity, available_at_coordinate, status}\" /tmp/ps.json'"
```
Check: `version` = `HTM_QT_M5F`, `available_capacity` > 0.

**Task**:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".task_list[] | select(.task_key == \\\"<task_key>\\\")\" /tmp/ps.json'"
```
Check: `status` = `to_be_assigned`. Note `transport_entity_id` (tote) and `destination_id` (PPS).

**Relay point** (find by tote ID):
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".relay_point_list[] | select(.reserving_tote_id == \\\"<RELAYY_XXXX>\\\")\" /tmp/ps.json'"
```
Note `htm_io_point` coordinates and `aisle_info`.

**PPS**:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".pps_list[] | select(.id == <pps_id>)\" /tmp/ps.json'"
```
Check: `pps_status` active, `can_assign_task` = `true`, `pps_type` = `RELAY`.

### Step 5 — Check IDC for both relay legs

Infinite IDC = `10000000` — no valid path.

**Leg 1** — bot position → relay point `htm_io_point`:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'curl -s \"localhost:8383/idc/calculate?x1=<bot_x>&y1=<bot_y>&x2=<relay_x>&y2=<relay_y>&floor=1&liftState=down&botVersion=HTM_QT_M5F\"'"
```

**Leg 2** — relay point → PPS `ranger_dock_coordinates`:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'curl -s \"localhost:8383/idc/calculate?x1=<relay_x>&y1=<relay_y>&x2=<pps_x>&y2=<pps_y>&floor=1&liftState=down&botVersion=HTM_QT_M5F\"'"
```

### Step 6 — Check order promotion (new-framework PICK)

In the new framework, PICK promotes an order only when ALL totes for that order are at relay. If any sibling tote is still in storage, no HTM relay→PPS task is generated.

```bash
# Check for blocked promotions
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep -E \"MSIOOrderSelectionStrategy|Selected orders for promotion|PPS Promoted Totes\" /app/data/logs/scheduler.log | grep -v \"QueueManager: Message:\" | tail -20'"
```

Diagnostic — orders cached but never promoted:
```
MSIOOrderSelectionStrategy: AllowedPromotions: 0
Selected orders for promotion for pps <id> ... are []
PPS Promoted Totes: {}
```

When this appears AND bots/IDC/PPS look healthy → sibling totes are still in storage.

```bash
# Orders on the stuck tote
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".task_list[] | select(.transport_entity_id == \\\"<tote_id>\\\") | .serviced_orders[].order_id\" /tmp/ps.json'"

# All totes for the same orders (sibling totes)
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq --arg oid \"<order_id>\" \"[.task_list[] | select(.serviced_orders[]?.order_id == \\\$oid) | {task_key, transport_entity_id, status, aisle_info, dest: .destination_id}]\" /tmp/ps.json'"
```

---

## Plain-English Root Cause Translations

After completing all diagnostic steps, map the finding to one of these:

| Technical finding | Tell the user |
|-------------------|---------------|
| `AllowedPromotions: 0`, sibling totes not at relay | "The system is waiting for all totes belonging to the same order group to arrive at the relay area before it assigns a robot. Some of those totes are still being fetched from storage — this is expected behavior. Bots will be assigned once all totes in the group arrive." |
| IDC = 10000000 on leg 1 | "There's no navigable route for a delivery bot to reach that relay pickup point. This is a warehouse map configuration issue that needs an engineer." |
| IDC = 10000000 on leg 2 | "There's no navigable route from the relay pickup point to the pick station. This is a warehouse map configuration issue." |
| `can_assign_task = false` on PPS | "Pick station [ID] is marked as unavailable in the system. The GMC team needs to check that station's status." |
| `available_capacity = 0` on bot | "The bot is fully loaded and can't take another tote. It needs to deliver its current tote first." |
| All `bin_details.is_virtual_bin_used = true` + promotions blocked | "Internal virtual storage tasks are occupying all slots at the pick station, blocking normal orders. The GMC team needs to investigate." |
| `pps_type != RELAY` | "The pick station is not configured for relay mode. This is a configuration issue." |
| No `RelaySubsystemInitializer` warnings at all | "The relay subsystem is not reporting any stuck assignments. The issue may have resolved itself or may be intermittent." |

---

## Things MVTS ignores (don't be misled by these — internal note)

- `relay_point_list[].available_at_time` — even a far-future value (e.g. `start_time + 24h`) does NOT filter the tote from HTM planning. A "+24h" sentinel here is NOT the root cause.

---

## Escalation Message Template — GMC Team

Show this when the root cause requires GMC intervention:

```
To: GMC Operations Team
Environment: [env name]
Time reported: [timestamp]

Summary: HTM delivery bots are not picking up totes from the relay area in [env].

What MVTS diagnostics show:
[Check all that apply and fill in details]

□ Pick station [ID] is marked unavailable (can_assign_task = false).
  → Please check the status of pick station [ID] and re-enable it if appropriate.

□ Tote [ID] has been at relay point [relay point ID] since [time] but is waiting for
  sibling tote [ID] to arrive from storage. The sibling tote has status [status].
  → Please check why that tote is not being moved from storage.

□ No navigable route exists for a delivery bot to reach relay point [ID] (IDC = infinite).
  → Map or routing configuration issue — needs engineer review.

□ Relay point [ID] is in a blocked state with no tote present.
  → This appears to be a GMC-side lock on that relay point.

Impact: [N] totes are waiting. Orders are not being fulfilled at pick station [ID].
Duration: Issue first observed at [time].
```

---

## Diagnosis Summary (show to user at the end)

```
**Environment**: [env name]
**What I checked**: Delivery bot availability, navigation routes to relay points, pick station status, and order promotion state.
**What I found**: [1-2 plain-English sentences on the root cause]
**Impact**: [e.g. "3 totes have been waiting at relay points for 45 minutes. Orders at pick station 12 are not being fulfilled."]
**What needs to happen**: [one of:]
  → "This is expected — waiting for sibling totes to arrive. No action needed."
  → "This needs the GMC team to intervene. Escalation message above."
  → "This needs an engineer — there's a map/routing issue."
  → "I found no clear cause. Monitoring the next few cycles."
```
