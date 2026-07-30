---
description: >
  MVTS Debugging — entry point for all MVTS issues. Resolves environment, checks health,
  and routes to the right sub-skill.
  TRIGGER when: user mentions MVTS, TPQ, VTM deadlock, HTM assignment, relay assignment,
  PPS starvation, high queue length, bots not assigned, problem statement debugging,
  IDC infinite, CrashLoopBackOff, or any debugging task in a named environment.
  TRIGGER also when: user says "bots aren't moving", "robots are stuck", "orders aren't going
  through", "warehouse is down", "nothing is happening in [env]", "bots are blocking each
  other", "pick stations have no work", "totes are waiting", "system crashed", "MVTS is down",
  or describes any warehouse operation problem without using technical terms.
argument-hint: Optional environment name (e.g. "onmbulk", "aphrelaybulk", "qa4-relaypotepic")
allowed-tools: ["Bash", "mcp__gor-global-mcp__list_environments", "mcp__gor-global-mcp__kube_connect",
  "mcp__gor-global-mcp__kube_shell", "mcp__gor-global-mcp__kube_pod_logs",
  "mcp__gor-global-mcp__query_metrics", "mcp__gor-global-mcp__run_influxdb_query_from_catalog",
  "mcp__gor-global-mcp__search_influxql_catalog", "mcp__gor-global-mcp__describe_measurement",
  "mcp__gor-global-mcp__search_logs", "mcp__gor-global-mcp__list_indices",
  "mcp__gor-global-mcp__health_check"]
---

# MVTS Debugging Guide

---

## Communication Protocol (always follow)

- Run diagnostic steps silently. Do NOT print raw kubectl commands, JSON, log lines, or InfluxDB rows to the user.
- After completing diagnostics, present a single plain-English **Diagnosis Summary** (template at the bottom).
- Translate findings: "available_capacity = 0" → "Bot is fully loaded and can't take another tote."
- Never ask the user to interpret output. Ask clarifying questions in plain English only.
- Lead with the conclusion, not the method.

---

## Step 0 — Acknowledge and confirm environment

Tell the user which environment you're checking:

> "Got it — checking MVTS in [env name] now. This may take a moment."

If no environment was given, ask: "Which environment should I check? (e.g. onmbulk, aphrelaybulk, qa4-relaypotepic)"

---

## Step 0.5 — First-run setup

On first invocation or a new machine, run `/mvts/setup` to configure SSH, JumpServer, and MCP permissions.

---

## Step 1 — Resolve the environment (silently)

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

## Step 2 — Check pod health (silently)

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

**On-prem:**
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

- Pod `CrashLoopBackOff` → use **`/mvts/ops/crash`**
- Pod `Running` → continue to Step 3

---

## Step 3 — Route based on what the user described

### Plain-English symptom → sub-skill

| What the user said | What it likely is | Sub-skill |
|--------------------|-------------------|-----------|
| "Bots aren't moving at all", "everything is frozen" | Pod crash or total failure | Pod down → `/mvts/ops/crash`; pod up → check both VTM and HTM |
| "Orders aren't being fulfilled", "pick stations idle", "operators have nothing to do" | HTM bots not delivering totes | `/mvts/ops/relay` |
| "Totes waiting at relay area but no bot comes", "bots skip relay points" | HTM relay assignment failure | `/mvts/ops/relay` |
| "Storage bots stuck", "bots blocking each other in aisles" | VTM deadlock | `/mvts/ops/vtm-stuck` |
| "Storage robots have no work", "throughput dropped", "aisle robots idle" | VTM aisle stall | `/mvts/ops/vtm-stuck` |
| "MVTS crashed", "scheduler is down", "service not running" | Pod in CrashLoopBackOff | `/mvts/ops/crash` |
| "System is slow", "bots taking too long", "transit times high" | IDC congestion multiplier | `/mvts/dev/idc-validate` |
| "Turn on/off a feature", "change a setting" | Config change | `/mvts/ops/config` |

### Technical symptom → sub-skill

| Symptom | Skill |
|---------|-------|
| Pod in CrashLoopBackOff / not starting | `/mvts/ops/crash` |
| HTM bots not assigned, totes stuck at relay points | `/mvts/ops/relay` |
| No VTM assignments, bots stuck in aisles | `/mvts/ops/vtm-stuck` |
| Operator idle gaps, PPS starvation, +24h available_start_time | `/mvts/dev/influxdb` |
| Predicted vs real timing analysis, navigation leg analysis | `/mvts/dev/influxdb` |
| `serviced_orders` / operator_time inflating or deflating | `/mvts/dev/operator-time-audit` |
| IDC multiplier wrong / over-estimating transit times | `/mvts/dev/idc-validate` |
| Multi vs single promotion ratio, MSIO dominating, multi-tote throughput analysis | `/mvts/dev/multi-single-analysis` |
| Is there a relationship/correlation between two or more metrics, bottleneck root-causing across metrics | `/mvts/dev/correlation-analysis` |

---

## Internal Reference — do not show commands or raw output to user

### Pod file locations

| File | Path |
|------|------|
| Config | `/app/data/config/local.application.properties` |
| Live log | `/app/data/logs/scheduler.log` |
| Archived logs | `/app/data/logs/scheduler.YYYY-MM-DD.N.log.gz` |

Pod name: `mvts-0` · Container: `ml-engine-mvts`

### Read config
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'"
```

Key flags: `ENABLE_RELAY`, `ENABLE_KAFKA`, `ENABLE_TRANSIT_TIME_MODEL`, `LOG_TO_INFLUXDB`, `ENABLE_VTM_TASK_REASSIGNMENT`

### Save latest problem statement
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep " Message:" /app/data/logs/scheduler.log | tail -1 | sed "s/.*Message: //" > /tmp/ps.json'
```
Always write to `/tmp/ps.json` first — JSON is ~950KB, piping directly causes truncation.

### Extract entities with jq
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

---

## Diagnosis Summary (show to user at the end)

```
**Environment**: [friendly name, e.g. "onmbulk"]
**Status**: [Running normally / Issue found / Offline]
**What I found**: [1-2 sentences in plain English]
**Impact**: [what is currently affected]
**What happens next**: [I fixed it / Needs GMC team / Needs an engineer / Monitoring continues]
```
