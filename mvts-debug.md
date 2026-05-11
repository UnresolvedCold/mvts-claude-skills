---
description: >
  MVTS Debugging Guide — entry point for all MVTS issues. Resolves environment, connects to the pod,
  and routes to the right sub-skill based on the symptom.
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

---

## Step 0 — First-run setup

On first invocation or on a new machine, run `/mvts-setup` to configure SSH, JumpServer, and MCP permissions.

---

## Step 1 — Resolve the environment

If `$ARGUMENTS` is set, that is the environment name — resolve it immediately.

**GCP environments** (namespace prefix `qa4-cluster-*`):
```
mcp__gor-global-mcp__list_environments   # find the environment alias
mcp__gor-global-mcp__kube_connect(cluster="qa4-cluster", project="greymatter-qa")
```

**On-prem environments** (namespace prefix `stpbulk-cluster-*`):
```bash
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"
# Namespace pattern: stpbulk-cluster-<env>-greymatter
```

| Namespace prefix | Cluster | Access method |
|-----------------|---------|---------------|
| `qa4-cluster-*` | qa4-cluster | GCP SA — `kube_connect` + `kube_shell` |
| `stpbulk-cluster-*` | stpbulk-cluster | On-prem — SSH JumpServer |

---

## Step 2 — Check pod health

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

**On-prem:**
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

- Pod `CrashLoopBackOff` → use **`/mvts-crashloop`**
- Pod `Running` → continue below

---

## Step 3 — Route to the right skill

| Symptom | Skill |
|---------|-------|
| Pod in CrashLoopBackOff / not starting | `/mvts-crashloop` |
| HTM bots not assigned, totes stuck at relay points | `/mvts-relay-debug` |
| No VTM assignments, bots stuck in aisles | `/mvts-vtm-deadlock` |
| Operator idle gaps, PPS starvation, +24h available_start_time | `/mvts-influxdb` |
| Predicted vs real timing analysis, navigation leg analysis | `/mvts-influxdb` |
| `serviced_orders` / operator_time inflating or deflating | `/mvts-operator-time-audit` |
| IDC multiplier wrong / over-estimating transit times | `/mvts-idc-validate` |

---

## Key reference: pod file locations

| File | Path |
|------|------|
| Config | `/app/data/config/local.application.properties` |
| Live log | `/app/data/logs/scheduler.log` |
| Archived logs | `/app/data/logs/scheduler.YYYY-MM-DD.N.log.gz` |

Pod name: `mvts-0` · Container: `ml-engine-mvts`

---

## Basic debugging commands

### Read config
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'cat /app/data/config/local.application.properties'"
# Or as JSON:
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'"
```

Key flags: `ENABLE_RELAY`, `ENABLE_KAFKA`, `ENABLE_TRANSIT_TIME_MODEL`, `LOG_TO_INFLUXDB`, `ENABLE_VTM_TASK_REASSIGNMENT`

### Change config at runtime (no restart needed)
```bash
# Single property
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8080/mvts/config/set?property=<NAME>&value=<value>"'

# Multiple properties
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s -X POST localhost:8080/mvts/config/set -H "Content-Type: application/json" -d "{\"PROP1\": \"val1\", \"PROP2\": \"val2\"}"'
```

### Save latest problem statement
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep " Message:" /app/data/logs/scheduler.log | tail -1 | sed "s/.*Message: //" > /tmp/ps.json'
```
Always write to `/tmp/ps.json` first — JSON is ~950KB, piping directly causes truncation.

Top-level PS keys: `task_list`, `ranger_list`, `relay_point_list`, `pps_list`, `adjacency_list`, `transport_entity_list`, `start_time`, `request_id`

### Extract entities with jq (from /tmp/ps.json)
```bash
# Bot by ID
jq ".ranger_list[] | select(.id == <bot_id>) | {id, coordinate, available_at_coordinate, version, status, available_capacity}" /tmp/ps.json

# Task by task_key
jq ".task_list[] | select(.task_key == \"<task_key>\")" /tmp/ps.json

# Relay point by tote ID
jq ".relay_point_list[] | select(.reserving_tote_id == \"<RELAYY_XXXX>\")" /tmp/ps.json

# PPS by ID
jq ".pps_list[] | select(.id == <pps_id>)" /tmp/ps.json
```

### IDC between two coordinates
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s "localhost:8383/idc/calculate?x1=<x1>&y1=<y1>&x2=<x2>&y2=<y2>&floor=1&liftState=down&botVersion=<HTM_QT_M5F|VTM_QT_C56_S_2D>"'
```
Returns travel time in ms. `10000000` = infinite = no valid path.

### Search logs for a request ID
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep "<request_id>" /app/data/logs/scheduler.log | grep -v "QueueManager: Message:"'
```
