---
description: >
  MVTS Debugging Guide — step-by-step playbook for debugging MVTS issues across any environment.
  TRIGGER when: user mentions MVTS, TPQ, VTM deadlock, HTM assignment, relay assignment, PPS starvation,
  high queue length, bots not assigned, problem statement debugging, IDC infinite, CrashLoopBackOff on mvts-0,
  or any debugging task in a named relay/bulk/staging environment.
  SKIP: general Kubernetes questions unrelated to MVTS; questions about GMC, navigation, or other services
  unless they are being debugged in the context of an MVTS issue.
argument-hint: Optional environment name (e.g. "qa4-relaypotepic", "onmbulk", "aphrelaybulk")
allowed-tools: ["Bash", "mcp__gor-global-mcp__list_environments", "mcp__gor-global-mcp__kube_connect",
  "mcp__gor-global-mcp__kube_shell", "mcp__gor-global-mcp__kube_pod_logs",
  "mcp__gor-global-mcp__query_metrics", "mcp__gor-global-mcp__run_influxdb_query_from_catalog",
  "mcp__gor-global-mcp__search_influxql_catalog", "mcp__gor-global-mcp__describe_measurement",
  "mcp__gor-global-mcp__search_logs", "mcp__gor-global-mcp__list_indices",
  "mcp__gor-global-mcp__health_check"]
---

# MVTS Debugging Guide

A step-by-step reference for debugging MVTS issues across any environment.

---

## First-Run Setup

**Run this section automatically on first invocation**, or when the user says "setup" or "configure this skill". Check each prerequisite and fix it if missing — do not ask the user to do things Claude can do automatically.

### 1. SSH JumpServer (required for stpbulk-cluster access)

Check if the JumpServer entry exists:
```bash
grep -c "Host JumpServer" ~/.ssh/config 2>/dev/null || echo "0"
```

If the count is `0`, add the entry. Prompt the user once for the JumpServer IP if unknown (default: `192.168.9.237`):
```bash
# Create ~/.ssh/config if it doesn't exist
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/config << 'EOF'

Host JumpServer
    HostName 192.168.9.237
    User shubham.kumar
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
chmod 600 ~/.ssh/config
```

Verify connectivity:
```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes JumpServer "echo OK" 2>&1
```

If `OK` is returned, JumpServer is reachable. If not, check VPN/network or ask the user to confirm the IP.

### 2. SSH key (if JumpServer auth fails)

Check if the default key exists:
```bash
ls ~/.ssh/id_rsa 2>/dev/null && echo "KEY_EXISTS" || echo "MISSING"
```

If missing, generate one and remind the user to add the public key to the JumpServer:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N "" -C "mvts-debug-skill"
echo "Public key to register on JumpServer:"
cat ~/.ssh/id_rsa.pub
```

### 3. gor-global-mcp MCP permissions

Check if the required MCP tools are in the allowlist:
```bash
python3 -c "
import json, os
path = os.path.expanduser('~/.claude/settings.local.json')
try:
    d = json.load(open(path))
    allowed = d.get('permissions', {}).get('allow', [])
    needed = [
        'mcp__gor-global-mcp__list_environments',
        'mcp__gor-global-mcp__kube_connect',
        'mcp__gor-global-mcp__kube_shell',
        'mcp__gor-global-mcp__kube_pod_logs',
        'mcp__gor-global-mcp__query_metrics',
        'mcp__gor-global-mcp__run_influxdb_query_from_catalog',
        'mcp__gor-global-mcp__search_influxql_catalog',
        'mcp__gor-global-mcp__describe_measurement',
        'mcp__gor-global-mcp__health_check',
    ]
    missing = [t for t in needed if t not in allowed]
    print('MISSING:', missing if missing else 'none')
except Exception as e:
    print('ERROR:', e)
"
```

If any are missing, add them:
```bash
python3 << 'EOF'
import json, os

path = os.path.expanduser('~/.claude/settings.local.json')
needed = [
    'mcp__gor-global-mcp__list_environments',
    'mcp__gor-global-mcp__kube_connect',
    'mcp__gor-global-mcp__kube_shell',
    'mcp__gor-global-mcp__kube_pod_logs',
    'mcp__gor-global-mcp__query_metrics',
    'mcp__gor-global-mcp__run_influxdb_query_from_catalog',
    'mcp__gor-global-mcp__search_influxql_catalog',
    'mcp__gor-global-mcp__describe_measurement',
    'mcp__gor-global-mcp__health_check',
    'Bash(ssh JumpServer *)',
    'Bash(ssh -o ConnectTimeout=* JumpServer *)',
]
try:
    d = json.load(open(path))
except (FileNotFoundError, json.JSONDecodeError):
    d = {}
d.setdefault('permissions', {}).setdefault('allow', [])
for t in needed:
    if t not in d['permissions']['allow']:
        d['permissions']['allow'].append(t)
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
print('Done — permissions updated')
EOF
```

### 4. gor-global-mcp server availability

The `gor-global-mcp` MCP server is provisioned automatically by the Claude Code enterprise integration for GOR org members — no manual install needed. If `mcp__gor-global-mcp__list_environments` is unavailable in the current session, the server may still be connecting. Wait 10s and retry, or restart the Claude Code session.

### 5. Git distribution — installing this skill

To install this skill from the git repo on a new machine:
```bash
# Clone the repo
git clone git@github.com:greyorange/mvts-claude-skills.git ~/Projects/GreyOrange/mvts-claude-skills

# Symlink skills into Claude's commands directory
mkdir -p ~/.claude/commands
ln -sf ~/Projects/GreyOrange/mvts-claude-skills/mvts-debug.md ~/.claude/commands/mvts-debug.md
ln -sf ~/Projects/GreyOrange/mvts-claude-skills/mvts-build.md ~/.claude/commands/mvts-build.md
```

Then run `/mvts-debug setup` in Claude Code — it will execute this checklist automatically.

---

## Infrastructure Access

### Step 0 — Resolve the Environment

When the user names an environment (e.g. `qa4-relaypotepic`, `onmbulk`, `aphrelaybulk`):

1. Call `mcp__gor-global-mcp__list_environments` to resolve the alias to a full namespace
2. The namespace tells you the cluster and access method:

| Namespace prefix | Cluster | Access method |
|-----------------|---------|---------------|
| `qa4-cluster-*` | `qa4-cluster` | GCP SA — use `kube_connect` + `kube_shell` |
| `stpbulk-cluster-*` | `stpbulk-cluster` | On-prem — use SSH JumpServer |

If `$ARGUMENTS` is set, that is the environment name — resolve it immediately before doing anything else.

### GCP Clusters (qa4-cluster-*, etc.)

Connect once per session, then use `kube_shell` for all commands:

```
# Connect (do this once)
mcp__gor-global-mcp__kube_connect(cluster="qa4-cluster", project="greymatter-qa")

# Check pod
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")

# Exec into pod
mcp__gor-global-mcp__kube_shell(command="kubectl exec mvts-0 -n <namespace> -- bash -c '<command>'")

# Logs
mcp__gor-global-mcp__kube_pod_logs(pod="mvts-0", namespace="<namespace>", ...)
```

### On-Prem Clusters (stpbulk-cluster-*)

Use SSH JumpServer. Always pass `-n <namespace>` explicitly:

- JumpServer alias: `JumpServer` (configured in `~/.ssh/config`, IP `192.168.9.237`)
- Key: `~/.ssh/id_rsa`

```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c '<command>'"
ssh JumpServer "kubectl logs mvts-0 -n <namespace> --previous | tail -50"
```

### MVTS Pod

- Pod name: `mvts-0`
- Container: `ml-engine-mvts`

---

## Key File Locations (inside pod)

| File | Path |
|------|------|
| Config | `/app/data/config/local.application.properties` |
| Live log | `/app/data/logs/scheduler.log` |
| Archived logs | `/app/data/logs/scheduler.YYYY-MM-DD.N.log.gz` |

---

## Basic Debugging

### 1. Check pod health

**GCP cluster:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

**On-prem:**
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

### 2. Read config

**GCP cluster:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl exec mvts-0 -n <namespace> -- bash -c 'cat /app/data/config/local.application.properties'")
```

**On-prem:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'cat /app/data/config/local.application.properties'"
```

Key config flags to note:
- `ENABLE_RELAY`, `ENABLE_KAFKA`, `ENABLE_TRANSIT_TIME_MODEL`
- `LOG_TO_INFLUXDB`, `INFLUX_DB_NAME`
- Bot version lists: `htm.bots`, `vtm.bots`
- `ENABLE_VTM_TASK_REASSIGNMENT`, `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM`

Or get all config as JSON via API:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'
```

### Changing config at runtime (no restart needed)

**Single property (GET):**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8080/mvts/config/set?property=<PROPERTY_NAME>&value=<value>"'
```

**Multiple properties at once (POST):**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s -X POST localhost:8080/mvts/config/set -H "Content-Type: application/json" -d "{\"PROPERTY_1\": \"value1\", \"PROPERTY_2\": \"value2\"}"'
```

The endpoint returns `{property, old_value, new_value}` confirming the change. It also persists the change to `local.application.properties` on disk.

### 3. Check input message (problem statement sanity)

Don't read all logs — grep for the input message directly:

```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep " Message:" /app/data/logs/scheduler.log | tail -1 | sed "s/.*Message: //" > /tmp/ps.json'
```

Always write to `/tmp/ps.json` inside the pod first — the JSON is large (~950KB) and piping it directly through jq causes truncation.

Top-level keys in the problem statement:
`task_list`, `ranger_list`, `relay_point_list`, `pps_list`, `adjacency_list`, `transport_entity_list`, `synchronization_info_list`, `start_time`, `request_id`, `planning_duration_seconds`, `maximize_picks`

### 4. Extracting entities with jq (always from /tmp/ps.json inside the pod)

**Get a specific request ID's message:**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep " Message:" /app/data/logs/scheduler.log | grep "<request_id>" | sed "s/.*Message: //" > /tmp/ps.json'
```

**Bot by ID** (field is `id`, not `ranger_id`):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".ranger_list[] | select(.id == <bot_id>) | {id, coordinate, available_at_coordinate, version, status, available_capacity}" /tmp/ps.json'
```

**Task by task_key:**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".task_list[] | select(.task_key == \"<task_key>\")" /tmp/ps.json'
```
Key fields: `transport_entity_id` (tote ID like `RELAYY_XXXX`), `destination_id` (PPS id), `status`, `assigned_ranger_id`

**Relay point by tote ID** (field is `reserving_tote_id`):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".relay_point_list[] | select(.reserving_tote_id == \"<RELAYY_XXXX>\")" /tmp/ps.json'
```
Key fields: `coordinate`, `htm_io_point`, `vtm_io_point`, `aisle_info`, `status`

**PPS by ID:**
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".pps_list[] | select(.id == <pps_id>)" /tmp/ps.json'
```
Key fields: `coordinate`, `ranger_dock_coordinates`, `pps_type`, `pps_status`, `can_assign_task`, `supported_butler`, `queue_length`

### 5. Get IDC (travel time) between two coordinates

IDC endpoint runs on port 8383 inside the pod:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8383/idc/calculate?x1=<x1>&y1=<y1>&x2=<x2>&y2=<y2>&floor=1&liftState=down&botVersion=<HTM_QT_M5F|VTM_QT_C56_S_2D>"'
```
Returns travel time in **milliseconds**.

For relay assignment debugging, check two legs:
1. Bot's `available_at_coordinate` → relay point's `htm_io_point`
2. Relay point's `htm_io_point` → PPS `ranger_dock_coordinates`

### 6. Search logs for a request ID

```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep "<request_id>" /app/data/logs/scheduler.log | grep -v "QueueManager: Message:"'
```

To get only WARN/ERROR lines, save output to a local file and grep:
```bash
grep -E "^2026.*(WARN|ERROR)" <saved_output_file>
```

> **Note**: Logs are not very robust — use them for initial orientation only. Prefer verifying against the problem statement directly.

---

## CrashLoopBackOff Debugging

### Check why pod is crashing

```bash
kubectl logs mvts-0 -n <namespace> --previous | tail -50
```

**Common cause — corrupt JAR**: `Invalid or corrupt jarfile app.jar`
Fix with a StatefulSet bounce:
```bash
kubectl scale sts mvts -n <namespace> --replicas=0 && sleep 5 && kubectl scale sts mvts -n <namespace> --replicas=1
```
This forces a fresh pull/copy of the JAR and usually resolves it. Not a code or deployment issue.

---

## Relay Assignment Debugging

**Issue pattern**: Totes are at relay points but HTM bots are not getting assigned.

### Step 1 — Get a reference bot and task
Grep for the relay initializer warning:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep "RelaySubsystemInitializer: Unable to find a time sample" /app/data/logs/scheduler.log | tail -5'
```
This gives: `bot <id>` and `task <task_key>`.

### Step 2 — Sanity check the problem statement
Save the matching problem statement to `/tmp/ps.json`, then check:
1. **Bot**: version (should be `HTM_QT_M5F`), `available_capacity`, `available_at_coordinate`
2. **Task**: `status` (should be `to_be_assigned`), `transport_entity_id` (the tote), `destination_id` (the PPS)
3. **Relay point**: find by `reserving_tote_id == transport_entity_id` — check `htm_io_point`, `aisle_info`
4. **PPS**: check `pps_status`, `can_assign_task`, `supported_butler`, `pps_type` (should be `RELAY`)

### Step 3 — Check IDC for both legs

**Infinite IDC = `10000000`** — means no valid path exists between those coordinates for that bot version.

```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8383/idc/calculate?x1=<bx>&y1=<by>&x2=<rx>&y2=<ry>&floor=1&liftState=down&botVersion=HTM_QT_M5F"'
```

**NEVER patch the StatefulSet directly** (`kubectl set image sts/mvts ...`) — the CD pipeline will detect drift and revert it. Always update `values.yaml`, commit, and push to the solution branch.

To verify the image actually running on a pod:
```bash
kubectl describe pod mvts-0 -n <namespace> | grep -i 'Image:'
```

**Root cause when IDC is infinite**: The HTM bot cannot reach the relay point due to a missing or blocked path. Fix: update the IDC or fix the map.

To find which relay point a coordinate belongs to:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".relay_point_list[] | select(.htm_io_point.x == <x> and .htm_io_point.y == <y>)" /tmp/ps.json'
```

---

## VTM Bot Cyclic Deadlock Debugging

**Symptom**: No VTM bots are getting assigned tasks.

### Step 1 — Get latest problem statement and extract VTM bot state
First check what VTM version is present (may differ per environment):
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq "[.ranger_list[].version] | unique" /tmp/ps.json'
```
Then extract current vs assigned aisle for all VTM bots:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'jq ".ranger_list[] | select(.version == \"VTM_QT_C56_GT_2D\") | {id, current_aisle: .current_aisle_info, assigned_aisle: .ranger_schedule[0].aisle_info, status, available_capacity}" /tmp/ps.json'
```

### Step 2 — Identify deadlock cycles
Build a mapping of: `bot → current_aisle → assigned_aisle`

Look for:
1. **Direct cycle**: Bot A at aisle X wants aisle Y, Bot B at aisle Y wants aisle X → neither can move
2. **Cascading blocks**: Other bots waiting to enter aisles involved in the cycle

### Step 3 — Confirm via InfluxDB transitive movement tables

Use `mcp__gor-global-mcp__query_metrics` with the environment alias.

**Check if any VTM assignments are happening at all:**
Query `mvts_relay_vtm_bot_assignment_details` — if this table is empty or has no recent rows (e.g. no data for 3+ hours), no VTM assignments have been made, which strongly indicates a deadlock.

**Visualize the deadlock graph:**
```
mvts_relay_vtm_transitive_movements_nodes   -- aisles as nodes with color/state
mvts_relay_vtm_transitive_movements_edges   -- bot movements as directed edges
```

Node color legend:
| Color | Hex | Meaning |
|-------|-----|---------|
| Purple | `#A78BFA` (`highlighted=true`) | Deadlocked aisle |
| Amber | `#FBBF24` | Occupied (bot present, not deadlocked) |
| Green | `#22C55E` | Free-moving |
| Gray | `#6B7280` | Empty aisle |

**Key signal**: If the same set of purple nodes appears unchanged across multiple consecutive `request_id` values (~13s apart), the deadlock is persistent.

### Step 4 — Trace the full cascade
For each blocked bot, check what aisle it's waiting for and whether that aisle's occupant is also blocked. Build the full dependency chain to find the root cycle.

### Step 5 — Traceback how the deadlock formed (InfluxDB)

Use `mvts_relay_vtm_bot_aisle_switches` to reconstruct the sequence of aisle assignments leading up to the deadlock.

**Schema**: `bot_id` (field, integer), `bot_id_tag` (tag), `from_aisle`, `to_aisle`, `transitive_aisles`, `traversal_aisles`, `request_id`

**Negative `from_aisle` values = dummy aisles** — bot was not in any real aisle (in transit, idle, or charging).

```influxql
-- All switches for the deadlocked bots (last 24h)
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

-- Bots coming from dummy aisles (startup or returning to operation)
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 48h
AND "InstallationId" = '<installation_id>'
AND from_aisle < 0
ORDER BY time ASC
```

**Traceback pattern to follow:**
1. Find the deadlocked bot pair (A at aisle X wants Y, B at aisle Y wants X)
2. Query aisle switches for both bots — build their full aisle history leading up to the deadlock
3. For each aisle involved, query who was there before — find the last bot to vacate before the deadlock assignment
4. Check if the previous occupant had already left before MVTS made the conflicting assignment
5. Look for a bot that ended up in a dummy aisle just before being assigned into the deadlock

```influxql
-- Last real tasks each bot executed
SELECT * FROM mvts_relay_vtm_bot_assignment_details
WHERE time > now() - 12h
AND "InstallationId" = '<installation_id>'
AND (bot_id = <A> OR bot_id = <B>)
ORDER BY time ASC
```
When both bots repeat the **exact same task_chain** every ~12s, the deadlock is confirmed.

**Root cause pattern:**
- Bot A finishes tasks in aisle X, MVTS sends it to aisle Y
- Bot B is in a dummy aisle (in transit); aisle X gets new tasks, MVTS assigns Bot B → aisle X
- Bot B physically ends up at aisle Y on its way to X
- Now: Bot A at X wants Y (Bot B is there), Bot B at Y wants X (Bot A is there) → deadlock
- MVTS made two independently valid decisions without detecting the physical path conflict

### Notes
- `current_aisle_info` can be `null` if bot is in transit between aisles
- Multiple bots can be assigned to the same aisle (contention, not necessarily deadlock)
- The core deadlock is always a cycle — fixing it unblocks the entire cascade
- **MVTS cannot resolve cyclic deadlocks** — this requires GMC action. Escalate to GMC team.

---

## Key InfluxDB Tables

All tables are in the `GreyOrange` database. Use `mcp__gor-global-mcp__query_metrics` with the environment alias. Filter by `InstallationId` (or `installation_id` depending on table) tag.

To find the `InstallationId` for an environment:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep -i "installation\|influx" /app/data/config/local.application.properties'
```

### `mvts_rtp_problem_assignment_details` — HTM assignments (written by MVTS)
One row per HTM bot assignment per solver cycle.

| Field | Description |
|-------|-------------|
| `assigned_bot_id` | Tag — HTM bot ID |
| `assigned_pps_id` | Tag — PPS the tote is going to |
| `msu_id` | Tote ID (`RELAYY_XXXX`) |
| `task_id` / `counter_id` | Task UUID and order reference |
| `assignment_type` | `new_task`, `cached`, or `current_schedule` — see below |
| `bot_start_time` | When MVTS wants the bot to start moving |
| `pps_queue_reach_time` | When MVTS predicts the bot will reach the PPS queue entrance |
| `start_time` | When MVTS predicts the operator will start working on the tote |
| `end_time` | Predicted operator finish time |
| `predicted_transit_time` | `pps_queue_reach_time - bot_start_time` in ms |
| `available_start_time` | When this bot is expected to be free for the next task |
| `problem_statement_time` / `request_id` | Identifies which solver cycle produced this row |
| `bins` / `entity_pick_sequence` | Bins to be picked and their sequence number |

**Time relationships:**
```
bot_start_time ──transit──► pps_queue_reach_time ──queue wait──► start_time ──op time──► end_time
                 (predicted_transit_time)          (start - reach)             (end - start)
```

**Assignment types:**
| Type | Meaning |
|------|---------|
| `new_task` | Freshly assigned this MVTS cycle — GMC hasn't dispatched it yet |
| `cached` | Sent to GMC in a prior cycle but GMC hasn't acknowledged/dispatched it yet |
| `current_schedule` | GMC has dispatched it — bot is actively executing the task |

```influxql
SELECT * FROM mvts_rtp_problem_assignment_details
WHERE time > now() - 1h AND "InstallationId" = '<id>'
AND "assigned_bot_id" = '<bot_id>'
ORDER BY time DESC
```

#### PPS Operator Timeline Gap Analysis

Sort all assignments for a PPS by `start_time` to get the ordered sequence of operator work. The gap between two consecutive `start_time` values is the operator's idle wait time.

**Diagnosing the gap by assignment_type:**
- Gap between two `current_schedule` or `cached` rows → **navigation or execution problem**
- Gap between two `new_task` rows → **MVTS planning problem**

```influxql
SELECT assignment_type, assigned_bot_id, start_time, pps_queue_reach_time,
       bot_start_time, available_start_time, counter_id
FROM mvts_rtp_problem_assignment_details
WHERE time > now() - 2h AND "InstallationId" = '<id>'
AND "assigned_pps_id" = '<pps_id>'
ORDER BY start_time ASC
```

#### Drilling into a `current_schedule` gap via the navigation table

**Step 1 — Identify the late bot and its expected window** from `mvts_rtp_problem_assignment_details`:
- `assigned_bot_id` → the bot to trace
- `bot_start_time` → when it should have departed (ms epoch)
- `pps_queue_reach_time` → when it should have arrived (ms epoch)
- `start_time` → when operator was expected to start

**Step 2 — Pull navigation legs:**
```influxql
SELECT path_type, idc_time_ms, path_travel_time_ms,
       deadlock_counts, deadlock_resolution_time_ms, start, goal
FROM task_cycle_times
WHERE time > '<bot_start_time as RFC3339>' AND time < '<start_time + 10min as RFC3339>'
AND "installation_id" = '<id>'
AND butler_id = <bot_id>
ORDER BY time ASC
```

**Step 3 — Identify the slow leg:**

| Pattern | Root cause |
|---------|-----------|
| `relay_io_point_to_relay_io_point` slow | Floor congestion on cross-aisle transit |
| `pps_entry_to_pps` actual >> idc with `idc ≈ 0` | PPS queue wait (another bot at the dock) |
| `deadlock_counts > 0` with high `deadlock_resolution_time_ms` | Navigation-level deadlock |
| All legs normal but bot arrived late | MVTS dispatched the bot too late — check `available_start_time` from previous cycle |

**⚠️ `available_start_time` +24h anomaly (PPS starvation bug):**

If a bot's `current_schedule` task has `bins = []` and `entity_pick_sequence = 0`, MVTS falls back to `available_start_time = problem_statement_time + 86400000ms` (+24h). When multiple bots all show +24h availability for the same PPS, MVTS cannot assign new bots there → **PPS goes idle**.

```influxql
SELECT assigned_bot_id, assignment_type, task_status, available_start_time, bins, entity_pick_sequence
FROM mvts_rtp_problem_assignment_details
WHERE time > now() - 1h AND "InstallationId" = '<id>'
AND "assigned_pps_id" = '<pps_id>'
AND assignment_type = 'current_schedule'
ORDER BY time DESC
```

---

### `mvts_pred_vs_real_times` — Predicted vs actual task timings (written by GMC)
One row per completed task.

| Field | Description |
|-------|-------------|
| `gmc_predicted_ranger_available_time` | When GMC predicted the bot would be free |
| `mvts_predicted_ranger_start_time` | When MVTS predicted the bot would start |
| `mvts_predicted_bot_arrival_time` | When MVTS predicted bot arrival at rack |
| `mvts_predicted_operator_start/end_time` | MVTS predicted operator window |
| `real_ranger_start_time` | When bot actually started |
| `real_bot_arrival_time` | When bot actually arrived at rack |
| `real_operator_start/end_time` | Actual operator pick window |
| `butler_id` | Tag — bot ID |
| `pps_id` | Tag — PPS ID |

```influxql
SELECT (real_ranger_start_time - mvts_predicted_ranger_start_time) AS prediction_error_ms,
       (real_bot_arrival_time - mvts_predicted_bot_arrival_time) AS arrival_error_ms, *
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
ORDER BY time DESC
```

---

### `task_cycle_times` — Actual bot movement times (written by Navigation team)
One row per physical bot movement (goto completion).

| Field | Description |
|-------|-------------|
| `butler_id` | Bot ID (field, not tag) |
| `idc_time_ms` | IDC predicted travel time (ms) |
| `path_travel_time_ms` | Actual travel time (ms) |
| `deadlock_counts` | Navigation-level path conflicts (different from VTM aisle deadlocks) |
| `deadlock_resolution_time_ms` | Time lost to navigation deadlocks |
| `start` / `goal` | Source and destination coordinates |
| `path_type` | Movement type |
| `task_type` | e.g. `relay_pps_task` |
| `original_idc_time_ms` | IDC time before any multiplier adjustment |

```influxql
SELECT (path_travel_time_ms - idc_time_ms) AS idc_error_ms, * FROM task_cycle_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
AND deadlock_counts > 0
ORDER BY time DESC
```

---

### `mvts_relay_vtm_bot_assignment_details` — VTM assignments (written by MVTS)

| Field | Type | Description |
|-------|------|-------------|
| `bot_id` | field (integer) | VTM bot ID |
| `current_aisle` | field | Aisle the bot is currently in |
| `task_chain` | field | Full pick/drop sequence |
| `request_id` | field | Solver cycle ID |
| `InstallationId` | tag | Installation filter |

---

### `mvts_relay_vtm_bot_aisle_switches` — VTM aisle assignment changes

| Field | Type | Description |
|-------|------|-------------|
| `bot_id` | field (integer) | VTM bot ID |
| `bot_id_tag` | tag | VTM bot ID (for filtering) |
| `from_aisle` | field | Previous aisle (negative = dummy/no aisle) |
| `to_aisle` | field | New assigned aisle |
| `transitive_aisles` | field | Intermediate aisles in the path |
| `request_id` | field | Solver cycle that triggered the switch |

---

## Cross-Table Debugging: Operator Timeline Gaps

Use when you want to understand why an operator was idle or why the actual timeline diverged from MVTS predictions.

### Join key
`mvts_rtp_problem_assignment_details.task_id` == `mvts_pred_vs_real_times.task_key` (both are GMC UUIDs).

`task_cycle_times.task_key` is a **Navigation UUID** — a different ID system. Correlate by `butler_id + time window` instead.

### Step 1 — Find tasks with large timeline gaps

| Gap | Formula | Meaning |
|-----|---------|---------|
| Bot start delay | `real_ranger_start_time - mvts_predicted_ranger_start_time` | MVTS underestimated how long bot was busy |
| Transit overrun | `real_bot_arrival_time - mvts_predicted_bot_arrival_time` | Bot took longer to travel than IDC predicted |
| PPS queue wait | `real_operator_start_time - real_bot_arrival_time` | Bot arrived but operator position wasn't free |
| Operator overrun | `real_operator_end_time - mvts_predicted_operator_end_time` | Operator took longer than planned |

```influxql
SELECT task_key, butler_id, pps_id,
  (real_bot_arrival_time - mvts_predicted_bot_arrival_time) AS transit_overrun_ms,
  (real_operator_start_time - real_bot_arrival_time) AS pps_queue_wait_ms,
  (real_operator_end_time - real_operator_start_time) AS actual_operator_time_ms,
  (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time) AS predicted_operator_time_ms
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
ORDER BY time DESC
```

### Step 2 — Drill into navigation legs for a specific task

```influxql
SELECT path_type, start, goal, idc_time_ms, path_travel_time_ms,
       deadlock_counts, deadlock_resolution_time_ms
FROM task_cycle_times
WHERE time > now() - 4h AND "installation_id" = '<id>'
AND butler_id = <bot_id>
ORDER BY time ASC
```

Full journey leg sequence:
```
relay_storable_to_relay_io_point    — pick up tote from rack
relay_io_point_to_relay_io_point    — cross-aisle transit
relay_io_point_to_relay_storable    — (multi-tote: pick next)
...
relay_io_point_to_pps_entry         — approach PPS
pps_entry_to_pps                    — enter PPS queue / dock
pps_to_pps_exit                     — exit PPS after operator picks
pps_exit_to_relay_io_point          — return to relay area
```

### Step 3 — Interpret the legs

**Transit overrun** — `path_travel_time_ms >> idc_time_ms` on a cross-aisle leg:
Floor congestion or IDC needs recalibration.

**PPS queue wait** — `pps_entry_to_pps` with `idc_time_ms ≈ 0` but large actual time:
Bot arrived but had to wait in queue — another bot was already at the dock.

**Navigation deadlock** — `deadlock_counts > 0` with high `deadlock_resolution_time_ms`:
Floor-level routing conflict. Not related to VTM aisle deadlocks.

**Post-PPS return overrun** — `pps_exit_to_relay_io_point` slow:
Return journey congested — common when many bots converge on relay area.

---

## VTM Reassignment / Deassignment Config Flags

### `ENABLE_VTM_TASK_REASSIGNMENT` (default: `true`)
- `true`: Previous cycle's VTM assignments are evaluated — pinned if still valid, deassigned if bot's aisle changed or TAM chain failed.
- `false`: Every cycle assigns VTM bots from scratch. No cascade deassignment or deadlock detection runs at all.

### `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM` (default: `false`)
- `true`: When bot B is stuck, bot A upstream of B gets its assignment dropped too, freeing the chain.
- `false`: No cascade deassignment — stuck bots hold their assignments.
- **Note**: As of f4 branch, this flag is only exercised in tests and is not yet wired into production code. Setting it via the config API may have no effect.

### Relationship to deadlocks
1. First check `ENABLE_VTM_TASK_REASSIGNMENT` — if `false`, the whole TAM/cascade system is off
2. `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM=true` adds cascade deassignment on TAM failure, but won't break a pure cyclic deadlock (A→B→A) — that still requires GMC action
